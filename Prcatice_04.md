# Практическая работа. Централизованное логирование ASP.NET Core в Seq (Docker)

**Дисциплина:** Информационные системы / Разработка серверных приложений
**Проект:** MyCollegePlus (backend — ASP.NET Core 10, инфраструктура — Docker Compose)
**Продолжительность:** 4 академических часа

---

## Цель работы

Подключить логирование в Seq, развёрнутый в Docker. Перевести все системные логи приложения в Seq, а также реализовать логирование действий в контроллере:

- **автоматически** — весь цикл прохождения запроса (HTTP-запрос → вход в action → результат/ошибка → время выполнения);
- **вручную** — пользовательские логи разработчика на основе `ILogger<T>`.

## Результат, который нужно получить

1. В веб-интерфейсе Seq (`http://seq.localhost`) видны события запуска/остановки приложения, предупреждения ASP.NET Core и ошибки.
2. На каждый HTTP-запрос к API в Seq появляется цепочка событий: начало action → пользовательские логи → конец action → итог HTTP-запроса.
3. Все события **структурные**: по свойствам (`NickName`, `StatusCode`, `Elapsed`, `Controller` и т. д.) можно фильтровать и строить запросы.
4. Приложение логирует в Seq как при локальном запуске (`dotnet run`), так и в контейнере `backend`.

## Теоретическая часть (кратко)

**Почему не хватает `Console.WriteLine` и стандартного консольного лога?** Консольный вывод живёт только в терминале или в `docker logs` конкретного контейнера, ограничен по размеру (в нашем `docker-compose.yml` — 3 файла по 10 МБ) и представляет собой обычный текст. Искать в нём «все ошибки за вчера у студента с ником X» неудобно.

**Структурное логирование** — сообщение пишется по *шаблону* (`"Студент {NickName} не найден"`), а значения подстановок сохраняются как отдельные именованные свойства. Получается не строка, а объект (JSON), по полям которого можно искать, как в базе данных.

**Seq** — сервер сбора и анализа структурных логов. Приложение отправляет события по HTTP на порт приёма (`5341`), а разработчик смотрит их в веб-интерфейсе с фильтрами, SQL-подобными запросами, графиками и оповещениями.

**ILogger и Serilog.** `ILogger<T>` — стандартная абстракция логирования .NET (`Microsoft.Extensions.Logging`). Код приложения пишет логи только через неё. **Serilog** — библиотека, которая подключается «под» `ILogger` в качестве провайдера и умеет отправлять события в разные места (*sinks*): консоль, файл, Seq и т. д. Так код контроллеров не зависит от того, куда в итоге уходят логи.

**Уровни логирования** (от подробного к критическому): `Trace/Verbose` → `Debug` → `Information` → `Warning` → `Error` → `Critical/Fatal`.

## Анализ исходного проекта

Перед началом изучите, что уже есть в проекте:

| Файл | Что в нём важно для работы |
|---|---|
| `docker-compose.yml` | Сервис `seq` (образ `datalust/seq:2026.1.17083`), порт приёма логов `5341` опубликован на `127.0.0.1`, данные в `./docker/seq-data`. Сервис `nginx` проксирует веб-интерфейс Seq на `http://seq.localhost`. Сервис `backend` собирается из `code/backend/MyCollegePlus`, но **ничего не знает о Seq**. Все сервисы в одной сети `mycollegeplus-net`. |
| `docker/nginx/default.conf` | Виртуальный хост `seq.localhost` → `proxy_pass http://seq:80`. |
| `code/backend/MyCollegePlus/Program.cs` | Минимальное приложение: контроллеры, OpenAPI. Логирование — стандартное (только консоль). |
| `appsettings.json` | Секция `Logging` — уровни встроенного логгера. |
| `Controllers/StudentController.cs` | Один метод `GetStudents()`, логов нет, конструктор пустой. |
| `Dockerfile` | В `HEALTHCHECK` вызывается `http://localhost:80/health`, но такого endpoint в приложении **нет** — контейнер будет `unhealthy`. Исправим попутно. |
| `MyCollegePlus.http` | Ссылается на несуществующий `/weatherforecast/` — заменим на реальные запросы. |

**Вывод:** инфраструктура (Seq + Nginx) готова, но приложение к ней не подключено. Наша задача — «соединить» backend и Seq и наполнить приложение осмысленными логами.

---

## Ход работы

### Шаг 1. Запустить Seq и проверить его доступность

```bash
docker compose up -d seq nginx
```

```bash
docker compose ps
```

Откройте в браузере `http://seq.localhost`. Должен открыться интерфейс Seq с пустым (или почти пустым) списком событий.

> **Почему это делается.** Прежде чем настраивать отправку логов, нужно убедиться, что получатель работает. Иначе при отсутствии событий в Seq будет непонятно, где ошибка — в приложении или в инфраструктуре. Обратите внимание на два адреса Seq: веб-интерфейс идёт через Nginx (`seq.localhost`, порт 80), а **приём** логов — напрямую на порт `5341`. Порт `5341` принимает только события и не отдаёт интерфейс, поэтому его безопаснее открывать приложениям.

**Контроль:** скриншот пустого интерфейса Seq.

---

### Шаг 2. Подключить пакеты Serilog

Перейдите в папку backend и установите пакеты:

```bash
cd code/backend/MyCollegePlus
```

```bash
dotnet add package Serilog.AspNetCore
```

```bash
dotnet add package Serilog.Sinks.Seq
```

```bash
dotnet add package Serilog.Enrichers.Environment
```

Проверьте `MyCollegePlus.csproj` — должны появиться строки (версии могут быть новее):

```xml
<PackageReference Include="Serilog.AspNetCore" Version="10.0.0" />
<PackageReference Include="Serilog.Enrichers.Environment" Version="3.0.1" />
<PackageReference Include="Serilog.Sinks.Seq" Version="9.1.0" />
```

> **Почему это делается.**
> - `Serilog.AspNetCore` — интеграция Serilog с хостом ASP.NET Core: подменяет провайдера `ILogger`, добавляет middleware логирования запросов, а также включает в себя консольный sink и чтение настроек из `appsettings.json`.
> - `Serilog.Sinks.Seq` — «приёмник», который пакетами отправляет события в Seq по HTTP.
> - `Serilog.Enrichers.Environment` — добавляет к каждому событию имя машины. В Docker это ID контейнера, что помогает понять, какой экземпляр приложения записал событие, когда их несколько.

---

### Шаг 3. Описать конфигурацию логирования в `appsettings.json`

Замените содержимое `appsettings.json`:

```json
{
  "Serilog": {
    "Using": [ "Serilog.Sinks.Console", "Serilog.Sinks.Seq" ],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.AspNetCore": "Warning",
        "Microsoft.Hosting.Lifetime": "Information",
        "System": "Warning",
        "MyCollegePlus": "Debug"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {SourceContext}: {Message:lj}{NewLine}{Exception}"
        }
      },
      {
        "Name": "Seq",
        "Args": { "serverUrl": "http://localhost:5341" }
      }
    ],
    "Enrich": [ "FromLogContext", "WithMachineName" ],
    "Properties": { "Application": "MyCollegePlus" }
  },
  "AllowedHosts": "*"
}
```

В `appsettings.Development.json` удалите секцию `Logging` (оставьте пустой объект `{ }`).

> **Почему это делается.**
> - **Настройки в конфигурации, а не в коде** — уровень логов и адрес Seq можно менять без перекомпиляции: через `appsettings.{Environment}.json` или переменные окружения Docker (это используем в шаге 8).
> - Секция `Logging` больше не нужна: после подключения Serilog уровни читаются из `Serilog:MinimumLevel`. Если оставить обе секции, студенту (и коллеге) неочевидно, какая из них действует.
> - **`Override`** — фильтр по *источнику* (`SourceContext`, обычно полное имя класса). Фреймворк Microsoft очень «болтлив» на уровне `Information` (несколько событий на каждый запрос), поэтому для него поднимаем порог до `Warning`. Но `Microsoft.Hosting.Lifetime` оставляем на `Information` — это системные события запуска и остановки приложения, они нужны. Для собственного кода (`MyCollegePlus`) разрешаем `Debug`, чтобы видеть подробности.
> - **Два sink'а**: консоль — для разработчика и `docker logs`, Seq — для централизованного анализа.
> - **`Properties.Application`** — свойство, добавляемое к каждому событию. Когда в один Seq пишут несколько сервисов, по нему их легко различить.
> - **`FromLogContext`** — включает свойства из `BeginScope` и `LogContext` (используем в шаге 6).

---

### Шаг 4. Подключить Serilog в `Program.cs`

Замените `Program.cs`:

```csharp
using MyCollegePlus.Filters;
using Serilog;
using Serilog.Events;

// Временный логгер: пишет ошибки, возникшие ещё ДО чтения конфигурации
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .CreateBootstrapLogger();

try
{
    Log.Information("Запуск приложения MyCollegePlus");

    var builder = WebApplication.CreateBuilder(args);

    // Serilog становится провайдером для всех ILogger<T> в приложении
    builder.Services.AddSerilog((services, lc) => lc
        .ReadFrom.Configuration(builder.Configuration)
        .ReadFrom.Services(services));

    builder.Services.AddOpenApi();
    // Глобальный фильтр автоматического логирования действий (шаг 5)
    builder.Services.AddControllers(options => options.Filters.Add<LogActionFilter>());

    var app = builder.Build();

    // Одно итоговое событие на каждый HTTP-запрос
    app.UseSerilogRequestLogging(options =>
    {
        options.MessageTemplate =
            "HTTP {RequestMethod} {RequestPath} ответил {StatusCode} за {Elapsed:0.0000} мс";

        // Ошибки — Error, healthcheck — Verbose (не попадает в Seq), остальное — Information
        options.GetLevel = (http, elapsed, ex) =>
            ex is not null || http.Response.StatusCode >= 500 ? LogEventLevel.Error
            : http.Request.Path.StartsWithSegments("/health") ? LogEventLevel.Verbose
            : LogEventLevel.Information;

        // Дополнительные свойства запроса
        options.EnrichDiagnosticContext = (diag, http) =>
        {
            diag.Set("ClientIp", http.Connection.RemoteIpAddress?.ToString());
            diag.Set("UserAgent", http.Request.Headers.UserAgent.ToString());
        };
    });

    if (app.Environment.IsDevelopment())
    {
        app.MapOpenApi();
    }

    app.UseHttpsRedirection();

    app.MapGet("/health", () => Results.Ok("healthy"));
    app.MapControllers();

    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Приложение аварийно завершилось при запуске");
}
finally
{
    // Отправить в Seq всё, что осталось в буфере
    Log.CloseAndFlush();
}
```

> **Почему это делается.**
> - **Bootstrap-логгер и `try/catch`.** Если приложение упадёт при старте (ошибка в конфигурации, недоступна БД), основной логгер ещё не создан, и ошибка «потеряется». Временный логгер гарантирует, что фатальная ошибка запуска будет записана.
> - **`AddSerilog(...)`** перенаправляет **все** системные логи (хост, Kestrel, MVC, маршрутизация, HTTPS-редирект и т. д.) и логи нашего кода в Serilog, а значит — в Seq. Это и есть «перевод всех системных логов в Seq»: менять код фреймворка не нужно, он и так пишет через `ILogger`.
> - **`ReadFrom.Services`** позволяет Serilog использовать сервисы из DI (например, собственные enricher'ы).
> - **`UseSerilogRequestLogging`** заменяет 3–4 разрозненных события ASP.NET Core на запрос одним событием с методом, путём, статус-кодом и временем. Middleware стоит **в начале** конвейера, чтобы измерять полное время запроса и видеть все последующие компоненты.
> - **`GetLevel`.** Healthcheck Docker вызывает `/health` каждые 5 секунд (см. `Dockerfile`) — это 17 280 бесполезных событий в сутки. Понижаем их до `Verbose`, и они отсекаются фильтром `MinimumLevel`. Ответы 5xx повышаем до `Error`, чтобы их было легко найти.
> - **`/health`** — добавлен, потому что `HEALTHCHECK` в `Dockerfile` обращается к нему, а без него контейнер навсегда остаётся `unhealthy`.
> - **`Log.CloseAndFlush()`.** Sink Seq отправляет события пакетами раз в несколько секунд. Без явного сброса последние события перед остановкой (в том числе причина падения) не дойдут до Seq.

**Не запускайте проект** — он пока не скомпилируется: класс `LogActionFilter` создадим на следующем шаге.

---

### Шаг 5. Автоматическое логирование цикла действия контроллера (Action Filter)

Создайте папку `Filters` и файл `Filters/LogActionFilter.cs`:

```csharp
using System.Diagnostics;
using Microsoft.AspNetCore.Mvc.Filters;
using Microsoft.AspNetCore.Mvc.Infrastructure;

namespace MyCollegePlus.Filters;

public class LogActionFilter : IAsyncActionFilter
{
    private readonly ILogger<LogActionFilter> _logger;

    public LogActionFilter(ILogger<LogActionFilter> logger)
    {
        _logger = logger;
    }

    public async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
    {
        var controller = context.RouteData.Values["controller"];
        var action = context.RouteData.Values["action"];

        // 1. До выполнения action: что вызвали и с какими параметрами
        _logger.LogInformation("Начало {Controller}.{Action}, аргументы: {@ActionArguments}",
            controller, action, context.ActionArguments);

        var stopwatch = Stopwatch.StartNew();
        var executed = await next();   // 2. Выполнение самого action
        stopwatch.Stop();

        // 3а. Action завершился необработанным исключением
        if (executed.Exception is not null && !executed.ExceptionHandled)
        {
            _logger.LogError(executed.Exception, "Ошибка в {Controller}.{Action} через {ElapsedMs:0.00} мс",
                controller, action, stopwatch.Elapsed.TotalMilliseconds);
            return;
        }

        // 3б. Успешное (или ожидаемое неуспешное, например 404) завершение
        var statusCode = (executed.Result as IStatusCodeActionResult)?.StatusCode
                         ?? context.HttpContext.Response.StatusCode;

        _logger.LogInformation(
            "Конец {Controller}.{Action}: результат {ResultType}, статус {StatusCode}, {ElapsedMs:0.00} мс",
            controller, action, executed.Result?.GetType().Name, statusCode, stopwatch.Elapsed.TotalMilliseconds);
    }
}
```

Фильтр уже зарегистрирован глобально в `Program.cs`: `options.Filters.Add<LogActionFilter>()`.

> **Почему это делается.**
> - **Сквозная функциональность (cross-cutting concern).** Логировать вход/выход в каждом методе каждого контроллера вручную — это дублирование кода, и кто-нибудь обязательно забудет. Фильтр MVC выполняется вокруг **любого** action автоматически.
> - **Почему фильтр, а не middleware.** Middleware не знает, какой контроллер и action выбраны и с какими аргументами (модель уже привязана к параметрам только к моменту фильтров). Фильтр работает внутри MVC и имеет доступ к `ActionArguments` и `Result`.
> - **`IAsyncActionFilter` вместо пары `OnActionExecuting/OnActionExecuted`** — один метод, поэтому `Stopwatch` и переменные живут в одной области видимости, а `await next()` чётко разделяет «до» и «после».
> - **`{@ActionArguments}`** — символ `@` означает *деструктуризацию*: объект сохраняется в Seq как вложенная структура, а не как результат `ToString()`.
> - **Регистрация через `Add<T>()`** — фильтр создаётся через DI, поэтому `ILogger<LogActionFilter>` внедряется в конструктор автоматически.
> - ⚠️ **Безопасность:** если в action передаются пароли, токены или персональные данные, логировать аргументы целиком нельзя. В реальных проектах такие поля маскируют или исключают.

**Полный цикл одного запроса в логах** (сверху вниз):

```
LogActionFilter        → Начало Student.GetStudent, аргументы: {"nickName": "w4rl0ck"}
StudentController      → Поиск студента по нику w4rl0ck               (ручной, Debug)
StudentController      → Найден студент {...}                          (ручной, Information)
LogActionFilter        → Конец Student.GetStudent: OkObjectResult, 200, 0.35 мс
RequestLoggingMiddleware → HTTP GET /api/Student/get-student/w4rl0ck ответил 200 за 16.4 мс
```

---

### Шаг 6. Ручное (пользовательское) логирование в контроллере через `ILogger`

Измените `Controllers/StudentController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using MyCollegePlus.Models;

namespace MyCollegePlus.Controllers;

[ApiController]
[Route("api/[controller]")]
public class StudentController: ControllerBase
{
    private readonly ILogger<StudentController> _logger;

    private readonly People[] _pepoples =
    [
        new People("Максим", "Рубченко", "Ильич", 39, "W4rl0ck"),
        new People("Александр", "Донцов", "Виторович", 40, "R0l4nd")
    ];
    
    public StudentController(ILogger<StudentController> logger)
    {
        _logger = logger;
    }

    [HttpGet("get-students")]
    public IActionResult GetStudents()
    {
        _logger.LogInformation("Запрошен список студентов, всего записей: {StudentCount}", _pepoples.Length);
        return new JsonResult(_pepoples);
    }

    [HttpGet("get-student/{nickName}")]
    public IActionResult GetStudent(string nickName)
    {
        // Свойство SearchNickName добавится ко ВСЕМ событиям внутри блока
        using (_logger.BeginScope(new Dictionary<string, object> { ["SearchNickName"] = nickName }))
        {
            _logger.LogDebug("Поиск студента по нику {NickName}", nickName);

            var student = _pepoples.FirstOrDefault(p =>
                p.NickName.Equals(nickName, StringComparison.OrdinalIgnoreCase));

            if (student is null)
            {
                _logger.LogWarning("Студент с ником {NickName} не найден", nickName);
                return NotFound();
            }

            _logger.LogInformation("Найден студент {@Student}", student);
            return Ok(student);
        }
    }

    // Обработанная ошибка: приложение само ловит исключение и логирует его
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

    // Необработанная ошибка: её поймает и залогирует фильтр из шага 5
    [HttpGet("crash")]
    public IActionResult Crash()
    {
        throw new ApplicationException("Необработанное исключение");
    }
}
```

> **Почему это делается.**
> - **`ILogger<StudentController>` через конструктор (DI).** Контроллер не создаёт логгер сам и не зависит от Serilog — только от абстракции .NET. Параметр типа `<StudentController>` задаёт `SourceContext = MyCollegePlus.Controllers.StudentController`, по которому в Seq фильтруются логи конкретного класса, а в `appsettings.json` настраивается уровень.
> - **Автоматический лог не заменяет ручной.** Фильтр знает только «что вызвали и чем закончилось». Только разработчик знает *бизнес-смысл*: «студент не найден», «найдено N записей», «ошибка на таком-то этапе».
> - **Шаблоны вместо интерполяции.** Пишите `LogWarning("Студент с ником {NickName} не найден", nickName)`, а **не** `LogWarning($"Студент с ником {nickName} не найден")`. В первом случае `NickName` — отдельное свойство, по которому можно искать, а все такие события имеют один шаблон (`@MessageTemplate`) и группируются. Во втором — каждое сообщение уникальная строка, структура теряется, а строка формируется даже тогда, когда уровень отключён.
> - **Выбор уровня осознанный:** `Debug` — подробности для отладки (в продакшене обычно выключено); `Information` — штатные бизнес-события; `Warning` — нештатная, но ожидаемая ситуация (404); `Error` — сбой операции.
> - **Исключение — первым параметром** `LogError(ex, ...)`. Тогда Seq сохранит тип, сообщение и стек-трейс в отдельном поле `@Exception`, а не просто текст.
> - **`BeginScope`** — свойство области (`SearchNickName`) автоматически прикрепляется ко всем событиям внутри `using`, в том числе к событиям из вызываемых сервисов. Это удобно для корреляции событий одной операции.
> - **Два вида ошибок** (`fail` и `crash`) показывают разницу: обработанную ошибку логирует сам разработчик, необработанную — автоматический фильтр и middleware.

---

### Шаг 7. Локальный запуск и проверка в Seq

Обновите `MyCollegePlus.http` (удалите запрос к несуществующему `/weatherforecast/`):

```http
@MyCollegePlus_HostAddress = http://localhost:5251

GET {{MyCollegePlus_HostAddress}}/api/Student/get-students
###
GET {{MyCollegePlus_HostAddress}}/api/Student/get-student/W4rl0ck
###
GET {{MyCollegePlus_HostAddress}}/api/Student/get-student/nobody
###
GET {{MyCollegePlus_HostAddress}}/api/Student/fail
###
GET {{MyCollegePlus_HostAddress}}/api/Student/crash
###
```

Запустите приложение:

```bash
dotnet run --launch-profile http
```

Выполните все запросы из `.http`-файла (в Rider / VS / VS Code с расширением REST Client) или через `curl`:

```bash
curl http://localhost:5251/api/Student/get-student/W4rl0ck
```

Откройте `http://seq.localhost` и убедитесь, что события пришли.

> **Почему это делается.** Сначала проверяем работу на машине разработчика: здесь проще отлаживать (точки останова, консоль). При локальном запуске приложение находится **вне** сети Docker, поэтому обращается к Seq через опубликованный порт `localhost:5341` — именно он указан в `appsettings.json`. Запросы подобраны так, чтобы в логах были все уровни: `Information`, `Debug`, `Warning`, `Error`.

**Если событий нет:** проверьте `docker compose ps` (Seq запущен?), адрес `serverUrl`, подождите 2–5 секунд (события отправляются пакетами). Ошибки самого sink'а можно увидеть, добавив в начало `Program.cs` строку `Serilog.Debugging.SelfLog.Enable(Console.Error);`.

---

### Шаг 8. Подключение контейнера backend к Seq

Отредактируйте сервис `backend` в `docker-compose.yml`:

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
        # Переопределяем адрес Seq из appsettings.json (элемент WriteTo с индексом 1)
        Serilog__WriteTo__1__Args__serverUrl: http://seq:5341
        Serilog__Properties__Environment: Docker
      networks:
        - mycollegeplus-net
```

Пересоберите и запустите:

```bash
docker compose up -d --build backend
```

```bash
curl http://localhost:8082/api/Student/get-student/R0l4nd
```

> **Почему это делается.**
> - **`localhost` внутри контейнера — это сам контейнер**, а не ваш компьютер. Поэтому адрес `http://localhost:5341` из `appsettings.json` в Docker не работает. Внутри общей сети `mycollegeplus-net` Seq доступен по имени сервиса: `http://seq:5341` (Docker DNS).
> - **Переменные окружения переопределяют `appsettings.json`.** Двойное подчёркивание `__` заменяет разделитель `:` в иерархии настроек, а `1` — индекс элемента массива `WriteTo` (0 — Console, 1 — Seq). Один и тот же образ работает в разных окружениях без пересборки — это принцип 12-factor app «конфигурация в окружении».
> - **`Environment: Docker`** — дополнительное свойство, по которому в Seq можно отличить события из контейнера от событий локального запуска.
> - **`depends_on: seq`** — Compose стартует Seq раньше backend, чтобы первые события запуска не терялись.

Проверьте:

```bash
docker compose ps
```

Контейнер `backend-college` должен перейти в состояние `healthy` (благодаря endpoint `/health`), а в Seq появятся события с `Environment = 'Docker'` и `MachineName` = ID контейнера.

---

### Шаг 9. Анализ логов в Seq

Выполните в строке поиска Seq следующие запросы и сделайте скриншоты результатов:

| № | Запрос | Что показывает |
|---|---|---|
| 1 | `@Level = 'Error'` | Все ошибки |
| 2 | `has(@Exception)` | События с исключением и стек-трейсом |
| 3 | `SourceContext like 'Microsoft%'` | Только системные логи фреймворка |
| 4 | `SourceContext = 'MyCollegePlus.Controllers.StudentController'` | Только ручные логи контроллера |
| 5 | `NickName = 'nobody'` | Поиск по свойству из шаблона сообщения |
| 6 | `StatusCode >= 400` | Неуспешные запросы |
| 7 | `Elapsed > 10` | Медленные запросы (дольше 10 мс) |
| 8 | `Environment = 'Docker'` | События только из контейнера |
| 9 | `select count(*) from stream group by RequestPath` | Статистика запросов по адресам |

Также:

1. Откройте любое событие `Конец Student.GetStudent` и найдите свойство `RequestId` (или `@TraceId`). Нажмите на значение → **Find** — Seq покажет **все события этого запроса**: системные, автоматические и ручные.
2. Создайте **Signal** (сохранённый фильтр) «Ошибки MyCollegePlus» с условием `@Level = 'Error' and Application = 'MyCollegePlus'`.

> **Почему это делается.** Цель централизованного логирования — не хранить логи, а **быстро находить ответы**: «что сломалось», «у кого», «как часто», «насколько медленно». Запросы демонстрируют главное преимущество структурных логов: фильтрация по свойствам вместо поиска подстроки. Корреляция по `RequestId`/`TraceId` показывает весь путь одного запроса, когда параллельно идут сотни других.

---

### Шаг 10 (дополнительно). Защита приёма логов API-ключом

1. В Seq: **Settings → API Keys → Add API Key**, задайте название `MyCollegePlus`, минимальный уровень `Debug`.
2. Добавьте ключ в настройки sink'а через переменную окружения (не коммитьте ключ в git!):

```yaml
        Serilog__WriteTo__1__Args__apiKey: ${SEQ_API_KEY}
```

и в `.env`: `SEQ_API_KEY=ваш_ключ`.

> **Почему это делается.** Сейчас любой процесс, имеющий доступ к порту `5341`, может записать в Seq что угодно (подделать логи или «завалить» хранилище). API-ключ идентифицирует приложение, позволяет отозвать доступ и задать минимальный уровень событий на стороне сервера. Секреты хранятся в `.env`/переменных окружения, а не в `appsettings.json` в репозитории.

---

## Индивидуальные задания

Выполните одно задание по номеру в журнале (№ mod 4 + 1):

1. Добавьте `POST api/Student/add` (приём `People` из тела запроса) с логами `Information` об успешном добавлении и `Warning` при попытке добавить дубликат ника. Найдите в Seq все попытки добавления дубликатов.
2. Перепишите ручные логи контроллера с помощью генератора исходного кода `[LoggerMessage]` (partial-методы). Объясните, чем этот способ быстрее.
3. Создайте атрибут `[NoArgumentsLogging]` и доработайте `LogActionFilter`, чтобы для отмеченных им action аргументы не логировались (защита персональных данных).
4. Добавьте middleware, который читает заголовок `X-Correlation-Id` (или генерирует новый GUID), кладёт его в `LogContext.PushProperty("CorrelationId", ...)` и возвращает в ответе. Покажите в Seq поиск по `CorrelationId`.


---

## Контрольные вопросы

1. Чем структурное логирование отличается от текстового? Приведите пример одного и того же события в обоих форматах.
2. Почему в коде приложения используется `ILogger<T>`, а не напрямую классы Serilog? Что изменится в контроллерах, если заменить Seq на Elasticsearch?
3. Почему нельзя писать `_logger.LogInformation($"Студент {nickName}")`? Назовите минимум две проблемы.
4. Что означает символ `@` в `{@Student}`? Что попадёт в лог без него?
5. Перечислите уровни логирования в порядке возрастания важности. Какой уровень вы выберете для: неверного пароля пользователя; недоступности БД; входа в метод с параметрами; успешной регистрации студента?
6. Для чего нужна секция `MinimumLevel:Override`? Почему для `Microsoft.AspNetCore` выставлен `Warning`, а для `Microsoft.Hosting.Lifetime` — `Information`?
7. Зачем создаётся bootstrap-логгер и вызывается `Log.CloseAndFlush()`? Что произойдёт без них при падении приложения на старте?
8. Чем `UseSerilogRequestLogging` отличается от `LogActionFilter`? Почему нельзя получить аргументы action в middleware?
9. Почему важно место подключения `UseSerilogRequestLogging` в конвейере middleware?
10. Почему при запуске в Docker адрес Seq — `http://seq:5341`, а при локальном запуске — `http://localhost:5341`?
11. Как переменная окружения `Serilog__WriteTo__1__Args__serverUrl` сопоставляется с `appsettings.json`? Что будет, если поменять местами элементы массива `WriteTo`?
12. Чем отличаются порты Seq `80` и `5341`? Почему в `docker-compose.yml` порт `5341` опубликован только на `127.0.0.1`?
13. Для чего используется `BeginScope`? Чем свойство области отличается от свойства в шаблоне сообщения?
14. Как в Seq найти все события, относящиеся к одному HTTP-запросу?
15. Почему запросы healthcheck к `/health` понижаются до уровня `Verbose`? Посчитайте, сколько событий в сутки они создавали бы.
16. Какие данные нельзя записывать в логи? Как в данной работе может произойти утечка и как её предотвратить?
17. Зачем Seq API-ключ, если порт и так доступен только с localhost?
18. В чём разница между обработанным (`/fail`) и необработанным (`/crash`) исключением с точки зрения логов? Какие компоненты записали событие в каждом случае?
