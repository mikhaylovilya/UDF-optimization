# Решение: оптимизация `dbo.F_WORKS_LIST()`

Клиентский запрос: `select top 3000 * from dbo.F_WORKS_LIST()`.  
Целевой объём: ~50 000 заказов, в среднем ~3 элемента на заказ.

---

## Задача 1-го уровня. Выявленные недочёты и проблемы производительности

### 1. Скалярные UDF в SELECT — главное узкое место

В теле `F_WORKS_LIST` для **каждой** строки `Works` вызываются:

| Вызов | Что делает |
|-------|------------|
| `dbo.F_WORKITEMS_COUNT_BY_ID_WORK(id, 0)` | `COUNT(*)` по `WorkItem` + подзапрос к `Analiz` |
| `dbo.F_WORKITEMS_COUNT_BY_ID_WORK(id, 1)` | то же ещё раз |
| `dbo.F_EMPLOYEE_FULLNAME(Id_Employee)` | выборка из `Employee` (+ возможный вызов `F_EMPLOYEE_GET`) |

Скалярные UDF в SQL Server (особенно до совместимости с inlining) выполняются **построчно** (RBAR). Оптимизатор почти не видит их содержимое, план не может «слить» логику в set-based JOIN/агрегацию.

При 50 000 неудалённых заказов это порядка:

- **100 000** проходов по `WorkItem` (два счётчика на заказ);
- **50 000** обращений к `Employee`.

Клиентский `TOP 3000` **не помогает**: функция сначала заполняет табличную переменную `@RESULT` **всеми** подходящими строками, и только потом внешний `SELECT TOP 3000` обрезает результат.

### 2. `F_WORKITEMS_COUNT_BY_ID_WORK`: антипаттерн `NOT IN` и двойной проход

```sql
and id_analiz not in (select id_analiz from analiz where is_group = 1)
and is_complit = @is_complit
```

Проблемы:

- подзапрос к `Analiz` выполняется в контексте UDF на каждый вызов;
- `NOT IN` хуже, чем `NOT EXISTS` / `LEFT JOIN ... WHERE ... IS NULL` / `JOIN` с фильтром `IS_GROUP = 0`;
- для готовых и неготовых элементов делается **два отдельных** `COUNT` вместо одного прохода с условной агрегацией (`SUM(CASE WHEN ...)`).

Индекс `XIF5WorkItem (Id_Work)` есть, но без `Is_Complit` / покрытия он не закрывает запрос оптимально; фильтрация по `id_analiz` всё равно тянет данные анализов.

### 3. `F_EMPLOYEE_FULLNAME`: лишняя логика и повторные чтения

- При `NULL` вызывается `F_EMPLOYEE_GET()` → поиск по `SYSTEM_USER` в `Employee` — для списка заказов это лишний и неожиданный side-effect.
- Формирование ФИО через строковые операции в UDF вместо JOIN к `Employee` (или использования уже существующей computed-колонки `FULL_NAME`, пусть и с другим форматом).
- При пустом результате — **второй** SELECT по той же таблице за `LOGIN_NAME`.
- Возможны `NULL` в середине конкатенации (`SURNAME + ' ' + ...`), если поля `NULL` вопреки DEFAULT.

### 4. Табличная переменная `@RESULT`

- Нет статистики → оптимизатор часто считает «1 строку», что ухудшает планы при последующем использовании.
- Нет индексов на табличной переменной (в старых версиях) → `ORDER BY id_work DESC` внутри INSERT дорого на больших объёмах.
- Весь набор пишется в tempdb-структуру табличной переменной, затем читается снаружи.

### 5. Фильтр и сортировка по `Works`

```sql
WHERE WORKS.IS_DEL <> 1
ORDER BY id_work desc
```

- На `IS_DEL` **нет индекса** → возможен полный scan `Works`. Само сравнение `IS_DEL <> 1` на `bit NOT NULL` проблемой не является, поэтому в решении оно сохраняется как есть.
- Сортировка по PK `Id_Work` сама по себе хороша (кластерный индекс), но из‑за фильтра и отсутствия covering/filtered index план может быть неоптимальным.
- Нет FK/индекса, явно связывающего `Works.StatusId` → `WorkStatus` (JOIN по статусу без гарантированного индекса на FK со стороны Works; у `WorkStatus` PK есть).

### 6. Нерелевантные / слабые индексы для этого сценария

Есть индексы на `Works(Id_Employee)`, `Works(ID_ORGANIZATION)`, `Works(Id_Employee_Del)` — для списка заказов почти не нужны.  
Критичных для списка нет:

- `(IS_DEL, Id_Work DESC)` или filtered index `WHERE IS_DEL = 0`;
- covering по полям, отдаваемым в список;
- составной по `WorkItem(Id_Work, Is_Complit)` / include `ID_ANALIZ`.

### 7. Прочие замечания по качеству кода

- В `RETURNS TABLE` колонка названа `FULL_NAME`, в SELECT — alias `EmployeeFullName` (путаница имён).
- `D_DATE` как `varchar(10)` через `CONVERT(..., 104)` — форматирование на стороне БД, лишняя CPU-работа; лучше отдавать `datetime` клиенту.
- `F_WORKS_LIST` без параметров: нельзя передать `@Top`, фильтр по дате и т.д. — приложение вынуждено тянуть «всё через UDF».
- Многоколоночный CASE для `Is_Print` читает 5 nullable datetime — приемлемо по CPU, но при covering-индексе эти поля должны попасть в INCLUDE, иначе key lookup.

**Итог уровня 1:** основная причина медленной загрузки — построчные скалярные UDF + материализация полного списка в табличную переменную до применения `TOP 3000`.

---

## Задача 2-го уровня. Оптимизация без изменения структуры БД

Ограничение: не создавать новые таблицы/столбцы/триггеры/индексы как «обязательную» схему. Допускается **переписать функцию** (тело/логику запросов) — это правка программного объекта, а не структуры данных.

Цель: `SELECT TOP 3000 * FROM dbo.F_WORKS_LIST()` за **1–2 секунды** на ~50k заказов / ~3 позиции.

### Идея

1. Убрать вызовы `F_WORKITEMS_COUNT_BY_ID_WORK` и `F_EMPLOYEE_FULLNAME`.
2. Считать счётчики **одним** set-based `GROUP BY` с `SUM(CASE ...)`, при этом агрегация ограничена уже отфильтрованными заказами (`AllData`), а не всей таблицей `WorkItem`.
3. ФИО сотрудника — через `LEFT JOIN Employee`.
4. **Не** добавлять `TOP` внутрь функции: функция по-прежнему возвращает все не удалённые заказы, а клиентский `SELECT TOP 3000 * FROM dbo.F_WORKS_LIST()` остаётся без изменений. Это важно для сохранения контракта и корректности проверки.
5. Заменить `NOT IN (групповые анализы)` на `NOT EXISTS` с `IS_GROUP = 1` — так сохраняется исходная семантика: исключаются только реально групповые анализы.

### Предлагаемая реализация `F_WORKS_LIST`

```sql
CREATE OR ALTER FUNCTION [dbo].[F_WORKS_LIST] ()
RETURNS @RESULT TABLE
(
  ID_WORK INT,
  CREATE_Date DATETIME,
  MaterialNumber DECIMAL(8,2),
  IS_Complit BIT,
  FIO VARCHAR(255),
  D_DATE varchar(10),
  WorkItemsNotComplit int,
  WorkItemsComplit int,
  FULL_NAME VARCHAR(101),
  StatusId smallint,
  StatusName VARCHAR(255),
  Is_Print bit
)
AS
BEGIN
  ;WITH AllData AS
  (
    SELECT
      w.Id_Work,
      w.CREATE_Date,
      w.MaterialNumber,
      w.IS_Complit,
      w.FIO,
      w.Id_Employee,
      w.StatusId,
      w.Print_Date,
      w.SendToClientDate,
      w.SendToDoctorDate,
      w.SendToOrgDate,
      w.SendToFax
    FROM dbo.Works AS w
    WHERE w.IS_DEL <> 1
  ),
  WorkItemCounts AS
  (
    SELECT
      wi.Id_Work,
      SUM(CASE WHEN wi.Is_Complit = 0 THEN 1 ELSE 0 END) AS WorkItemsNotComplit,
      SUM(CASE WHEN wi.Is_Complit = 1 THEN 1 ELSE 0 END) AS WorkItemsComplit
    FROM dbo.WorkItem AS wi
    INNER JOIN AllData AS ad
      ON ad.Id_Work = wi.Id_Work
    WHERE NOT EXISTS
    (
      SELECT 1
      FROM dbo.Analiz AS a
      WHERE a.ID_ANALIZ = wi.ID_ANALIZ
        AND a.IS_GROUP = 1
    )
    GROUP BY wi.Id_Work
  )
  INSERT INTO @RESULT
  (
    ID_WORK,
    CREATE_Date,
    MaterialNumber,
    IS_Complit,
    FIO,
    D_DATE,
    WorkItemsNotComplit,
    WorkItemsComplit,
    FULL_NAME,
    StatusId,
    StatusName,
    Is_Print
  )
  SELECT
    ad.Id_Work,
    ad.CREATE_Date,
    ad.MaterialNumber,
    ad.IS_Complit,
    ad.FIO,
    CONVERT(varchar(10), ad.CREATE_Date, 104) AS D_DATE,
    ISNULL(wic.WorkItemsNotComplit, 0) AS WorkItemsNotComplit,
    ISNULL(wic.WorkItemsComplit, 0) AS WorkItemsComplit,
    RTRIM(
      e.Surname + ' ' +
      UPPER(SUBSTRING(e.Name, 1, 1)) + '. ' +
      UPPER(SUBSTRING(e.Patronymic, 1, 1)) + '.'
    ) AS FULL_NAME,
    ad.StatusId,
    ws.StatusName,
    CAST(
      CASE
        WHEN ad.Print_Date IS NOT NULL
          OR ad.SendToClientDate IS NOT NULL
          OR ad.SendToDoctorDate IS NOT NULL
          OR ad.SendToOrgDate IS NOT NULL
          OR ad.SendToFax IS NOT NULL
        THEN 1
        ELSE 0
      END AS bit
    ) AS Is_Print
  FROM AllData AS ad
  LEFT JOIN WorkItemCounts AS wic
    ON wic.Id_Work = ad.Id_Work
  LEFT JOIN dbo.Employee AS e
    ON e.Id_Employee = ad.Id_Employee
  LEFT JOIN dbo.WorkStatus AS ws
    ON ws.StatusID = ad.StatusId
  ORDER BY ad.Id_Work DESC;

  RETURN;
END
GO
```

### Почему этого достаточно для 1–2 сек

| Было | Стало |
|------|--------|
| ~2×N скалярных COUNT по WorkItem | Один `GROUP BY Id_Work`, ограниченный `AllData` |
| N вызовов UDF ФИО | Один `LEFT JOIN Employee` |
| Материализация полного списка построчно | Один set-based `INSERT ... SELECT`; `TOP 3000` остаётся в клиентском запросе |
| `NOT IN` + повторный scan Analiz в UDF | Один `NOT EXISTS` по `Analiz` |

Функция возвращает все не удалённые заказы, как и раньше; внешний `TOP 3000` по-прежнему обрезает результат на стороне клиента. Выигрыш достигается не за счёт изменения контракта, а за счёт замены построчных UDF на один set-based запрос. На 50k заказов / ~150k work items один hash/stream aggregate + join обычно укладывается в 1–2 секунды даже без новых индексов (имеющийся `XIF5WorkItem(Id_Work)` и PK `Analiz` помогают).

### Фактические замеры

Проверка проведена локально: SQL Server 2022 (16.0.4325.4, CU28) в Docker под эмуляцией `linux/amd64` на Apple Silicon. Объём данных: 50 000 `Works`, 150 000 `WorkItem`, 20 `Employee`, 55 `Analiz` (50 негрупповых + 5 групповых). Старая версия функции сохранена как `F_WORKS_LIST_OLD` для сравнения. Два последовательных замера после прогрева через `SET STATISTICS TIME ON`:

| Версия | CPU time | Elapsed time |
|--------|----------|--------------|
| `F_WORKS_LIST_OLD` (исходная, со скалярными UDF) | 3424–3615 мс | 3567–3614 мс |
| `F_WORKS_LIST` (set-based) | 138–140 мс | 140 мс |
| Выигрыш | ~25–26× | ~25–26× |

Корректность результата проверена `EXCEPT` в обе стороны:

- на **полном** наборе (50 000 строк без `TOP`) расхождений 0, обе функции возвращают по 50 000 строк;
- сравнение `SELECT TOP 3000 *` **без внешнего `ORDER BY`** дало 2991 мнимое расхождение: `TOP` без сортировки не гарантирует, какие именно строки попадут в выборку из табличной переменной (у OLD `max(ID_WORK)=20691`, у NEW `17875`). С явным `ORDER BY ID_WORK DESC` в обоих внешних запросах расхождений 0, обе выборки содержат `ID_WORK` 47001–50000. Клиентский запрос при этом менять не нужно — расхождение было артефактом тестового сравнения, а не разной логикой функций.

> Абсолютные миллисекунды на macOS под эмуляцией x64 не показательны для реального сервера: показателен относительный выигрыш (~26×) и переход от ~2N скалярных вызовов к одному set-based запросу.

### Замечания по совместимости с приложением

- Сигнатура, набор колонок `@RESULT` и условие `IS_DEL <> 1` сохранены — клиентский `SELECT TOP 3000 * FROM dbo.F_WORKS_LIST()` менять не нужно. Внутри функции `TOP` больше нет, поэтому результат функции не сужается до 3000 строк.
- Поведение `F_EMPLOYEE_FULLNAME` при `Id_Employee IS NULL` (подстановка текущего пользователя через `F_EMPLOYEE_GET`) и fallback на `LOGIN_NAME` при пустом ФИО **намеренно не воспроизводятся**: ФИО берётся напрямую из `Employee` по `Works.Id_Employee`. Это единственное осознанное отклонение в поведении; при требования 100% совместимости пришлось бы оставить вызов UDF только для `NULL`/пустых значений, но тогда часть выигрыша теряется.
- `CREATE OR ALTER` выбран вместо `CREATE FUNCTION`, чтобы скрипт можно было применять и к уже существующему объекту (в том числе при повторной проверке).

### Про индексы

Индексы в решение уровня 2 **не добавляются**: это изменение структуры БД, которое заданием не разрешено. Возможные варианты индексов описаны ниже в задании 3 как отдельный шаг. Set-based переписывание функции без них даёт целевые 1–2 секунды.

### Генератор тестовых данных (для проверки)

```sql
-- статусы
INSERT INTO WorkStatus (StatusName) VALUES ('New'), ('InProgress'), ('Done');

-- сотрудники
DECLARE @i int = 1;
WHILE @i <= 20
BEGIN
  INSERT INTO Employee (Login_Name, Name, Patronymic, Surname, Archived, IS_Role)
  VALUES ('user' + CAST(@i AS varchar(10)), 'Name', 'Patr', 'Surname' + CAST(@i AS varchar(10)), 0, 0);
  SET @i += 1;
END;

-- анализы (не групповые)
SET @i = 1;
WHILE @i <= 50
BEGIN
  INSERT INTO Analiz (IS_GROUP, CODE_NAME, FULL_NAME)
  VALUES (0, 'A' + CAST(@i AS varchar(10)), 'Analiz ' + CAST(@i AS varchar(10)));
  SET @i += 1;
END;

-- 50000 заказов
SET @i = 1;
WHILE @i <= 50000
BEGIN
  INSERT INTO Works (IS_Complit, Id_Employee, FIO, Is_Del, StatusId, MaterialNumber)
  VALUES (0, (@i % 20) + 1, 'Patient ' + CAST(@i AS varchar(10)), 0, (@i % 3) + 1, @i);
  SET @i += 1;
END;

-- ~3 workitem на заказ
INSERT INTO WorkItem (Is_Complit, ID_ANALIZ, Id_Work, Is_Print, Is_Select)
SELECT
  CASE WHEN v.n = 1 THEN 1 ELSE 0 END,
  (w.Id_Work % 50) + 1,
  w.Id_Work,
  1, 0
FROM Works w
CROSS JOIN (VALUES (1),(2),(3)) v(n);
```

Замер:

```sql
SET STATISTICS TIME ON;
SELECT TOP 3000 * FROM dbo.F_WORKS_LIST();
SET STATISTICS TIME OFF;
```

### Использование LLM (промпты)

Промпт 1 (анализ):

> Проанализируй multi-statement TVF `F_WORKS_LIST` в MS SQL Server и связанные скалярные функции `F_WORKITEMS_COUNT_BY_ID_WORK`, `F_EMPLOYEE_FULLNAME`. Клиент делает `SELECT TOP 3000 * FROM dbo.F_WORKS_LIST()` на ~50 000 заказов. Перечисли проблемы производительности и почему TOP не спасает.

Промпт 2 (оптимизация без смены схемы):

> Перепиши `F_WORKS_LIST` без вызовов скалярных UDF: set-based подсчёт готовых/неготовых WorkItem через NOT EXISTS по IS_GROUP=1, JOIN к Employee и WorkStatus. Не добавляй TOP внутрь функции — клиентский TOP 3000 остаётся снаружи; агрегацию ограничь уже отфильтрованными заказами. Сигнатуру таблицы результата не меняй. Объясни, почему это уложится в 1–2 секунды.

Промпт 3 (уровень 3):

> Какие минусы у денормализации счётчиков WorkItem в Works, материализованных view, триггеров и дополнительных индексов для этого списка заказов?

---

## Задача 3-го уровня. Структурные изменения: недостатки и риски

Если set-based переписывания функции недостаточно или нужны дальнейшие ускорения, типичные структурные шаги и их цена:

### 0. Индексы (в решение уровня 2 не входят)

Создание индексов меняет физическую структуру БД, поэтому это отдельный шаг, а не часть оптимизации без изменения схемы:

```sql
CREATE NONCLUSTERED INDEX [IX_WorkItem_IdWork_IsComplit_IncAnaliz]
  ON [dbo].[WorkItem] ([Id_Work], [Is_Complit])
  INCLUDE ([ID_ANALIZ]);

-- для NOT EXISTS по групповым анализам
CREATE NONCLUSTERED INDEX [IX_Analiz_IsGroup_IncId]
  ON [dbo].[Analiz] ([IS_GROUP])
  INCLUDE ([ID_ANALIZ]);
-- либо более узкий вариант по ключу: (ID_ANALIZ, IS_GROUP)

-- фильтруемый индекс под выборку не удалённых заказов
CREATE NONCLUSTERED INDEX [IX_Works_NotDeleted]
  ON [dbo].[Works] ([Id_Work] DESC)
  WHERE [IS_DEL] = 0;
```

### 1. Денормализация счётчиков в `Works`

Например, столбцы `WorkItemsComplit`, `WorkItemsNotComplit`, поддержка триггерами на `WorkItem` / `Analiz`.

**Минусы:**

- риск рассинхрона при прямых UPDATE/bulk load / отключённых триггерах;
- усложнение логики записи, отладки, миграций;
- триггеры увеличивают длительность каждой вставки/смены статуса анализа;
- при частых обновлениях `WorkItem` — contention на строках `Works`;
- изменение `IS_GROUP` у анализа требует массового пересчёта связанных заказов.

### 2. Отдельная таблица-кэш / summary (`WorksListCache`)

**Минусы:** дублирование данных, необходимость ETL/триггеров/job’ов обновления, eventual consistency, дополнительное место на диске, усложнение бэкапов и прав доступа.

### 3. Индексированное (материализованное) представление

**Минусы:**

- жёсткие ограничения SQL Server на schema-bound indexed views (детерминизм, типы JOIN, агрегаты);
- накладные расходы на DML по базовым таблицам;
- сложность сопровождения при эволюции схемы `WorkItem`/`Analiz`.

### 4. Много «широких» covering-индексов под один экран

**Минусы:** замедление INSERT/UPDATE/DELETE, рост БД, более долгие rebuild/reorganize, риск выбора оптимизатором неоптимального индекса для других запросов.

### 5. Замена TVF на хранимую процедуру

**Минусы:** ломает текущий контракт приложения (`SELECT ... FROM dbo.F_WORKS_LIST()`), если клиент нельзя менять — неприемлемо. Процедура удобнее для параметров/`OPTION`/`RECOMPILE`, но это уже изменение API.

### 6. Computed persisted columns / смена типов (`D_DATE`, флаги печати)

**Минусы:** увеличение размера строки, время на rebuild таблицы, необходимость регрессионного тестирования всех отчётов, читающих те же поля.

### 7. Партиционирование `Works` / `WorkItem`

**Минусы:** избыточно для 50k строк, операционная сложность (sliding window, aligned indexes), выгода почти нулевая на таком объёме.

---

## Рекомендуемый порядок внедрения

1. **Сначала** — set-based переписывание `F_WORKS_LIST` без изменения его контракта (уровень 2): максимальный эффект, минимальный риск для модели данных. `TOP 3000` остаётся в клиентском запросе.
2. **При необходимости** — точечные индексы под фильтр/JOIN списка (уровень 3, изменение структуры).
3. **Только если** профиль нагрузки на запись низкий, а чтение списка критично и объёмы вырастут на порядки — рассматривать денормализацию/кэш с пониманием рисков из уровня 3.

Ожидаемый результат после шага 1: время `SELECT TOP 3000 * FROM dbo.F_WORKS_LIST()` на описанном объёме — в пределах **1–2 секунд** без изменения таблиц.
