Проект реализует и демонстрирует основные паттерны проектирования

## Запуск

Сначала склонируйте проект

```bash
git clone https://github.com/StudentSK0/FinanceTracker
cd FinanceTracker
```

Затем запустите

```bash
cd FinanceTracker
dotnet build
dotnet run --project Finance.App
```

При первом запуске выполняется автоматическая инициализация `finance.db`.


## Описание проекта

Модуль учёта финансов состоит из доменной модели, фасадов предметной логики и инфраструктуры хранения данных: SQLite, Dapper и репозитории, а также консольного пользовательского интерфейса.

| Область     | Возможности |
|-------------|-------------|
| Счета       | Создание, переименование, удаление, пересчёт баланса, просмотр |
| Категории   | Создание, переименование, удаление, просмотр |
| Операции    | Создание, редактирование, удаление с автоматической корректировкой баланса |
| Аналитика   | Расчёт доходов/расходов за период, группировка по категориям |
| Экспорт     | JSON и CSV (Visitor) |
| Импорт      | JSON, YAML, CSV (Template Method) |
| UI          | Меню на паттерне «Команда» + измерение времени выполнения|

## Принципы SOLID

| Принцип | Где реализован | Польза |
|--------|---------------|----------------|--------------|
| S | Доменные сущности | Тестируемость и простота |
| O | Импорт/экспорт через наследование | Расширяемость |
| L | IExportVisitor / ImporterBase | Ослабление связности |
| I | Разделённые репозитории | Минимизация зависимостей |
| D | Фасады зависят от интерфейсов | Модульность |



### S — Single Responsibility

Каждая доменная сущность отвечает только за свою собственную предметную роль и не содержит логики,
относящейся к хранению данных, пользовательскому интерфейс или аналитике.

**Пример — класс `BankAccount`:**  
Файл: `Finance.Domain/Entities/BankAccount.cs`

```csharp
public sealed class BankAccount
{
    public Guid Id { get; }
    public string Name { get; private set; }
    public decimal Balance { get; private set; }

    public void Rename(string newName) => Name = newName;

    public void Apply(Operation op)
        => Balance += op.Type == OpType.Income ? op.Amount : -op.Amount;

    public void Revert(Operation op)
        => Balance -= op.Type == OpType.Income ? op.Amount : -op.Amount;

    public void SetBalance(decimal value) => Balance = value;
}
```
`BankAccount` хранит состояние счёта и знает правила его изменения. Он *не* зависит от БД, UI или сервисов экспорта. Это упрощает тестирование и делает модель устойчивой к изменениям.



### O — Open/Closed. Открыт для расширения, закрыт для изменения

При добавлении нового формата импорта или экспорта существующий код изменять не нужно.

**Пример — класс `ImporterBase`:**  
Файл: `Finance.Infrastructure/Import/ImporterBase.cs`

```csharp
public abstract class ImporterBase : IImporter
{
    public void Import(string path)
    {
        var dto = Load(path); // точка расширения
        Upsert(dto);          // общая логика
    }

    protected abstract RootDto Load(string path);
}
```

Добавление нового формата:

```csharp
public sealed class ExcelImporter : ImporterBase
{
    protected override RootDto Load(string path) { ... }
}
```

Таким образом система развивается через наследование, не нарушая существующий код.


### L — Liskov Substitution

Все импортеры реализуют одинаковый контракт, поэтому их можно взаимозаменять:

```csharp
_import.Import(_json, path);
_import.Import(_yaml, path);
_import.Import(_csv, path);
```

Замена одного импорта на другой не нарушает работу системы — это и есть соблюдение принципа LSP.


### I — Interface Segregation

Вместо одного большого репозитория используются специализированные:

```csharp
public interface IAccountRepo
{
    void Add(BankAccount acc);
    BankAccount Get(Guid id);
    IEnumerable<BankAccount> List();
    void Update(BankAccount acc);
    void Remove(Guid id);
}
```

```csharp
public interface ICategoryRepo
{
    void Add(Category cat);
    Category Get(Guid id);
    IEnumerable<Category> List();
    void Rename(Guid id, string newName);
    void Remove(Guid id);
}
```

```csharp
public interface IOperationRepo
{
    Operation Get(Guid id);
    IEnumerable<Operation> List();
    IEnumerable<Operation> ListByAccount(Guid accountId);
    void AddAndApplyBalance(Operation op, BankAccount account);
    void RemoveAndRevertBalance(Operation op, BankAccount account);
}
```

Каждый интерфейс описывает только одну предметную роль, код чище и проще тестировать.


### D — Dependency Inversion

Высокоуровневые компоненты зависят от абстракций, а не от конкретных реализаций.

```csharp
services.AddSingleton<IAccountRepo>(sp =>
    new CachedAccountRepo(
        new DbAccountRepo(sp.GetRequiredService<DbConnectionFactory>())
    ));
```

Фасады получают интерфейсы:

```csharp
public OperationFacade(IOperationRepo ops, IAccountRepo accounts) { ... }
```

## принципы GRASP

| Принцип | Где реализован | Польза |
|--------|------------|--------|
| **Information Expert** | `BankAccount.Apply()` / `BankAccount.Revert()` | Логика изменения баланса находится у владельца данных |
| **Controller** | Фасады предметной логики | UI остаётся «тонким» и не содержит бизнес-правил |
| **Low Coupling** | Доменный слой не зависит от БД | Слои системы независимы, хранилище можно заменить |
| **High Cohesion** | Каждая сущность выполняет строго одну задачу | Упрощает понимание и сопровождение кода |


### Controller

Каждый пользовательский сценарий оформлен как отдельная команда, реализующая `ICommand`.
Команда получает ввод, вызывает фасады и не взаимодействует с БД напрямую:

```csharp
public sealed class AddOperationCommand : ICommand
{
    private readonly AccountFacade  _accounts;
    private readonly OperationFacade _ops;

    public void Execute()
    {
        var acc = ConsoleSelector.Select(
            _accounts.List().ToList(),
            a => $"{a.Name} ({a.Balance}₽)"
        );

        Console.Write("Тип (income/expense): ");
        var t = Console.ReadLine()!.Trim().ToLower() == "income"
            ? OpType.Income : OpType.Expense;

        Console.Write("Сумма: ");
        var amount = decimal.Parse(Console.ReadLine()!);

        _ops.Create(t, acc.Id, amount, DateOnly.FromDateTime(DateTime.Today));
    }
}
```

Команда описывает *сценарий*, а не доменную логику. UI остаётся нечувствительным к изменениям модели.


### Low Coupling

Фасады не зависят от конкретного хранилища — только от интерфейсов:

```csharp
public sealed class OperationFacade
{
    private readonly IOperationRepo _ops;
    private readonly IAccountRepo   _accounts;

    public OperationFacade(IOperationRepo ops, IAccountRepo accounts)
    {
        _ops      = ops;
        _accounts = accounts;
    }
}
```

Конкретные реализации (`DbOperationRepo`, `CachedAccountRepo`, SQLite, Dapper...) подключаются в `Program.cs`.  
Это позволяет заменить хранилище без изменения доменной логики.


### High Cohesion

Каждый фасад обслуживает одну сущность и не смешивает обязанности:

```csharp
public sealed class AccountFacade
{
    public BankAccount Create(string name, decimal openingBalance = 0m) { … }
    public void Rename(Guid id, string newName) { … }
    public void Delete(Guid id) { … }
    public decimal RecalcBalance(Guid accountId) { … }
    public IEnumerable<BankAccount> List() { … }
}
```

Фасад не содержит логики категорий, аналитики или операций, область ответственности остаётся чёткой.


### Information Expert

Изменение баланса выполняет владелец данных — `BankAccount`:

```csharp
public void Apply(Operation op)
{
    Balance += op.Type == OpType.Income ? op.Amount : -op.Amount;
}

public void Revert(Operation op)
{
    Balance -= op.Type == OpType.Income ? op.Amount : -op.Amount;
}
```

Логика находится рядом с данными → код проще, корректнее и устойчивее.


### Indirection. Промежуточное звено

Взаимодействие UI и БД проходит через фасады:

```
UI => Команда => Фасад => Репозиторий => БД
```

Это защищает UI и команды от изменений инфраструктуры.


### Pure Fabrication

Кэширующие репозитории (`CachedAccountRepo`, `CachedCategoryRepo`) не являются частью предметной модели.  
Они созданы для оптимизации производительности, не нарушая бизнес-логику.
## Паттерны GoF

| Паттерн | Где реализован | Польза |
|--------|------------|------------|
| **Facade** | Фасады доменной логики | Сокрытие сложных сценариев и упрощение интерфейса |
| **Command** | `Finance.App/Commands/*` | Каждое действие пользователя оформлено как объект команды |
| **Decorator** | `TimedCommand` | Добавление измерения времени без изменения исходных команд |
| **Template Method** | `ImporterBase` | Повторное использование общего алгоритма импорта с переопределяемыми шагами |
| **Visitor** | `IExportVisitor` + экспортёры | Добавление новых форматов экспорта без модификации доменных сущностей |
| **Factory** | `DomainFactory` | Создание корректных и согласованных сущностей доменной модели |
| **Proxy (Cache)** | `CachedCategoryRepo`, `CachedAccountRepo` | Ускорение доступа к данным за счёт кэширования |


### Фасад

Пользователь не работает напрямую с доменными сущностями.  
Для взаимодействия используются фасады предметной логики:

- `AccountFacade` — создание, переименование, пересчёт и получение списка счетов;
- `CategoryFacade` — управление категориями;
- `OperationFacade` — создание, обновление и удаление операций с автоматической корректировкой баланса;
- `AnalyticsFacade` — вычисление доходов, расходов и группировок;
- `ExportFacade` — подготовка данных к экспорту;
- `ImportFacade` — загрузка данных из внешних источников.

Фасады инкапсулируют использование репозиториев и скрывают внутреннюю структуру домена.  
Это делает систему устойчивой к изменению способа хранения и форматов данных.


### Команда

Каждое действие в пользовательском интерфейсе оформлено как отдельная команда (`Finance.App.Commands`). Например:

- `CreateAccountCommand`
- `AddOperationCommand`
- `EditOperationCommand`
- `ExportCommand`
- `ImportCommand`
- `GroupByCategoryCommand`

Команда определяет **что** должно быть сделано, но не **как**. Это упрощает тестирование, позволяет добавлять новые действия без изменения `Program.cs`.


### Декоратор

Для измерения времени выполнения используется `TimedCommand`:

```csharp
public sealed class TimedCommand : ICommand
{
    private readonly ICommand _inner;
    public string Name => _inner.Name;

    public void Execute()
    {
        var sw = Stopwatch.StartNew();
        _inner.Execute();
        sw.Stop();
        Console.WriteLine($"(Выполнено за {sw.ElapsedMilliseconds} мс)");
    }
}
```

Поведение команды расширяется *без изменения её исходного кода*.


### Шаблонный метод

Общий алгоритм импорта описан в `ImporterBase`:

```
Import()
 ├─ Load()    // определяется в наследниках (JSON/YAML/CSV)
 └─ Upsert()  // общая логика обновления данных
```

Это обеспечивает расширяемость и позволяет избежать дублирования логики.


### Посетитель

Экспорт данных реализован через интерфейс `IExportVisitor`:

```csharp
public interface IExportVisitor
{
    void Visit(
        IEnumerable<BankAccount> acc,
        IEnumerable<Category>   cat,
        IEnumerable<Operation>  ops);
}
```

Добавление нового формата экспорта (XML, Excel, HTML) не требует изменения домена.


### Прокси с кэшированием

Используются кэширующие репозитории:

- `CachedAccountRepo`
- `CachedCategoryRepo`

Кэш ускоряет повторные запросы и не изменяет доменную модель.

```md
UI => Команда => Фасад => Репозиторий => БД
```

Интерфейсы хранилища не меняются → остальные части системы остаются неизменными.
