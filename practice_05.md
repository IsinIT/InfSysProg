# Практическая работа №2. Подключение SQLite и Entity Framework Core (подход Code First)

**Дисциплина:** Информационные системы / Разработка серверных приложений
**Проект:** MyCollegePlus (backend — ASP.NET Core 10, EF Core 10, SQLite; инфраструктура — Docker Compose)
**Продолжительность:** 4 академических часа
**Предыдущая работа:** №1 «Централизованное логирование ASP.NET Core в Seq (Docker)»

---

## Цель работы

Подключить SQLite в качестве базы данных приложения с помощью Entity Framework Core (подход **Code First**), перенести все данные, жёстко записанные в контроллерах, в БД, заполнить БД первоначальными данными (seed), реализовать получение данных из БД, создать endpoint для добавления данных в БД и логировать в Seq все SQL-запросы, сформированные Entity Framework.

## Результат, который нужно получить

1. В проекте есть класс контекста `CollegeDbContext` и миграция `InitialCreate`, по которой создаётся файл БД `college.db` с таблицей `Students`.
2. Два студента, которые раньше были записаны массивом в `StudentController`, теперь хранятся в БД (добавлены миграцией как начальные данные).
3. В контроллере не осталось захардкоженных данных: `GET api/Student/get-students` и `GET api/Student/get-student/{nickName}` читают данные из БД.
4. `POST api/Student/add-student` добавляет студента в БД, проверяет входные данные (`400`) и уникальность ника (`409`), возвращает `201 Created`.
5. При первом запуске БД автоматически заполняется первоначальными данными (пять студентов учебной группы). Повторные запуски не создают дубликатов.
6. Данные сохраняются после перезапуска приложения, в том числе в Docker-контейнере.
7. **Все** SQL-запросы, сформированные EF Core (миграции, seed, чтение, добавление), записываются в Seq как структурные события с текстом SQL, параметрами и временем выполнения. Их можно связать с HTTP-запросом (продолжение работы №1).

## Исходное состояние

Работа №1 выполнена: Serilog и Seq подключены, есть `Filters/LogActionFilter.cs`, а `StudentController` пишет ручные логи. Данные студентов хранятся **в коде контроллера**:

```csharp
private readonly People[] _pepoples =
[
    new People("Максим", "Рубченко", "Ильич", 39, "W4rl0ck"),
    new People("Александр", "Донцов", "Виторович", 40, "R0l4nd")
];
```

Модель — позиционная запись (record) без идентификатора:

```csharp
record People(string FirstName, string LastName, string MiddleName, uint Age, string NickName);
```

**Проблемы такого хранения:**

- Массив — поле контроллера. ASP.NET Core **создаёт новый экземпляр контроллера на каждый запрос**, поэтому любые изменения массива пропадут к следующему запросу. Добавить студента в принципе невозможно.
- Данные теряются при перезапуске и не разделяются между несколькими экземплярами приложения.
- Чтобы изменить данные, нужно пересобирать и заново выкладывать приложение.
- У записи нет уникального идентификатора, поэтому на конкретного студента нельзя надёжно сослаться.

---

## Теоретическая часть

### ORM и Entity Framework Core

**ORM (Object-Relational Mapping)** — технология, которая связывает объекты языка программирования (классы C#) с таблицами реляционной БД. Разработчик работает с объектами и LINQ-запросами, а ORM сам генерирует SQL, выполняет его и превращает строки результата в объекты.

**Entity Framework Core (EF Core)** — официальная ORM платформы .NET от Microsoft. Основные понятия:

| Понятие | Что это | В нашем проекте |
|---|---|---|
| **Сущность (Entity)** | Класс, экземпляры которого хранятся в таблице | `People` |
| **DbContext** | Сеанс работы с БД: подключение, отслеживание изменений, сохранение | `CollegeDbContext` |
| **DbSet\<T\>** | Коллекция сущностей одного типа — «таблица» в коде, к ней пишутся LINQ-запросы | `Students` |
| **Провайдер** | Пакет, который переводит запросы EF в диалект конкретной СУБД | `Microsoft.EntityFrameworkCore.Sqlite` |
| **Миграция** | Класс C#, описывающий изменение схемы БД (создать таблицу, добавить столбец…) | `InitialCreate` |

**Почему ORM, а не «ручной» SQL через ADO.NET.** Запросы проверяются компилятором (опечатка в имени поля — ошибка сборки, а не ошибка в продакшене). Параметры всегда передаются через параметризованные запросы — защита от SQL-инъекций. Меньше шаблонного кода по чтению `DataReader` и сборке объектов. Смена СУБД (SQLite → PostgreSQL, который уже есть в нашем `docker-compose.yml`) сводится в основном к замене провайдера. Цена — ORM нужно понимать, иначе легко написать неэффективный запрос. Поэтому в работе мы выводим генерируемый SQL в Seq.

### Подход Code First

Есть два основных подхода:

- **Database First** — сначала проектируется БД (вручную или DBA), затем по ней генерируются классы (`dotnet ef dbcontext scaffold`).
- **Code First** — сначала пишутся классы C# и конфигурация, а схема БД **генерируется из кода** с помощью миграций.

**Почему здесь Code First:**

1. **Модель — в коде и в git.** Схема БД версионируется вместе с приложением. По истории коммитов видно, кто, когда и зачем добавил столбец.
2. **Миграции — воспроизводимая эволюция схемы.** Любой разработчик, тестовый стенд или контейнер получают *одинаковую* БД одной командой. Не нужно пересылать SQL-скрипты.
3. **Новый проект без существующей БД.** Database First оправдан, когда БД уже есть (унаследованная система). У нас БД нет — значит, естественно описывать её кодом.
4. **Начальные данные (seed) тоже в коде.** Так мы переносим студентов из контроллера в БД: они попадают в миграцию.

### Почему SQLite

**SQLite** — встраиваемая СУБД: вся БД хранится в одном файле, отдельный сервер не нужен, библиотека поставляется вместе с NuGet-пакетом.

- **Для учебного проекта и прототипа** — ноль настроек: не нужен контейнер, пароли, сеть. Можно сосредоточиться на EF Core.
- **Полноценный SQL и транзакции (ACID)**, индексы, ограничения уникальности.
- **Ограничения:** одна запись в момент времени (низкая конкурентность записи), нет пользователей и прав, не подходит для нескольких экземпляров приложения с общей БД. Для продакшена в нашем стеке предусмотрен PostgreSQL — благодаря EF Core переход на него займёт минимум изменений.

### Время жизни DbContext и внедрение зависимостей

`AddDbContext` регистрирует контекст как **Scoped** — один экземпляр на HTTP-запрос. Контекст **не потокобезопасен** и накапливает отслеживаемые объекты, поэтому он должен жить недолго: создаётся на запрос и уничтожается после него. Контроллер получает его через конструктор (DI), как получал `ILogger` в работе №1.

### Асинхронный доступ к данным

Методы `ToListAsync`, `FirstOrDefaultAsync`, `SaveChangesAsync` не блокируют поток, пока БД выполняет запрос. Поток возвращается в пул и обслуживает другие запросы, что повышает пропускную способность сервера. `CancellationToken` позволяет прервать запрос к БД, если клиент закрыл соединение.

### DTO вместо сущности на входе API

**DTO (Data Transfer Object)** — отдельный класс для данных, приходящих в API. Принимать в `POST` саму сущность опасно: клиент может прислать `Id` или поля, которые он менять не должен (**overposting / mass assignment**). DTO содержит только разрешённые поля и правила их проверки.

### Начальное заполнение БД (seed)

**Seed (засев)** — автоматическое наполнение новой БД данными, без которых приложение не работает или которые нужны для демонстрации и проверки: справочники, учётная запись администратора, тестовая группа студентов. Ручное добавление записей после каждого развёртывания ненадёжно: о нём забывают, и окружения начинают различаться. В EF Core два механизма:

- **`HasData`** — данные описываются в модели и попадают в миграцию. Подходит для неизменных данных, являющихся частью схемы.
- **`UseSeeding` / `UseAsyncSeeding`** (EF Core 9+) — произвольный код, который EF выполняет после применения миграций. Подходит для первоначальных данных, которые удобно менять без миграций.

Главное требование к seed-коду — **идемпотентность**: сколько бы раз он ни выполнился, результат одинаковый.

### Логирование SQL-запросов EF Core

EF Core пишет каждый выполненный SQL-запрос в категорию логов `Microsoft.EntityFrameworkCore.Database.Command`. Так как Serilog в работе №1 подключён как провайдер `ILogger`, эти события автоматически попадают в Seq. Нужно лишь разрешить категорию в фильтре уровней. Логи SQL позволяют увидеть, во что превратился LINQ-запрос, сколько запросов выполняет один HTTP-запрос и сколько длится каждый. Значения параметров по умолчанию скрыты, так как это персональные данные. Их включают только в разработке (`EnableSensitiveDataLogging`).

---

## Ход работы

### Шаг 1. Установить пакеты EF Core и инструмент `dotnet-ef`

Перейдите в папку backend:

```bash
cd code/backend/MyCollegePlus
```

Установите провайдер SQLite (он подтягивает сам EF Core):

```bash
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
```

Установите пакет времени разработки (нужен для команд миграций):

```bash
dotnet add package Microsoft.EntityFrameworkCore.Design
```

Создайте манифест локальных инструментов и установите `dotnet-ef`:

```bash
dotnet new tool-manifest
```

```bash
dotnet tool install dotnet-ef
```

Проверьте:

```bash
dotnet ef --version
```

В `MyCollegePlus.csproj` должны появиться строки (версии могут быть новее, но **мажорная версия EF Core и `dotnet-ef` должна совпадать**):

```xml
<PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="10.0.14">
  <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
  <PrivateAssets>all</PrivateAssets>
</PackageReference>
<PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="10.0.14" />
```

> **Почему это делается.**
> - `Microsoft.EntityFrameworkCore.Sqlite` — провайдер: переводит LINQ в SQL диалекта SQLite и содержит нативную библиотеку SQLite для Windows, Linux и macOS.
> - `Microsoft.EntityFrameworkCore.Design` нужен **только инструментам** (создание миграций). Пометка `PrivateAssets=all` означает, что он не попадает в зависимости собранного приложения.
> - **Локальный инструмент** (файл `dotnet-tools.json`) вместо глобального `dotnet tool install -g`: версия `dotnet-ef` фиксируется в репозитории, и у всех разработчиков она одинаковая. Другой участник команды выполняет `dotnet tool restore` и получает ту же версию.

---

### Шаг 2. Превратить модель `People` в сущность

Замените содержимое `Models/People.cs`:

```csharp
namespace MyCollegePlus.Models;

public class People
{
    public int Id { get; set; }
    public required string FirstName { get; set; }
    public required string LastName { get; set; }
    public string? MiddleName { get; set; }
    public uint Age { get; set; }
    public required string NickName { get; set; }
}
```

> **Почему это делается.**
> - **Первичный ключ `Id`.** Любой строке таблицы нужен уникальный идентификатор. По соглашению EF Core свойство с именем `Id` (или `PeopleId`) автоматически становится первичным ключом, а для `int` значение генерирует сама БД (автоинкремент).
> - **Класс со свойствами `get; set;` вместо позиционного record.** EF Core отслеживает изменения сущности и материализует объекты из строк таблицы, присваивая значения свойствам. Изменяемый класс — самый простой и предсказуемый вариант для сущности. Record с семантикой «равенство по значению» плохо сочетается с отслеживанием изменений: две сущности с одинаковыми полями, но разными `Id`, — разные записи.
> - **`public`.** Раньше record был `internal` (модификатор не указан). Сущность используется в публичном `DbSet` контекста, поэтому должна быть публичной.
> - **`required` и `string?`** (Nullable reference types включены в `.csproj`). EF Core читает эту информацию: `string` превращается в столбец `NOT NULL`, а `string?` — в `NULL`. Отчество сделано необязательным: оно есть не у всех.

---

### Шаг 3. Создать контекст БД и перенести данные из контроллера

Создайте папку `Data` и файл `Data/CollegeDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using MyCollegePlus.Models;

namespace MyCollegePlus.Data;

public class CollegeDbContext : DbContext
{
    public CollegeDbContext(DbContextOptions<CollegeDbContext> options) : base(options)
    {
    }

    public DbSet<People> Students => Set<People>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<People>(entity =>
        {
            entity.ToTable("Students");
            entity.HasKey(p => p.Id);

            entity.Property(p => p.FirstName).HasMaxLength(100).IsRequired();
            entity.Property(p => p.LastName).HasMaxLength(100).IsRequired();
            entity.Property(p => p.MiddleName).HasMaxLength(100);
            // NOCASE: "W4rl0ck" и "w4rl0ck" считаются одним ником
            entity.Property(p => p.NickName).HasMaxLength(50).IsRequired().UseCollation("NOCASE");

            entity.HasIndex(p => p.NickName).IsUnique();

            // Начальные данные — перенесены из массива в StudentController
            entity.HasData(
                new People { Id = 1, FirstName = "Максим", LastName = "Рубченко", MiddleName = "Ильич", Age = 39, NickName = "W4rl0ck" },
                new People { Id = 2, FirstName = "Александр", LastName = "Донцов", MiddleName = "Виторович", Age = 40, NickName = "R0l4nd" });
        });
    }
}
```

> **Почему это делается.**
> - **Конструктор с `DbContextOptions`.** Контекст не знает, какая у него СУБД и строка подключения: это передаётся снаружи через DI (шаг 4). Поэтому тот же контекст можно подключить к SQLite в разработке, к PostgreSQL в продакшене или к in-memory БД в тестах.
> - **`DbSet<People> Students`** — точка входа для запросов: `_db.Students.Where(...)`. Имя таблицы задаём явно (`ToTable("Students")`), так как по умолчанию оно совпало бы с именем DbSet, а «People» плохо описывает содержимое.
> - **Fluent API (`OnModelCreating`) вместо атрибутов в сущности.** Конфигурация хранения (длины, индексы, сортировка) собрана в одном месте и не «засоряет» модель. Fluent API также умеет больше атрибутов: уникальные индексы, collation, seed-данные.
> - **`HasMaxLength`.** SQLite длину не проверяет, но EF учитывает её в модели. При переходе на PostgreSQL это станет `varchar(100)`. Кроме того, это документация требований к данным.
> - **Уникальный индекс на `NickName`** — гарантия уровня БД: даже если в коде будет ошибка, два одинаковых ника не сохранятся. Индекс также ускоряет поиск по нику.
> - **`UseCollation("NOCASE")`** — сравнение строк без учёта регистра в SQLite. Поиск `get-student/w4rl0ck` найдёт `W4rl0ck`, а уникальный индекс не даст зарегистрировать `w4rl0ck` при существующем `W4rl0ck`. Это сохраняет поведение из работы №1, где поиск выполнялся с `StringComparison.OrdinalIgnoreCase`.
> - **`HasData` (seed) — перенос захардкоженных данных.** Начальные данные попадают в миграцию как `INSERT`. Они появятся в любой новой БД автоматически и ровно один раз. `Id` в `HasData` указывается **явно**: EF должен однозначно отличать эти строки в следующих миграциях (чтобы обновить или удалить их, если seed изменится).

---

### Шаг 4. Строка подключения и регистрация контекста

**4.1.** В `appsettings.json` добавьте секцию `ConnectionStrings` (на верхнем уровне, рядом с `Serilog`):

```json
  "ConnectionStrings": {
    "CollegeDb": "Data Source=college.db"
  }
```

**4.2.** В `Program.cs` добавьте using-директивы:

```csharp
using Microsoft.EntityFrameworkCore;
using MyCollegePlus.Data;
```

и регистрацию контекста перед `builder.Services.AddOpenApi();`:

```csharp
    builder.Services.AddDbContext<CollegeDbContext>(options =>
        options.UseSqlite(builder.Configuration.GetConnectionString("CollegeDb")));
```

> **Почему это делается.**
> - **Строка подключения в конфигурации, а не в коде** — тот же принцип, что и с адресом Seq в работе №1: в Docker путь к файлу БД переопределяется переменной окружения без пересборки (шаг 12). `GetConnectionString("CollegeDb")` читает `ConnectionStrings:CollegeDb`.
> - **`Data Source=college.db`** — путь относительно рабочего каталога. При `dotnet run` это папка проекта. Файл создастся при первом применении миграций.
> - **`AddDbContext`** регистрирует контекст в DI с временем жизни *Scoped* (один на HTTP-запрос) и настраивает провайдер SQLite.

---

### Шаг 5. Логирование в Seq всех SQL-запросов, сформированных EF Core

EF Core пишет каждый выполненный SQL-запрос через тот же `ILogger`, что и остальное приложение. Наша задача — пропустить эти события через фильтры Serilog из работы №1 и сделать их максимально информативными.

**5.1. Разрешить категорию SQL-команд.** В `appsettings.json` в секцию `Serilog:MinimumLevel:Override` добавьте строку:

```json
        "Microsoft.EntityFrameworkCore.Database.Command": "Information"
```

Итоговая секция `Override`:

```json
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.AspNetCore": "Warning",
        "Microsoft.Hosting.Lifetime": "Information",
        "Microsoft.EntityFrameworkCore.Database.Command": "Information",
        "System": "Warning",
        "MyCollegePlus": "Debug"
      }
```

**5.2. Значения параметров и подробные ошибки — только в разработке.** Замените регистрацию контекста из шага 4 на:

```csharp
    builder.Services.AddDbContext<CollegeDbContext>(options =>
    {
        options.UseSqlite(builder.Configuration.GetConnectionString("CollegeDb"));

        // Значения параметров SQL и подробные ошибки — только в разработке
        if (builder.Environment.IsDevelopment())
        {
            options.EnableSensitiveDataLogging()
                .EnableDetailedErrors();
        }
    });
```

**5.3. Помечать запросы в коде.** В запросах контроллера (шаг 10) используется `TagWith`:

```csharp
var students = await _db.Students
    .TagWith("StudentController.GetStudents: список студентов")
    .AsNoTracking()
    ...
```

Тег попадает в сам SQL как комментарий, поэтому в Seq видно, какой метод сформировал запрос:

```sql
-- StudentController.GetStudents: список студентов

SELECT "s"."Id", "s"."Age", "s"."FirstName", "s"."LastName", "s"."MiddleName", "s"."NickName"
FROM "Students" AS "s"
ORDER BY "s"."LastName"
```

**Как выглядит SQL-событие в Seq.** EF Core пишет событие по шаблону `Executed DbCommand ({elapsed}ms) [Parameters=[{parameters}], …]{newLine}{commandText}`, поэтому у события есть **отдельные свойства**:

| Свойство | Содержимое | Пример |
|---|---|---|
| `commandText` | Текст SQL | `INSERT INTO "Students" (...) VALUES (@p0, ...) RETURNING "Id";` |
| `parameters` | Параметры запроса (значения — только при `EnableSensitiveDataLogging`) | `@p0='18', @p1='Иван' (Nullable = false) (Size = 4), …` |
| `elapsed` | Время выполнения в мс (**строка**) | `"2"` |
| `EventId.Name` | Тип события | `Microsoft.EntityFrameworkCore.Database.Command.CommandExecuted` |
| `RequestId`, `RequestPath`, `ActionName` | Из логирования запроса (работа №1) | позволяют связать SQL с HTTP-запросом |

> **Почему это делается.**
> - **Понимать, что делает ORM.** LINQ скрывает SQL. Без логов невозможно заметить, что простой на вид код выполняет десятки запросов (проблема *N+1*), выбирает лишние столбцы или сортирует данные не в БД.
> - **Почему `Override` только для одной категории.** В работе №1 порог для `Microsoft.*` поднят до `Warning`, иначе EF и ASP.NET Core «заваливают» лог служебными сообщениями (открытие соединений, компиляция запросов и т. п.). Категория `Microsoft.EntityFrameworkCore.Database.Command` — это *ровно* выполненные команды. Включаем только её — получаем все SQL-запросы без шума. Ошибки SQL (`CommandError`) пишутся уровнем `Error` и попадают в Seq при любой настройке.
> - **Все запросы — значит все.** В Seq попадут не только запросы контроллера, но и SQL миграций (`CREATE TABLE`, `__EFMigrationsHistory`) и начального заполнения (шаг 9) — весь жизненный цикл работы с БД.
> - **`EnableSensitiveDataLogging` только в Development.** По умолчанию EF скрывает значения параметров (`@p1='?'`), потому что там могут быть персональные данные, пароли и токены. В разработке значения нужны для отладки. В продакшене (в том числе в нашем Docker-контейнере, где окружение `Production`) запись персональных данных в логи недопустима (152-ФЗ, GDPR).
> - **`EnableDetailedErrors`** — более понятные сообщения об ошибках материализации (например, какой столбец вернул неожиданный `NULL`). Добавляет накладные расходы, поэтому тоже только для разработки.
> - **`TagWith`** — в большом приложении сотни похожих `SELECT`. Тег-комментарий однозначно связывает SQL с местом в коде, и его можно найти в Seq по `commandText like '%GetStudents%'`.
> - **Структурные свойства, а не текст.** Благодаря `commandText` и `elapsed` в Seq можно искать «все `INSERT`» или «запросы дольше 10 мс» как по полям БД. Поскольку `elapsed` записывается строкой, для числового сравнения используется `ToNumber(elapsed)`.
> - ⚠️ **В продакшене** логирование каждого запроса обычно выключают (уровень `Warning`) или оставляют только медленные запросы — из-за объёма логов и нагрузки.

---

### Шаг 6. Исправить обработку ошибок запуска для инструментов EF

В `Program.cs` из работы №1 замените строку

```csharp
catch (Exception ex)
```

на

```csharp
catch (Exception ex) when (ex is not HostAbortedException)
```

> **Почему это делается.** Команды `dotnet ef` запускают `Program.cs`, чтобы получить настроенный `CollegeDbContext` из DI. После получения контекста инструмент **намеренно прерывает** запуск хоста, выбрасывая `HostAbortedException`. Без фильтра `when` наш блок `catch` из работы №1 запишет это как `FTL Приложение аварийно завершилось при запуске` (в консоль и в Seq) при каждом создании миграции — ложная фатальная ошибка. Фильтр исключений пропускает этот штатный сценарий и оставляет логирование настоящих сбоев.

---

### Шаг 7. Создать первую миграцию

```bash
dotnet ef migrations add InitialCreate
```

В проекте появится папка `Migrations`:

| Файл | Назначение |
|---|---|
| `2026…_InitialCreate.cs` | Методы `Up` (применить изменения) и `Down` (откатить их) |
| `2026…_InitialCreate.Designer.cs` | Снимок модели на момент этой миграции (служебный) |
| `CollegeDbContextModelSnapshot.cs` | Текущий снимок всей модели. С ним сравнивается код при создании следующей миграции |

Откройте `…_InitialCreate.cs` и найдите в методе `Up`:

- `CreateTable("Students", …)` — столбцы, их типы (`INTEGER`, `TEXT`) и `nullable`. `Id` помечен `Sqlite:Autoincrement`, у `NickName` указан `collation: "NOCASE"`;
- `InsertData(…)` — два студента, перенесённые из контроллера;
- `CreateIndex("IX_Students_NickName", …, unique: true)`.

> **Почему это делается.**
> - **Миграция — это «diff» модели.** EF сравнивает текущий код модели со снимком (`ModelSnapshot`) и генерирует код, который переводит БД из старого состояния в новое. Первая миграция создаёт всё с нуля.
> - **Миграция — это код C#, а не готовый SQL.** Один и тот же файл применится к SQLite или PostgreSQL: SQL сгенерирует провайдер.
> - **Файлы миграций обязательно коммитятся в git** вместе с изменением модели. Это история схемы БД.
> - **Миграцию нужно просматривать перед применением.** Генератор может, например, удалить и пересоздать столбец при переименовании свойства — с потерей данных.
> - Если ошиблись в модели до применения миграции, удалите последнюю миграцию командой `dotnet ef migrations remove`, исправьте код и создайте её заново.

Посмотреть SQL, который будет выполнен (полезно для отчёта):

```bash
dotnet ef migrations script
```

---

### Шаг 8. Применить миграцию и посмотреть БД

**8.1. Применение вручную:**

```bash
dotnet ef database update
```

В папке проекта появится файл `college.db`.

**8.2. Автоматическое применение при запуске приложения.** В `Program.cs` сразу после `var app = builder.Build();` добавьте:

```csharp
    // Применяем недостающие миграции при запуске (создаёт БД, если её нет)
    using (var scope = app.Services.CreateScope())
    {
        var db = scope.ServiceProvider.GetRequiredService<CollegeDbContext>();
        db.Database.Migrate();
    }
```

**8.3. Просмотр БД.** Откройте `college.db` в DB Browser for SQLite, DBeaver, Rider (Database) или консольной утилите `sqlite3`:

```bash
sqlite3 college.db ".schema Students"
```

```bash
sqlite3 college.db "select * from Students; select * from __EFMigrationsHistory;"
```

Ожидаемый результат: таблица `Students` с двумя студентами и таблица `__EFMigrationsHistory` с одной записью `…_InitialCreate`.

**8.4.** Добавьте в `.gitignore` (в корне репозитория) файлы БД:

```gitignore
*.db
*.db-shm
*.db-wal
```

> **Почему это делается.**
> - **`__EFMigrationsHistory`** — журнал применённых миграций. По нему EF понимает, какие миграции уже выполнены, и при следующем `update`/`Migrate()` применяет только новые. Поэтому повторный запуск безопасен.
> - **`Migrate()` при старте.** Контейнер в Docker должен сам привести БД к актуальной схеме, ведь вручную запускать `dotnet ef` внутри контейнера неудобно (а SDK в runtime-образе нет). Для учебного проекта с одним экземпляром это самый простой надёжный вариант.
> - **Ограничение этого подхода:** если запущено несколько экземпляров приложения, они начнут мигрировать БД одновременно. Кроме того, у приложения должны быть права на изменение схемы. В продакшене миграции обычно применяют отдельным шагом развёртывания: SQL-скриптом (`dotnet ef migrations script --idempotent`) или исполняемым bundle (`dotnet ef migrations bundle`).
> - **`CreateScope()`.** Контекст зарегистрирован как Scoped, а при старте HTTP-запроса ещё нет. Поэтому область создаётся вручную, а `using` гарантирует освобождение контекста и подключения.
> - **Файл БД не хранится в git:** это *данные*, а не код. Схему восстанавливают миграции, а у каждого разработчика своя локальная БД.

---

### Шаг 9. Начальное заполнение БД (seed) через `UseSeeding`

В шаге 3 через `HasData` в БД попали два студента, перенесённые из контроллера. Теперь заполним БД **первоначальным набором данных** для работы и проверки приложения — пятью студентами учебной группы. Для этого используется механизм seeding EF Core (появился в EF Core 9): методы `UseSeeding` и `UseAsyncSeeding`.

**9.1.** Создайте файл `Data/StudentSeeder.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using MyCollegePlus.Models;

namespace MyCollegePlus.Data;

public static class StudentSeeder
{
    // Каждый вызов создаёт новые объекты: одни и те же экземпляры нельзя добавить в два разных контекста
    private static People[] InitialStudents() =>
    [
        new People { FirstName = "Иван", LastName = "Петров", MiddleName = "Сергеевич", Age = 18, NickName = "Ivan_P" },
        new People { FirstName = "Анна", LastName = "Смирнова", MiddleName = "Олеговна", Age = 19, NickName = "AnnaS" },
        new People { FirstName = "Дмитрий", LastName = "Козлов", MiddleName = null, Age = 20, NickName = "D1ma" },
        new People { FirstName = "Екатерина", LastName = "Новикова", MiddleName = "Андреевна", Age = 18, NickName = "Kate_N" },
        new People { FirstName = "Артём", LastName = "Морозов", MiddleName = "Игоревич", Age = 21, NickName = "M0roz" }
    ];

    public static void Seed(DbContext context)
    {
        var students = context.Set<People>();
        var seed = InitialStudents();
        var nickNames = seed.Select(s => s.NickName).ToList();

        // Какие из начальных студентов уже есть в БД (один SELECT ... WHERE NickName IN (...))
        var existing = students
            .Where(p => nickNames.Contains(p.NickName))
            .Select(p => p.NickName)
            .ToList();

        var missing = seed.Where(s => !existing.Contains(s.NickName, StringComparer.OrdinalIgnoreCase)).ToList();
        if (missing.Count == 0) return;

        students.AddRange(missing);
        context.SaveChanges();
    }

    public static async Task SeedAsync(DbContext context, CancellationToken ct)
    {
        var students = context.Set<People>();
        var seed = InitialStudents();
        var nickNames = seed.Select(s => s.NickName).ToList();

        var existing = await students
            .Where(p => nickNames.Contains(p.NickName))
            .Select(p => p.NickName)
            .ToListAsync(ct);

        var missing = seed.Where(s => !existing.Contains(s.NickName, StringComparer.OrdinalIgnoreCase)).ToList();
        if (missing.Count == 0) return;

        students.AddRange(missing);
        await context.SaveChangesAsync(ct);
    }
}
```

**9.2.** Подключите seeder при регистрации контекста. Итоговый вид `AddDbContext` в `Program.cs`:

```csharp
    builder.Services.AddDbContext<CollegeDbContext>(options =>
    {
        options.UseSqlite(builder.Configuration.GetConnectionString("CollegeDb"))
            .UseSeeding((context, _) => StudentSeeder.Seed(context))
            .UseAsyncSeeding((context, _, ct) => StudentSeeder.SeedAsync(context, ct));

        // Значения параметров SQL и подробные ошибки — только в разработке
        if (builder.Environment.IsDevelopment())
        {
            options.EnableSensitiveDataLogging()
                .EnableDetailedErrors();
        }
    });
```

Вызывать seeder вручную не нужно: EF Core запускает его сам **после применения миграций** — в `db.Database.Migrate()` (шаг 8), а также при `dotnet ef database update`.

**9.3. Проверка.** Удалите старую БД и запустите приложение:

```bash
rm -f college.db college.db-shm college.db-wal
```

```bash
dotnet run --launch-profile http
```

```bash
sqlite3 college.db "select Id, LastName, NickName from Students;"
```

Ожидается **7 студентов**: Id 1–2 — из `HasData` (миграция), Id 3–7 — из seeder'а. Перезапустите приложение и повторите запрос: студентов по-прежнему 7, дубликатов нет.

В Seq найдите SQL начального заполнения:

```
commandText like '%"NickName" IN (%' or commandText like 'INSERT INTO "Students"%'
```

При первом запуске будут `SELECT … WHERE "NickName" IN (…)` и пять `INSERT`, при повторном — только `SELECT`.

> **Почему это делается.**
> - **Два механизма заполнения — разные задачи.**
>
>   | | `HasData` (шаг 3) | `UseSeeding` / `UseAsyncSeeding` |
>   |---|---|---|
>   | Где живут данные | В модели и **в миграции** (как `InsertData`) | В обычном коде C#, выполняется при запуске |
>   | Изменение данных | Требует новой миграции | Достаточно изменить код |
>   | Ключи | `Id` обязательно задаётся вручную | `Id` генерирует БД |
>   | Логика | Нет: только константы | Любая: проверки, связи, чтение из файла, вычисляемые значения |
>   | Подходит для | Справочников, которые являются частью схемы (статусы, роли) и данных, перенесённых из кода | Первоначальных и демонстрационных данных, тестовых наборов |
>
>   Студенты из контроллера — фиксированные данные, без которых старое API «теряет» записи, поэтому они в `HasData`. Учебная группа — первоначальное наполнение, которое удобно менять, не создавая миграций, поэтому оно в seeder'е.
> - **Почему не просто код после `Migrate()`.** `UseSeeding` — штатная точка расширения EF: она вызывается и из приложения, и из `dotnet ef database update`, и из `EnsureCreated()`. Заполнение не «забудут» вызвать ни в одном сценарии.
> - **Синхронная и асинхронная версии.** `Migrate()` и `EnsureCreated()` вызывают `UseSeeding`, а `MigrateAsync()` и `EnsureCreatedAsync()` — `UseAsyncSeeding`; какой из них вызовут инструменты EF, зависит от версии. Документация EF требует реализовывать **обе** с одинаковой логикой, иначе при одном из способов запуска данные не появятся.
> - **Идемпотентность.** Seeder выполняется **при каждом запуске**, поэтому обязан быть идемпотентным: повторный запуск не должен создавать дубликаты. Мы добавляем только тех начальных студентов, чьих ников ещё нет. Проверка — одним запросом `IN (...)`, а не отдельным запросом на каждого студента. Сравнение без учёта регистра соответствует collation `NOCASE` в БД. Особенность: если удалить начального студента через API, при следующем запуске он появится снова. Если это нежелательно, проверяют «есть ли вообще данные» (`if (students.Any()) return;`).
> - **Новые объекты на каждый вызов.** Если хранить начальных студентов в статическом массиве, после первого `SaveChanges` у объектов появятся `Id`, и они окажутся привязаны к уже уничтоженному контексту. Метод `InitialStudents()` каждый раз создаёт свежие экземпляры.
> - **SQL заполнения виден в Seq** (шаг 5): при запуске видно, что именно и сколько записей добавил seeder.

---

### Шаг 10. Читать данные из БД в контроллере

Замените `Controllers/StudentController.cs` (действия `Fail` и `Crash` из работы №1 оставлены без изменений):

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using MyCollegePlus.Data;
using MyCollegePlus.Models;
using MyCollegePlus.Models.Dto;

namespace MyCollegePlus.Controllers;

[ApiController]
[Route("api/[controller]")]
public class StudentController: ControllerBase
{
    private readonly ILogger<StudentController> _logger;
    private readonly CollegeDbContext _db;

    public StudentController(ILogger<StudentController> logger, CollegeDbContext db)
    {
        _logger = logger;
        _db = db;
    }

    [HttpGet("get-students")]
    public async Task<IActionResult> GetStudents(CancellationToken ct)
    {
        var students = await _db.Students
            .TagWith("StudentController.GetStudents: список студентов")
            .AsNoTracking()
            .OrderBy(p => p.LastName)
            .ToListAsync(ct);

        _logger.LogInformation("Запрошен список студентов, всего записей: {StudentCount}", students.Count);
        return Ok(students);
    }

    [HttpGet("get-student/{nickName}")]
    public async Task<IActionResult> GetStudent(string nickName, CancellationToken ct)
    {
        using (_logger.BeginScope(new Dictionary<string, object> { ["SearchNickName"] = nickName }))
        {
            _logger.LogDebug("Поиск студента по нику {NickName}", nickName);

            var student = await _db.Students
                .TagWith("StudentController.GetStudent: поиск по нику")
                .AsNoTracking()
                .FirstOrDefaultAsync(p => p.NickName == nickName, ct);

            if (student is null)
            {
                _logger.LogWarning("Студент с ником {NickName} не найден", nickName);
                return NotFound();
            }

            _logger.LogInformation("Найден студент {@Student}", student);
            return Ok(student);
        }
    }

    [HttpPost("add-student")]
    public async Task<IActionResult> AddStudent(CreateStudentRequest request, CancellationToken ct)
    {
        if (await _db.Students.AnyAsync(p => p.NickName == request.NickName, ct))
        {
            _logger.LogWarning("Попытка добавить студента с занятым ником {NickName}", request.NickName);
            return Conflict(new { message = $"Ник {request.NickName} уже занят" });
        }

        var student = new People
        {
            FirstName = request.FirstName,
            LastName = request.LastName,
            MiddleName = request.MiddleName,
            Age = request.Age,
            NickName = request.NickName
        };

        _db.Students.Add(student);
        await _db.SaveChangesAsync(ct);

        _logger.LogInformation("Добавлен студент {NickName} с Id {StudentId}", student.NickName, student.Id);
        return CreatedAtAction(nameof(GetStudent), new { nickName = student.NickName }, student);
    }

    [HttpGet("fail")]
    public IActionResult Fail()
    {
        try
        {
            throw new InvalidOperationException("Тестовая ошибка для проверки логирования");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Не удалось обработать запрос {Path}", HttpContext.Request.Path);
            return Problem("Внутренняя ошибка сервера");
        }
    }

    [HttpGet("crash")]
    public IActionResult Crash()
    {
        throw new ApplicationException("Необработанное исключение");
    }
}
```

Проект пока не скомпилируется: `CreateStudentRequest` создаём на шаге 11. Сначала разберём чтение.

> **Почему это делается (чтение).**
> - **Массив `_pepoples` удалён полностью** — единственным источником данных стала БД. Если оставить данные в двух местах, они неизбежно разойдутся.
> - **`CollegeDbContext` внедряется через конструктор** рядом с `ILogger`. Контроллер не создаёт подключение сам и не знает, какая СУБД используется.
> - **LINQ → SQL.** `OrderBy(...).ToListAsync()` не сортирует данные в памяти: EF переводит выражение в `SELECT … ORDER BY "LastName"`, и сортировку выполняет БД. Запрос отправляется только в момент `ToListAsync`/`FirstOrDefaultAsync` (*отложенное выполнение*). Сам SQL вы увидите в Seq (шаг 13).
> - **`TagWith(...)`** добавляет в SQL комментарий с именем метода, чтобы в Seq было видно, откуда пришёл запрос (см. шаг 5).
> - **`AsNoTracking()`** — данные только читаются и отдаются клиенту. Отслеживание изменений не нужно, а без него запрос работает быстрее и расходует меньше памяти. Для изменяемых сущностей (как в `AddStudent`) отслеживание, наоборот, необходимо.
> - **`p.NickName == nickName`** выполняется в БД. Благодаря collation `NOCASE` (шаг 3) сравнение идёт без учёта регистра, и `ToLower()` не нужен: он помешал бы БД использовать индекс.
> - **`async Task<IActionResult>` и `CancellationToken`** — см. теорию: поток не блокируется на время запроса, а запрос к БД отменяется, если клиент ушёл.
> - **`Ok(students)` вместо `new JsonResult(...)`** — стандартный результат ASP.NET Core, учитывающий заголовок `Accept` (согласование формата). Фильтр из работы №1 по-прежнему записывает статус и тип результата.
> - **Ручные логи из работы №1 сохранены**, а `StudentCount` теперь отражает реальное количество строк в БД.

---

### Шаг 11. DTO и endpoint добавления студента

Создайте папку `Models/Dto` и файл `Models/Dto/CreateStudentRequest.cs`:

```csharp
using System.ComponentModel.DataAnnotations;

namespace MyCollegePlus.Models.Dto;

public record CreateStudentRequest(
    [Required, StringLength(100)] string FirstName,
    [Required, StringLength(100)] string LastName,
    [StringLength(100)] string? MiddleName,
    [Range(14, 120)] uint Age,
    [Required, StringLength(50, MinimumLength = 3)] string NickName);
```

Соберите проект:

```bash
dotnet build
```

Разберите действие `AddStudent` из шага 10.

> **Почему это делается.**
> - **DTO вместо сущности.** В DTO нет `Id`: клиент не может навязать первичный ключ, его генерирует БД (защита от overposting). Модель API и модель хранения можно менять независимо. Здесь record уместен: DTO — неизменяемые данные «на один раз».
> - **Валидация атрибутами (DataAnnotations).** Благодаря `[ApiController]` невалидная модель автоматически отклоняется с ответом `400 Bad Request` и перечнем ошибок (формат RFC 9110 Problem Details), до входа в метод. Валидация на границе доверия (входные данные от клиента) обязательна. Ограничения длины совпадают с `HasMaxLength` в контексте.
> - **Проверка уникальности ника → `409 Conflict`.** Это бизнес-правило, которое лучше сообщить клиенту понятным ответом, а не ошибкой БД `500`. Уникальный индекс в БД остаётся «последним рубежом» на случай гонки двух одновременных запросов. В этом случае `SaveChangesAsync` выбросит `DbUpdateException`, и её залогирует фильтр из работы №1.
> - **`Add` + `SaveChangesAsync`.** `Add` только помечает объект как `Added` в трекере изменений. Реальный `INSERT` выполняется в `SaveChangesAsync` в рамках транзакции. После сохранения EF записывает в `student.Id` значение, сгенерированное БД.
> - **`201 Created` + `CreatedAtAction`** — по стандарту REST ответ на создание ресурса содержит заголовок `Location` с адресом нового ресурса (`/api/Student/get-student/{nickName}`) и сам созданный объект.
> - **Логи:** `Warning` — нарушение бизнес-правила, `Information` — успешное добавление с `StudentId`. В Seq можно найти всех добавленных студентов запросом `StudentId is not null`.

---

### Шаг 12. Хранение БД при запуске в Docker

Отредактируйте сервис `backend` в `docker-compose.yml` (переменные Seq из работы №1 сохраняются):

```yaml
  backend:
      container_name: backend-college
      build:
        context: ./code/backend/MyCollegePlus
        dockerfile: Dockerfile
      depends_on:
        - seq
      ports:
        - "8082:80"
      environment:
        ASPNETCORE_HTTP_PORTS: 80
        Serilog__WriteTo__1__Args__serverUrl: http://seq:5341
        Serilog__Properties__Environment: Docker
        # Файл БД — в смонтированном каталоге, чтобы пережить пересоздание контейнера
        ConnectionStrings__CollegeDb: Data Source=/app/data/college.db
      volumes:
        - ./docker/sqlite-data:/app/data
      networks:
        - mycollegeplus-net
```

Пересоберите и запустите:

```bash
docker compose up -d --build backend
```

```bash
curl http://localhost:8082/api/Student/get-students
```

Убедитесь, что на хосте появился файл `docker/sqlite-data/college.db`. Добавьте студента (шаг 13), выполните `docker compose down`, затем `docker compose up -d backend` и проверьте, что студент сохранился.

> **Почему это делается.**
> - **Файловая система контейнера временная.** При `docker compose down` или пересборке образа контейнер удаляется вместе со всеми файлами внутри, включая `college.db`. Поэтому файл БД выносится в смонтированный каталог хоста — так же, как в проекте уже хранятся данные PostgreSQL, MongoDB и Seq (`./docker/*-data`).
> - **Монтируется каталог, а не сам файл.** SQLite рядом с файлом БД создаёт служебные файлы (`-wal`, `-shm`, журнал), и им нужен доступ на запись к каталогу.
> - **Путь переопределяется переменной окружения** `ConnectionStrings__CollegeDb` — тот же механизм, что и для адреса Seq: `__` заменяет `:` в иерархии настроек.
> - **Миграции применятся сами** при старте контейнера благодаря `Migrate()` (шаг 8), после чего выполнится seed (шаг 9). При первом запуске создаётся БД с начальными данными, при последующих — ничего не меняется.
> - **В контейнере окружение `Production`**, поэтому SQL-запросы попадут в Seq **без значений параметров** (`@p1='?'`) — так работает защита из шага 5. Сравните в Seq события с `Environment = 'Docker'` и локальные.
> - Добавьте `docker/sqlite-data/` в `.gitignore`, как и другие каталоги данных.

---

### Шаг 13. Проверка работы

Обновите `MyCollegePlus.http`:

```http
@MyCollegePlus_HostAddress = http://localhost:5251

GET {{MyCollegePlus_HostAddress}}/api/Student/get-students
###
GET {{MyCollegePlus_HostAddress}}/api/Student/get-student/w4rl0ck
###
POST {{MyCollegePlus_HostAddress}}/api/Student/add-student
Content-Type: application/json

{
  "firstName": "Пётр",
  "lastName": "Иванов",
  "middleName": null,
  "age": 19,
  "nickName": "Petr_I"
}
###
# Повторное добавление (ник в другом регистре) — ожидается 409
POST {{MyCollegePlus_HostAddress}}/api/Student/add-student
Content-Type: application/json

{
  "firstName": "Пётр",
  "lastName": "Сидоров",
  "age": 20,
  "nickName": "petr_i"
}
###
# Невалидные данные — ожидается 400
POST {{MyCollegePlus_HostAddress}}/api/Student/add-student
Content-Type: application/json

{
  "firstName": "",
  "lastName": "Петров",
  "age": 5,
  "nickName": "ab"
}
###
```

Запустите приложение:

```bash
dotnet run --launch-profile http
```

Выполните запросы по очереди. Ожидаемые результаты:

| Запрос | Ожидаемый ответ |
|---|---|
| `get-students` | `200`, 7 студентов (2 из `HasData` + 5 из seeder'а), отсортированы по фамилии |
| `get-student/w4rl0ck` | `200`, студент `W4rl0ck` (поиск без учёта регистра) |
| `add-student` (Пётр Иванов) | `201 Created`, заголовок `Location: …/get-student/Petr_I`, в теле `"id": 8` |
| `add-student` (`petr_i`) | `409 Conflict`, «Ник petr_i уже занят» |
| `add-student` (невалидный) | `400`, ошибки по полям `FirstName`, `Age`, `NickName` |
| `get-students` повторно | `200`, 8 студентов |

Попробуйте также добавить студента с ником из начальных данных (например, `annas`) — тоже `409`.

Остановите приложение (`Ctrl+C`), запустите снова и выполните `get-student/PETR_I` — студент должен найтись, а `get-students` вернёт те же 8 записей (seeder не создал дубликатов). **Данные пережили перезапуск** — это главное отличие от массива в контроллере.

**Проверка в Seq** (`http://seq.localhost`):

| Запрос Seq | Что показывает |
|---|---|
| `SourceContext = 'Microsoft.EntityFrameworkCore.Database.Command'` | Все SQL-запросы, сформированные EF Core |
| `commandText like 'INSERT%'` | SQL добавления студентов (seeder и `add-student`) |
| `commandText like '%StudentController.GetStudents%'` | SQL, помеченный `TagWith` в `GetStudents` |
| `commandText like '%CREATE TABLE%'` | SQL миграций |
| `parameters like '%Petr_I%'` | Запросы с конкретным значением параметра (только в Development) |
| `ToNumber(elapsed) > 10` | Медленные SQL-запросы (дольше 10 мс) |
| `RequestPath = '/api/Student/add-student' and has(commandText)` | Все SQL одного endpoint |
| `select count(*) from stream where has(commandText) group by RequestPath` | Количество SQL-запросов по endpoint'ам |
| `StudentId is not null` | События добавления студентов |
| `@Message like '%занятым ником%'` | Попытки добавить дубликат |

Откройте любой запрос `get-students` и с помощью `RequestId` (см. работу №1) найдите все его события: начало action → SQL `SELECT` → ручной лог → конец action → итог HTTP-запроса.

> **Почему это делается.** Проверяются все требования цели: начальное заполнение, чтение, добавление, валидация, бизнес-правило, сохранность после перезапуска. Просмотр SQL в Seq учит контролировать, во что ORM превращает LINQ. Это главный навык, позволяющий избежать медленных запросов.

---

## Индивидуальные задания

Выполните одно задание по номеру в журнале (№ mod 4 + 1):

1. **Эволюция схемы.** Добавьте в `People` необязательное поле `Email` (до 200 символов, уникальное, если указано), добавьте его в DTO с атрибутом `[EmailAddress]`. Создайте миграцию `AddStudentEmail`, изучите её код и `__EFMigrationsHistory` после применения. Объясните, почему существующие данные не пропали.
2. **Изменение и удаление.** Реализуйте `PUT api/Student/update-student/{id}` и `DELETE api/Student/delete-student/{id}` с ответами `204`/`404`/`409` и логами. Объясните, как работает отслеживание изменений при обновлении.
3. **Связь один-ко-многим.** Создайте сущность `Group` (`Id`, `Name`) и связь «группа — студенты» (`GroupId` в `People`). Добавьте seed для 2 групп, миграцию и endpoint `GET api/Group/{id}/students` (используйте `Include`). Покажите сгенерированный SQL (`JOIN`) в Seq.
4. **Постраничный вывод и поиск.** Доработайте `get-students`: параметры `page`, `pageSize` (не более 50) и `search` (поиск по фамилии). Используйте `Skip/Take`, верните общее количество записей. Покажите в Seq SQL с `LIMIT/OFFSET`.

## Содержание отчёта

1. Титульный лист, цель работы.
2. Листинги: `People.cs`, `CollegeDbContext.cs`, `StudentSeeder.cs`, `CreateStudentRequest.cs`, `StudentController.cs`, изменённые фрагменты `Program.cs`, `appsettings.json`, `docker-compose.yml`.
3. Код миграции `InitialCreate` (метод `Up`) и вывод `dotnet ef migrations script`.
4. Скриншоты: структура таблицы `Students` и её содержимое после seed (7 записей); `__EFMigrationsHistory`; ответы всех запросов из шага 13; SQL-запросы в Seq (минимум: SQL миграции, `INSERT` seeder'а, `SELECT` с тегом `TagWith`, SQL с параметрами, связанные по `RequestId` события одного запроса); данные после перезапуска контейнера.
5. Результат индивидуального задания.
6. Ответы на контрольные вопросы, выводы.

---

## Контрольные вопросы

1. Что такое ORM? Какие преимущества и недостатки у ORM по сравнению с SQL-запросами через ADO.NET?
2. Чем подход Code First отличается от Database First? В каких ситуациях выбирают каждый из них?
3. Почему данные в массиве-поле контроллера нельзя изменять между запросами? Каково время жизни контроллера в ASP.NET Core?
4. Почему для учебного проекта выбрана SQLite? Назовите ограничения SQLite, из-за которых в продакшене выбирают PostgreSQL.
5. Какую роль выполняют `DbContext` и `DbSet<T>`? Почему контекст регистрируется как Scoped?
6. По какому соглашению EF Core определяет первичный ключ? Как EF понимает, что столбец допускает `NULL`?
7. Чем Fluent API отличается от атрибутов DataAnnotations? Что из конфигурации этой работы нельзя выразить атрибутами?
8. Что такое миграция? Из каких файлов она состоит и зачем нужен `ModelSnapshot`?
9. Для чего нужна таблица `__EFMigrationsHistory`? Что произойдёт при повторном вызове `Migrate()`?
10. Какие риски у применения миграций при старте приложения? Как применяют миграции в продакшене?
11. Как с помощью `HasData` перенести данные из кода в БД? Почему в seed-данных `Id` задаётся явно?
12. Зачем нужен уникальный индекс, если в контроллере уже есть проверка `AnyAsync`? Что такое состояние гонки (race condition) в этом сценарии?
13. Зачем используется collation `NOCASE`? Почему `p.NickName.ToLower() == nickName.ToLower()` — худшее решение?
14. Что делает `AsNoTracking()` и когда его нельзя использовать?
15. Когда LINQ-запрос EF Core на самом деле отправляется в БД? Что такое отложенное выполнение?
16. Зачем методы доступа к данным делают асинхронными? Для чего передаётся `CancellationToken`?
17. Что такое DTO и overposting? Почему в `POST` принимается `CreateStudentRequest`, а не `People`?
18. Что делают `Add` и `SaveChangesAsync` по отдельности? Когда у нового студента появляется `Id`?
19. Почему успешное добавление возвращает `201 Created`, а не `200 OK`? Что содержит заголовок `Location`?
20. Почему в Docker файл БД монтируется томом, и почему монтируется каталог, а не сам файл?
21. Зачем в `catch` в `Program.cs` добавлен фильтр `when (ex is not HostAbortedException)`?
22. Как увидеть SQL, сгенерированный EF Core? Почему эту категорию логов включают на уровне `Information` только в разработке?
23. Чем отличаются механизмы seed `HasData` и `UseSeeding`? Для каких данных подходит каждый из них?
24. Что такое идемпотентность seed-кода? Как она обеспечена в `StudentSeeder`, и что произойдёт, если её не обеспечить?
25. Почему нужно реализовать и `UseSeeding`, и `UseAsyncSeeding`? Когда EF Core вызывает каждый из них?
26. Почему `InitialStudents()` — метод, создающий новые объекты, а не статический массив?
27. Почему для логирования SQL разрешена только категория `Microsoft.EntityFrameworkCore.Database.Command`, а не весь `Microsoft.EntityFrameworkCore`?
28. Что делает `EnableSensitiveDataLogging`? Почему он включается только в окружении Development?
29. Для чего используется `TagWith`? Как найти в Seq SQL-запросы конкретного метода контроллера?
30. Какие свойства есть у SQL-события EF Core в Seq? Как найти запросы дольше 10 мс и почему нужен `ToNumber(elapsed)`?
