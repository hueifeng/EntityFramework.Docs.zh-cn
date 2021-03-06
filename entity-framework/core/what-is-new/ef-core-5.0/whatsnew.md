---
title: EF Core 5.0 中的新增功能
description: EF Core 5.0 中的新功能概述
author: ajcvickers
ms.date: 07/20/2020
uid: core/what-is-new/ef-core-5.0/whatsnew
ms.openlocfilehash: 3a1f5c7d44ad0e4d648492c4edcf14678c73538e
ms.sourcegitcommit: 6f7af3f138bf7c724cbdda261f97e5cf7035e8d7
ms.translationtype: HT
ms.contentlocale: zh-CN
ms.lasthandoff: 08/25/2020
ms.locfileid: "88847587"
---
# <a name="whats-new-in-ef-core-50"></a>EF Core 5.0 中的新增功能

EF Core 5.0 目前正在开发中。 此页面将包含每个预览版中引入的令人关注的更改的简要介绍。

此页面不会复制 [EF Core 5.0](xref:core/what-is-new/ef-core-5.0/plan) 的计划。 计划中介绍了 EF Core 5.0 的整体主题，其中包括我们在交付最终版本之前打算包含的所有内容。

发布时，我们会将此处的链接添加到官方文档。

## <a name="preview-8"></a>预览版 8

## <a name="table-per-type-tpt-mapping"></a>每个类型一张表 (TPT) 映射

默认情况下，EF Core 会将 .NET 类型的继承层次结构映射到单个数据库表。 这称为每个层次结构一张表 (TPH) 映射。 EF Core 5.0 还允许将继承层次结构中的每个 .NET 类型映射到另一个数据库表，这称为每个类型一张表 (TPT) 映射。

例如，请考虑具有映射的层次结构的此模型：

```c#
public class Animal
{
    public int Id { get; set; }
    public string Species { get; set; }
}

public class Pet : Animal
{
    public string Name { get; set; }
}

public class Cat : Pet
{
    public string EdcuationLevel { get; set; }
}

public class Dog : Pet
{
    public string FavoriteToy { get; set; }
}
```

默认情况下，EF Core 会将此模型映射到单个表：

```sql
CREATE TABLE [Animals] (
    [Id] int NOT NULL IDENTITY,
    [Species] nvarchar(max) NULL,
    [Discriminator] nvarchar(max) NOT NULL,
    [Name] nvarchar(max) NULL,
    [EdcuationLevel] nvarchar(max) NULL,
    [FavoriteToy] nvarchar(max) NULL,
    CONSTRAINT [PK_Animals] PRIMARY KEY ([Id])
);
```

但是，将每个实体类型映射到不同的表，会得到每个类型一张表的结果：

```sql
CREATE TABLE [Animals] (
    [Id] int NOT NULL IDENTITY,
    [Species] nvarchar(max) NULL,
    CONSTRAINT [PK_Animals] PRIMARY KEY ([Id])
);

CREATE TABLE [Pets] (
    [Id] int NOT NULL,
    [Name] nvarchar(max) NULL,
    CONSTRAINT [PK_Pets] PRIMARY KEY ([Id]),
    CONSTRAINT [FK_Pets_Animals_Id] FOREIGN KEY ([Id]) REFERENCES [Animals] ([Id]) ON DELETE NO ACTION
);

CREATE TABLE [Cats] (
    [Id] int NOT NULL,
    [EdcuationLevel] nvarchar(max) NULL,
    CONSTRAINT [PK_Cats] PRIMARY KEY ([Id]),
    CONSTRAINT [FK_Cats_Animals_Id] FOREIGN KEY ([Id]) REFERENCES [Animals] ([Id]) ON DELETE NO ACTION,
    CONSTRAINT [FK_Cats_Pets_Id] FOREIGN KEY ([Id]) REFERENCES [Pets] ([Id]) ON DELETE NO ACTION
);

CREATE TABLE [Dogs] (
    [Id] int NOT NULL,
    [FavoriteToy] nvarchar(max) NULL,
    CONSTRAINT [PK_Dogs] PRIMARY KEY ([Id]),
    CONSTRAINT [FK_Dogs_Animals_Id] FOREIGN KEY ([Id]) REFERENCES [Animals] ([Id]) ON DELETE NO ACTION,
    CONSTRAINT [FK_Dogs_Pets_Id] FOREIGN KEY ([Id]) REFERENCES [Pets] ([Id]) ON DELETE NO ACTION
);
```

请注意，上述外键约束的创建是在对预览版 8 的代码创建分支后添加的。

可使用映射特性将实体类型映射到不同的表：

```c#
[Table("Animals")]
public class Animal
{
    public int Id { get; set; }
    public string Species { get; set; }
}

[Table("Pets")]
public class Pet : Animal
{
    public string Name { get; set; }
}

[Table("Cats")]
public class Cat : Pet
{
    public string EdcuationLevel { get; set; }
}

[Table("Dogs")]
public class Dog : Pet
{
    public string FavoriteToy { get; set; }
}
```

或使用 `ModelBuilder` 配置：

```c#
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Animal>().ToTable("Animals");
    modelBuilder.Entity<Pet>().ToTable("Pets");
    modelBuilder.Entity<Cat>().ToTable("Cats");
    modelBuilder.Entity<Dog>().ToTable("Dogs");
}
```

记录信息可通过问题 [#1979](https://github.com/dotnet/EntityFramework.Docs/issues/1979) 进行跟踪。

### <a name="migrations-rebuild-sqlite-tables"></a>迁移：重新生成 SQLite 表

与其他数据库相比，SQLite 的架构操作功能相对有限。 例如，从现有表中删除列需要删除并重新创建整个表。 EF Core 5.0 迁移现在支持自动重新生成表，以便实现所需的架构更改。

例如，假设为 `Unicorn` 实体类型创建了 `Unicorns` 表：

```c#
public class Unicorn
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int Age { get; set; }
}
```

```sql
CREATE TABLE "Unicorns" (
    "Id" INTEGER NOT NULL CONSTRAINT "PK_Unicorns" PRIMARY KEY AUTOINCREMENT,
    "Name" TEXT NULL,
    "Age" INTEGER NOT NULL
);
```

然后我们了解到，存储独角兽的年龄被认为非常不礼貌，因此让我们删除该属性，添加新的迁移，并更新数据库。 使用 EF Core 3.1 时，此更新将失败，因为无法删除该列。 在 EF Core 5.0 中，迁移将改为重新生成表：

```sql
CREATE TABLE "ef_temp_Unicorns" (
    "Id" INTEGER NOT NULL CONSTRAINT "PK_Unicorns" PRIMARY KEY AUTOINCREMENT,
    "Name" TEXT NULL
);

INSERT INTO "ef_temp_Unicorns" ("Id", "Name")
SELECT "Id", "Name"
FROM Unicorns;

PRAGMA foreign_keys = 0;

DROP TABLE "Unicorns";

ALTER TABLE "ef_temp_Unicorns" RENAME TO "Unicorns";

PRAGMA foreign_keys = 1;
```

请注意：
* 创建一个临时表，其中包含新表所需的架构
* 将数据从当前表复制到临时表中
* 关闭外键强制执行
* 删除当前表
* 将临时表重命名为新表

记录信息可通过问题 [#2583](https://github.com/dotnet/EntityFramework.Docs/issues/2583) 进行跟踪。

### <a name="table-valued-functions"></a>表值函数

此功能是 [@pmiddleton](https://github.com/pmiddleton) 在社区中贡献的。 非常感谢贡献此功能！

EF Core 5.0 为 .NET 方法到表值函数 (TVF) 的映射提供一流支持。 然后可在 LINQ 查询中使用这些函数，在此类查询中，函数结果上的其他组合也将转换为 SQL。

例如，请考虑在 SQL Server 数据库中定义的以下 TVF：

```sql
CREATE FUNCTION GetReports(@employeeId int)
RETURNS @reports TABLE
(
    Name nvarchar(50) NOT NULL,
    IsDeveloper bit NOT NULL
)
AS
BEGIN
    WITH cteEmployees AS
    (
        SELECT Id, Name, ManagerId, IsDeveloper
        FROM Employees
        WHERE Id = @employeeId
        UNION ALL
        SELECT e.Id, e.Name, e.ManagerId, e.IsDeveloper
        FROM Employees e
        INNER JOIN cteEmployees cteEmp ON cteEmp.Id = e.ManagerId
    )
    INSERT INTO @reports
    SELECT Name, IsDeveloper
    FROM cteEmployees
    WHERE Id != @employeeId

    RETURN
END
```

EF Core 模型需要两种实体类型才能使用此 TVF：
* 以正常方式映射到 Employees 表的 `Employee` 类型
* 与 TVF 返回的形状相匹配的 `Report` 类型

```c#
public class Employee
{
    public int Id { get; set; }
    public string Name { get; set; }
    public bool IsDeveloper { get; set; }

    public int? ManagerId { get; set; }
    public virtual Employee Manager { get; set; }
}
```

```c#
public class Report
{
    public string Name { get; set; }
    public bool IsDeveloper { get; set; }
}
```

这些类型必须包含在 EF Core 模型中：

```c#
modelBuilder.Entity<Employee>();
modelBuilder.Entity(typeof(Report)).HasNoKey();
```

请注意，`Report` 没有主键，因此必须这样配置。

最后，必须将 .NET 方法映射到数据库中的 TVF。 可以使用新的 `FromExpression` 方法在 DbContext 上定义此方法：

```c#
public IQueryable<Report> GetReports(int managerId)
    => FromExpression(() => GetReports(managerId));
```

此方法使用与上面定义的 TVF 匹配的参数和返回类型。 然后在 OnModelCreating 中将该方法添加到 EF Core 模型：

```c#
modelBuilder.HasDbFunction(() => GetReports(default));
```

（在此处使用 lambda 可轻松将 `MethodInfo` 传递到 EF Core。 将忽略传递给该方法的参数。）

现在可以编写查询，以调用 `GetReports` 并对结果进行组合。 例如：

```c#
from e in context.Employees
from rc in context.GetReports(e.Id)
where rc.IsDeveloper == true
select new
{
  ManagerName = e.Name,
  EmployeeName = rc.Name,
})
```

在 SQL Server 上，这将转换为：

```sql
SELECT [e].[Name] AS [ManagerName], [g].[Name] AS [EmployeeName]
FROM [Employees] AS [e]
CROSS APPLY [dbo].[GetReports]([e].[Id]) AS [g]
WHERE [g].[IsDeveloper] = CAST(1 AS bit)
```

请注意，SQL 以 `Employees` 表为根，调用 `GetReports`，然后在函数的结果上添加一个额外的 WHERE 子句。

### <a name="flexible-queryupdate-mapping"></a>灵活查询/更新映射

EF Core 5.0 允许将同一实体类型映射到不同的数据库对象。 这些对象可以是表、视图或函数。

例如，可将一个实体类型同时映射到数据库视图和数据库表：

```c#
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder
        .Entity<Blog>()
        .ToTable("Blogs")
        .ToView("BlogsView");
}
```

默认情况下，EF Core 随后将从视图进行查询，将更新发送到表。 例如，执行以下代码：

```c#
var blog = context.Set<Blog>().Single(e => e.Name == "One Unicorn");

blog.Name = "1unicorn2";

context.SaveChanges();
```

生成针对视图的查询，然后对表进行更新：

```sql
SELECT TOP(2) [b].[Id], [b].[Name], [b].[Url]
FROM [BlogsView] AS [b]
WHERE [b].[Name] = N'One Unicorn'

SET NOCOUNT ON;
UPDATE [Blogs] SET [Name] = @p0
WHERE [Id] = @p1;
SELECT @@ROWCOUNT;
```

### <a name="context-wide-split-query-configuration"></a>上下文范围的拆分查询配置

现在可以将拆分查询（见下文）配置为 DbContext 执行的任何查询的默认值。 此配置仅适用于关系提供程序，因此必须将其指定为 `UseProvider` 配置的一部分。 例如：

```c#
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    => optionsBuilder
        .UseSqlServer(
            Your.SqlServerConnectionString,
            b => b.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery));
```

记录信息可通过问题 [#2407](https://github.com/dotnet/EntityFramework.Docs/issues/2407) 进行跟踪。

### <a name="physicaladdress-mapping"></a>PhysicalAddress 映射

此功能是 [@ralmsdeveloper](https://github.com/ralmsdeveloper) 在社区中贡献的。 非常感谢贡献此功能！

标准 .NET [PhysicalAddress 类](/dotnet/api/system.net.networkinformation.physicaladdress)现自动映射到尚不具备原生支持的数据库的字符串列中。 有关详细信息，请参阅下面的 `IPAddress` 示例。

## <a name="preview-7"></a>预览版 7

### <a name="dbcontextfactory"></a>DbContextFactory

EF Core 5.0 引入了 `AddDbContextFactory` 和 `AddPooledDbContextFactory`，可用于在应用程序的依赖关系注入 (D.I.) 容器中注册工厂来创建 DbContext 实例。 例如：

```csharp
services.AddDbContextFactory<SomeDbContext>(b =>
    b.UseSqlServer(@"Server=(localdb)\mssqllocaldb;Database=Test"));
```

应用程序服务（例如 ASP.NET Core 控制器）之后可依赖于服务构造函数中的 `IDbContextFactory<TContext>`。 例如：

```csharp
public class MyController
{
    private readonly IDbContextFactory<SomeDbContext> _contextFactory;

    public MyController(IDbContextFactory<SomeDbContext> contextFactory)
    {
        _contextFactory = contextFactory;
    }
}
```

之后可根据需要创建和使用 DbContext 实例。 例如：

```csharp
public void DoSomeThing()
{
    using (var context = _contextFactory.CreateDbContext())
    {
        // ...
    }
}
```

请注意，以这种方式创建的 DbContext 实例并非由应用程序的服务提供程序进行管理，因此必须由应用程序释放。 此类分离对于 Blazor 应用程序非常有用（此情况下建议使用 `IDbContextFactory`），但在其他方案中也很有用。

你可通过调用 `AddPooledDbContextFactory` 来共用 DbContext 实例。 此类共用的原理与 `AddDbContextPool` 相同，且限制也相同。

记录信息可通过问题 [#2523](https://github.com/dotnet/EntityFramework.Docs/issues/2523) 进行跟踪。

### <a name="reset-dbcontext-state"></a>重置 DbContext 状态

EF Core 5.0 引入了 `ChangeTracker.Clear()`，它可清除所有被跟踪实体的 DbContext。 当使用为每个工作单元创建生存期较短的新上下文实例的最佳做法时，通常不需要这样做。 但如果需要重置 DbContext 实例的状态，则使用新的 `Clear()` 方法比大量分离所有实体的性能和可靠性更强。

记录信息可通过问题 [#2524](https://github.com/dotnet/EntityFramework.Docs/issues/2524) 进行跟踪。

### <a name="new-pattern-for-store-generated-defaults"></a>存储生成的默认值的新模式

EF Core 允许为可能还有默认值约束的列设置显式值。 EF Core 使用 type 属性类型的 CLR 默认值作为此项的 sentinel；如果该值不是 CLR 默认值，则会将其插入，否则将使用数据库默认值。

这会使 CLR 默认值不是良好的 sentinel（最值得注意的是 `bool` 属性）的类型出现问题。 EF Core 5.0 现允许支持字段在此类情况下可为 null。 例如：

```csharp
public class Blog
{
    private bool? _isValid;

    public bool IsValid
    {
        get => _isValid ?? false;
        set => _isValid = value;
    }
}
```

请注意，支持字段可为 null，但公开属性不可为 null。 这允许 sentinel 值为 `null`，而不影响实体类型的公开内容。 在本例中，如果绝不设置 `IsValid`，则将使用数据库默认值，因为支持字段仍为 null。 如果设置 `true` 或 `false`，则此值将显式保存到数据库中。

记录信息可通过问题 [#2525](https://github.com/dotnet/EntityFramework.Docs/issues/2525) 进行跟踪。

### <a name="cosmos-partition-keys"></a>Cosmos 分区键

EF Core 允许将 Cosmos 分区键包含在 EF 模型中。 例如：

```csharp
modelBuilder.Entity<Customer>().HasPartitionKey(b => b.AlternateKey)
```

从预览版 7 开始，分区键将包含在实体类型的 PK 中，用于改进某些查询的性能。

记录信息可通过问题 [#2471](https://github.com/dotnet/EntityFramework.Docs/issues/2471) 进行跟踪。

### <a name="cosmos-configuration"></a>Cosmos 配置

EF Core 5.0 改进了 Cosmos 和 Cosmos 连接的配置。

之前，EF Core 需要在连接到 Cosmos 数据库时显式指定终结点和密钥。 而 EF Core 5.0 允许使用连接字符串。 此外，EF Core 5.0 还允许显式设置 WebProxy 实例。 例如：

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    => optionsBuilder
        .UseCosmos("my-cosmos-connection-string", "MyDb",
            cosmosOptionsBuilder =>
            {
                cosmosOptionsBuilder.WebProxy(myProxyInstance);
            });
```

现在还可配置许多其他超时值、限制等。 例如：

```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    => optionsBuilder
        .UseCosmos("my-cosmos-connection-string", "MyDb",
            cosmosOptionsBuilder =>
            {
                cosmosOptionsBuilder.LimitToEndpoint();
                cosmosOptionsBuilder.RequestTimeout(requestTimeout);
                cosmosOptionsBuilder.OpenTcpConnectionTimeout(timeout);
                cosmosOptionsBuilder.IdleTcpConnectionTimeout(timeout);
                cosmosOptionsBuilder.GatewayModeMaxConnectionLimit(connectionLimit);
                cosmosOptionsBuilder.MaxTcpConnectionsPerEndpoint(connectionLimit);
                cosmosOptionsBuilder.MaxRequestsPerTcpConnection(requestLimit);
            });
```

最后，默认的连接模式现为 `ConnectionMode.Gateway`，兼容性通常更好。

记录信息可通过问题 [#2471](https://github.com/dotnet/EntityFramework.Docs/issues/2471) 进行跟踪。

### <a name="scaffold-dbcontext-now-singularizes"></a>Scaffold-DbContext 现为单数形式

此前，当从现有数据库搭建 DbContext 时，EF Core 将创建与数据库中的表名称匹配的实体类型名称。 例如，表 `People` 和 `Addresses` 对应的实体类型名称为 `People` 和 `Addresses`。

在之前的版本中，此行为是通过注册复数化服务来配置的。 现在，在 EF Core 5.0 中，[Humanizer](https://www.nuget.org/packages/Humanizer.Core/) 包用作默认复数化服务。 这意味着 `People` 和 `Addresses` 现将被反向工程处理为名叫 `Person` 和 `Address` 的实体类型。

### <a name="savepoints"></a>保存点

EF Core 现支持[保存点](/SQL/t-sql/language-elements/save-transaction-transact-sql?view=sql-server-ver15#remarks)，它们可更好地控制执行多个操作的事务。

你可手动创建、发布和回滚保存点。 例如：

```csharp
context.Database.CreateSavepoint("MySavePoint");
```

此外，在执行 `SaveChanges` 失败时，EF Core 现在会回滚到上一个保存点。 因此，可重试 SaveChanges，而不用重试整个事务。

记录信息可通过问题 [#2429](https://github.com/dotnet/EntityFramework.Docs/issues/2429) 进行跟踪。

## <a name="preview-6"></a>预览版 6

### <a name="split-queries-for-related-collections"></a>适合相关集合的拆分查询

从 EF Core 3.0 开始，EF Core 始终会为每个 LINQ 查询生成一个 SQL 查询。 这可确保在使用中的事务模式的约束内返回的数据保持一致。 但是，当查询使用 `Include` 或投影来返回多个相关集合时，速度可能会变得非常慢。

EF Core 5.0 现允许将包含相关集合的一个 LINQ 查询拆分成多个 SQL 查询。 这可能明显提升性能，但如果在两次查询之间数据发生了变化，则可能导致返回的结果不一致。 你能使用可序列化的事务或快照事务来缓解这种情况并通过拆分查询实现一致性，但这可能会带来其他性能成本并导致行为差异。

#### <a name="split-queries-with-include"></a>用 Include 拆分查询

例如，请考虑使用 `Include` 提取两个级别的相关集合的查询：

```CSharp
var artists = context.Artists
    .Include(e => e.Albums).ThenInclude(e => e.Tags)
    .ToList();
```

默认情况下，EF Core 将使用 SQLite 提供程序生成以下 SQL：

```sql
SELECT "a"."Id", "a"."Name", "t0"."Id", "t0"."ArtistId", "t0"."Title", "t0"."Id0", "t0"."AlbumId", "t0"."Name"
FROM "Artists" AS "a"
LEFT JOIN (
    SELECT "a0"."Id", "a0"."ArtistId", "a0"."Title", "t"."Id" AS "Id0", "t"."AlbumId", "t"."Name"
    FROM "Album" AS "a0"
    LEFT JOIN "Tag" AS "t" ON "a0"."Id" = "t"."AlbumId"
) AS "t0" ON "a"."Id" = "t0"."ArtistId"
ORDER BY "a"."Id", "t0"."Id", "t0"."Id0"
```

可使用新的 `AsSplitQuery` API 来更改此行为。 例如：

```CSharp
var artists = context.Artists
    .AsSplitQuery()
    .Include(e => e.Albums).ThenInclude(e => e.Tags)
    .ToList();
```

AsSplitQuery 适用于所有关系数据库提供程序，且如同 AsNoTracking 一样，可在查询中的任何位置使用。 EF Core 现将生成以下 3 个 SQL 查询：

```sql
SELECT "a"."Id", "a"."Name"
FROM "Artists" AS "a"
ORDER BY "a"."Id"

SELECT "a0"."Id", "a0"."ArtistId", "a0"."Title", "a"."Id"
FROM "Artists" AS "a"
INNER JOIN "Album" AS "a0" ON "a"."Id" = "a0"."ArtistId"
ORDER BY "a"."Id", "a0"."Id"

SELECT "t"."Id", "t"."AlbumId", "t"."Name", "a"."Id", "a0"."Id"
FROM "Artists" AS "a"
INNER JOIN "Album" AS "a0" ON "a"."Id" = "a0"."ArtistId"
INNER JOIN "Tag" AS "t" ON "a0"."Id" = "t"."AlbumId"
ORDER BY "a"."Id", "a0"."Id"
```

支持对查询根目录执行各种操作。 其中包括 OrderBy/Skip/Take、联接操作、FirstOrDefault 和类似的单结果选择操作。

请注意，预览版 6 中不支持带 OrderBy/Skip/Take 的 Filtered Include，但它们可在日常版本中使用，且将包含在预览版 7 中。

#### <a name="split-queries-with-collection-projections"></a>用集合投影来拆分查询

在投影中加载集合时，也可使用 `AsSplitQuery`。 例如：

```CSharp
context.Artists
    .AsSplitQuery()
    .Select(e => new
    {
        Artist = e,
        Albums = e.Albums,
    }).ToList();
```

使用 SQLite 提供程序时，此 LINQ 查询会生成以下两个 SQL 查询：

```sql
SELECT "a"."Id", "a"."Name"
FROM "Artists" AS "a"
ORDER BY "a"."Id"

SELECT "a0"."Id", "a0"."ArtistId", "a0"."Title", "a"."Id"
FROM "Artists" AS "a"
INNER JOIN "Album" AS "a0" ON "a"."Id" = "a0"."ArtistId"
ORDER BY "a"."Id"
```

请注意，仅支持将集合具体化。 上例中 `e.Albums` 之后的任何组合都不会产生拆分查询。 这方面的改进由 [#21234](https://github.com/dotnet/efcore/issues/21234) 进行跟踪。

### <a name="indexattribute"></a>IndexAttribute

这个新的 IndexAttribute 可置于实体类型上来指定单列的索引。 例如：

```CSharp
[Index(nameof(FullName), IsUnique = true)]
public class User
{
    public int Id { get; set; }

    [MaxLength(128)]
    public string FullName { get; set; }
}
```

对于 SQL Server，迁移随后将生成以下 SQL：

```sql
CREATE UNIQUE INDEX [IX_Users_FullName]
    ON [Users] ([FullName])
    WHERE [FullName] IS NOT NULL;
```

IndexAttribute 也可用于指定横跨多个列的索引。 例如：

```CSharp
[Index(nameof(FirstName), nameof(LastName), IsUnique = true)]
public class User
{
    public int Id { get; set; }

    [MaxLength(64)]
    public string FirstName { get; set; }

    [MaxLength(64)]
    public string LastName { get; set; }
}
```

对于 SQL Server，这会导致：

```sql
CREATE UNIQUE INDEX [IX_Users_FirstName_LastName]
    ON [Users] ([FirstName], [LastName])
    WHERE [FirstName] IS NOT NULL AND [LastName] IS NOT NULL;
```

记录信息可通过问题 [#2407](https://github.com/dotnet/EntityFramework.Docs/issues/2407) 进行跟踪。

### <a name="improved-query-translation-exceptions"></a>改进了查询转换异常时显示的消息

我们将继续改进在查询转换失败时生成的异常消息。 例如，以下查询使用未映射的属性 `IsSigned`：

```CSharp
var artists = context.Artists.Where(e => e.IsSigned).ToList();
```

EF Core 将引发以下异常，指出由于 `IsSigned` 未映射而导致转换失败：

> 未经处理的异常。 System.InvalidOperationException：无法转换 LINQ 表达式 "DbSet<Artist>() .Where(a => a.IsSigned)"。 其他信息：实体类型 "Artist" 上成员 "IsSigned" 的转换失败。 可能未映射指定的成员。 请用可转换的形式重新编写查询，或者插入对 AsEnumerable()、AsAsyncEnumerable()、ToList() 或 ToListAsync() 的调用来显式切换到客户端评估。 有关更多信息，请参见 https://go.microsoft.com/fwlink/?linkid=2101038 。

同样地，在尝试用依赖区域性的语义来转换字符串比较时，现将生成信息更丰富的异常消息。 例如，以下查询尝试使用 `StringComparison.CurrentCulture`：

```CSharp
var artists = context.Artists
    .Where(e => e.Name.Equals("The Unicorns", StringComparison.CurrentCulture))
    .ToList();
```

EF Core 现将引发以下异常：

> 未经处理的异常。 System.InvalidOperationException：无法转换 LINQ 表达式 "DbSet<Artist>() .Where(a => a.Name.Equals( value:"The Unicorns", comparisonType:CurrentCulture))"。 其他信息：不支持采用 "StringComparison" 的 "string.Equals" 方法的转换。 有关更多信息，请参见 https://go.microsoft.com/fwlink/?linkid=2129535 。 请用可转换的形式重新编写查询，或者插入对 AsEnumerable()、AsAsyncEnumerable()、ToList() 或 ToListAsync() 的调用来显式切换到客户端评估。 有关更多信息，请参见 https://go.microsoft.com/fwlink/?linkid=2101038 。

### <a name="specify-transaction-id"></a>指定事务 ID

此功能是 [@Marusyk](https://github.com/Marusyk) 在社区中贡献的。 非常感谢贡献此功能！

EF Core 公开一个事务 ID 来跨调用关联事务。 此 ID 通常是在启动事务时由 EF Core 设置的。 如果转而由应用程序启动事务，则应用程序可使用此功能来显式设置事务 ID，使其在所用的每个位置都正确关联。 例如：

```CSharp
using (context.Database.UseTransaction(myTransaction, myId))
{
   ...
}
```

### <a name="ipaddress-mapping"></a>IPAddress 映射

此功能是 [@ralmsdeveloper](https://github.com/ralmsdeveloper) 在社区中贡献的。 非常感谢贡献此功能！

标准 .NET [IPAddress 类](/dotnet/api/system.net.ipaddress) 现自动映射到尚不具备原生支持的数据库的字符串列中。 例如，请考虑映射以下实体类型：

```CSharp
public class Host
{
    public int Id { get; set; }
    public IPAddress Address { get; set; }
}
```

在 SQL Server 上，这将导致迁移创建以下表：

```sql
CREATE TABLE [Host] (
    [Id] int NOT NULL,
    [Address] nvarchar(45) NULL,
    CONSTRAINT [PK_Host] PRIMARY KEY ([Id]));
```

之后，可按正常的方式添加实体：

```CSharp
context.AddRange(
    new Host { Address = IPAddress.Parse("127.0.0.1")},
    new Host { Address = IPAddress.Parse("0000:0000:0000:0000:0000:0000:0000:0001")});
```

所生成的 SQL 将插入规范化的 IPv4 或 IPv6 地址：

```sql
Executed DbCommand (14ms) [Parameters=[@p0='1', @p1='127.0.0.1' (Size = 45), @p2='2', @p3='::1' (Size = 45)], CommandType='Text', CommandTimeout='30']
      SET NOCOUNT ON;
      INSERT INTO [Host] ([Id], [Address])
      VALUES (@p0, @p1), (@p2, @p3);
```

### <a name="exclude-onconfiguring-when-scaffolding"></a>在创建基架时排除 OnConfiguring

在基于现有数据库创建 DbContext 的基架时，EF Core 会默认创建一个 OnConfiguring 重载，该重载有一个连接字符串，以便上下文可立即供用户使用。 不过，如果你只有带 OnConfiguring 的分部类，或者你要用其他方式部署上下文，则此方法将不起作用。

若要解决这种情况，现可指示基架命令不生成 OnConfiguring。 例如：

```
dotnet ef dbcontext scaffold "Data Source=(localdb)\MSSQLLocalDB;Initial Catalog=Chinook" Microsoft.EntityFrameworkCore.SqlServer --no-onconfiguring
```

或者在包管理器控制台中：

```
Scaffold-DbContext 'Data Source=(localdb)\MSSQLLocalDB;Initial Catalog=Chinook' Microsoft.EntityFrameworkCore.SqlServer -NoOnConfiguring
```

请注意，建议使用[命名的连接字符串和安全存储（如用户机密）](/core/managing-schemas/scaffolding?tabs=vs#configuration-and-user-secrets)。

### <a name="translations-for-firstordefault-on-strings"></a>转换字符串中的 FirstOrDefault

此功能是 [@dvoreckyaa](https://github.com/dvoreckyaa) 在社区中贡献的。 非常感谢贡献此功能！

现将转换字符串中字符的 FirstOrDefault 和类似运算符。 例如，使用 SQL Server 时，以下 LINQ 查询：

```CSharp
context.Customers.Where(c => c.ContactName.FirstOrDefault() == 'A').ToList();
```

将转换为以下 SQL：

```sql
SELECT [c].[Id], [c].[ContactName]
FROM [Customer] AS [c]
WHERE SUBSTRING([c].[ContactName], 1, 1) = N'A'
```

### <a name="simplify-case-blocks"></a>简化 CASE 块

EF Core 现使用 CASE 块生成效果更佳的查询。 例如，以下 LINQ 查询：

```CSharp
context.Weapons
    .OrderBy(w => w.Name.CompareTo("Marcus' Lancer") == 0)
    .ThenBy(w => w.Id)
```

之前在 SQL Server 上正式转换为：

```sql
SELECT [w].[Id], [w].[AmmunitionType], [w].[IsAutomatic], [w].[Name], [w].[OwnerFullName], [w].[SynergyWithId]
FROM [Weapons] AS [w]
ORDER BY CASE
    WHEN (CASE
        WHEN [w].[Name] = N'Marcus'' Lancer' THEN 0
        WHEN [w].[Name] > N'Marcus'' Lancer' THEN 1
        WHEN [w].[Name] < N'Marcus'' Lancer' THEN -1
    END = 0) AND CASE
        WHEN [w].[Name] = N'Marcus'' Lancer' THEN 0
        WHEN [w].[Name] > N'Marcus'' Lancer' THEN 1
        WHEN [w].[Name] < N'Marcus'' Lancer' THEN -1
    END IS NOT NULL THEN CAST(1 AS bit)
    ELSE CAST(0 AS bit)
END, [w].[Id]");
```

但它现在转换为：

```sql
SELECT [w].[Id], [w].[AmmunitionType], [w].[IsAutomatic], [w].[Name], [w].[OwnerFullName], [w].[SynergyWithId]
FROM [Weapons] AS [w]
ORDER BY CASE
    WHEN ([w].[Name] = N'Marcus'' Lancer') AND [w].[Name] IS NOT NULL THEN CAST(1 AS bit)
    ELSE CAST(0 AS bit)
END, [w].[Id]");
```

## <a name="preview-5"></a>预览版 5

### <a name="database-collations"></a>数据库排序规则

现可在 EF 模式中指定数据库的默认排序规则。 这将传输到生成的迁移，在创建数据库时设置排序规则。 例如：

```CSharp
modelBuilder.UseCollation("German_PhoneBook_CI_AS");
```

然后，迁移会生成以下内容，在 SQL Server 上创建数据库：

```sql
CREATE DATABASE [Test]
COLLATE German_PhoneBook_CI_AS;
```

可指定用于特定数据库列的排序规则。 例如：

```CSharp
 modelBuilder
     .Entity<User>()
     .Property(e => e.Name)
     .UseCollation("German_PhoneBook_CI_AS");
```

对于不使用迁移的列，可在创建 DbContext 基架时从数据库对排序规则进行反向工程处理。

最后，可通过 `EF.Functions.Collate()` 使用不同的排序规则进行即席查询。 例如：

```CSharp
context.Users.Single(e => EF.Functions.Collate(e.Name, "French_CI_AS") == "Jean-Michel Jarre");
```

这将为 SQL Server 生成以下查询：

```sql
SELECT TOP(2) [u].[Id], [u].[Name]
FROM [Users] AS [u]
WHERE [u].[Name] COLLATE French_CI_AS = N'Jean-Michel Jarre'
```

请注意，应谨慎使用即席排序规则，因为它们可能会对数据库性能造成不良影响。

文档可通过问题 [#2273](https://github.com/dotnet/EntityFramework.Docs/issues/2273) 进行跟踪。

### <a name="flow-arguments-into-idesigntimedbcontextfactory"></a>将参数传输到 IDesignTimeDbContextFactory

参数现从命令行传输到 [IDesignTimeDbContextFactory](/dotnet/api/microsoft.entityframeworkcore.design.idesigntimedbcontextfactory-1?view=efcore-3.1) 的 `CreateDbContext` 方法。 例如，若要指明这是开发生成，可以在命令行上传递自定义参数（例如 `dev`）：

```
dotnet ef migrations add two --verbose --dev
```

然后，该参数将传输到工厂，它在这里可用于控制如何创建和初始化上下文。 例如：

```CSharp
public class MyDbContextFactory : IDesignTimeDbContextFactory<SomeDbContext>
{
    public SomeDbContext CreateDbContext(string[] args)
        => new SomeDbContext(args.Contains("--dev"));
}
```

文档可通过问题 [#2419](https://github.com/dotnet/EntityFramework.Docs/issues/2419) 进行跟踪。

### <a name="no-tracking-queries-with-identity-resolution"></a>具有标识解析的非跟踪查询

现可将非跟踪查询配置来执行标识解析。 例如，以下查询将为每个 Post 创建一个新的 Blog 实例，即使每个 Blog 的主键相同也是如此。

```CSharp
context.Posts.AsNoTracking().Include(e => e.Blog).ToList();
```

然而，可更改此查询来确保只创建一个 Blog 实例，代价通常是稍微拖慢一些速度和总是使用更多的内存：

```CSharp
context.Posts.AsNoTracking().PerformIdentityResolution().Include(e => e.Blog).ToList();
```

请注意，这仅适用于非跟踪查询，因为所有跟踪查询已显示此行为。 此外，在 API 评审后，将更改 `PerformIdentityResolution` 语法。 请查看 [#19877](https://github.com/dotnet/efcore/issues/19877#issuecomment-637371073)。

文档可通过问题 [#1895](https://github.com/dotnet/EntityFramework.Docs/issues/1895) 进行跟踪。

### <a name="stored-persisted-computed-columns"></a>存储的（持久化）计算列

大多数数据库都允许在计算后存储计算得到的列值。 虽然这会占用磁盘空间，但仅在更新时对计算的列计算一次，而不是在每次检索它的值时都计算。 这样也可对某些数据库编制列的索引。

EF Core 5.0 允许将计算列配置为存储计算列。 例如：

```CSharp
modelBuilder
    .Entity<User>()
    .Property(e => e.SomethingComputed)
    .HasComputedColumnSql("my sql", stored: true);
```

### <a name="sqlite-computed-columns"></a>SQLite 计算列

EF Core 现支持在 SQLite 数据库中使用计算列。

## <a name="preview-4"></a>预览版 4

### <a name="configure-database-precisionscale-in-model"></a>在模型中配置数据库精度/小数位数

现在，可以使用模型生成器指定属性的精度和小数位数。 例如：

```CSharp
modelBuilder
    .Entity<Blog>()
    .Property(b => b.Numeric)
    .HasPrecision(16, 4);
```

还可以通过完整的数据库类型（如“decimal(16,4)”）设置精度和小数位数。

文档可通过问题 [#527](https://github.com/dotnet/EntityFramework.Docs/issues/527) 进行跟踪。

### <a name="specify-sql-server-index-fill-factor"></a>指定 SQL Server 索引填充因子

现在可以在 SQL Server 上创建索引时指定填充因子。 例如：

```CSharp
modelBuilder
    .Entity<Customer>()
    .HasIndex(e => e.Name)
    .HasFillFactor(90);
```

## <a name="preview-3"></a>预览版 3

### <a name="filtered-include"></a>经过筛选的包含

Include 方法现在支持筛选包含的实体。 例如：

```CSharp
var blogs = context.Blogs
    .Include(e => e.Posts.Where(p => p.Title.Contains("Cheese")))
    .ToList();
```

此查询将一并返回包含每个关联文章的博客（仅当文章标题包含“Cheese”时）。

Skip 和 Take 也可用于减少包含的实体数量。 例如：

```CSharp
var blogs = context.Blogs
    .Include(e => e.Posts.OrderByDescending(post => post.Title).Take(5)))
    .ToList();
```
此查询将返回博客，每个博客最多包含 5 篇文章。

有关完整详细信息，请参阅 [Include 文档](xref:core/querying/related-data#filtered-include)。

### <a name="new-modelbuilder-api-for-navigation-properties"></a>用于导航属性的新 ModelBuilder API

导航属性主要在[定义关系](xref:core/modeling/relationships)时配置。 但是，在导航属性需要额外配置的情况下，可以使用新的 `Navigation` 方法。 例如，如需在根据约定找不到支持字段时设置导航的支持字段，请执行以下操作：

```CSharp
modelBuilder.Entity<Blog>().Navigation(e => e.Posts).HasField("_myposts");
```

请注意：`Navigation` API 不会替换关系配置。 但是，它允许在已发现或定义的关系中进行其他导航属性配置。

请参阅[配置导航属性文档](xref:core/modeling/relationships#configuring-navigation-properties)。

### <a name="new-command-line-parameters-for-namespaces-and-connection-strings"></a>用于命名空间和连接字符串的新命令行参数

迁移和基架现在允许在命令行上指定命名空间。 例如，如需对数据库进行反向工程，将上下文和模型类放在不同的命名空间中：

```
dotnet ef dbcontext scaffold "connection string" Microsoft.EntityFrameworkCore.SqlServer --context-namespace "My.Context" --namespace "My.Model"
```

有关完整详细信息，请参阅[迁移](xref:core/managing-schemas/migrations/index#namespaces)和[反向工程](xref:core/managing-schemas/scaffolding#directories-and-namespaces)文档。

---
此外，连接字符串现在可以传递到 `database-update` 命令：

```
dotnet ef database update --connection "connection string"
```

有关完整的详细信息，请参阅[工具文档](xref:core/miscellaneous/cli/dotnet#dotnet-ef-database-update)。

等效参数已添加到 VS 程序包管理器控制台中使用的 PowerShell 命令中。

### <a name="enabledetailederrors-has-returned"></a>EnableDetailedErrors 已返回

出于性能原因，在从数据库读取值时，EF 不会执行额外的 null 检查。 当遇到意外的 null 时，这可能会导致难以查找根本原因的异常。

使用 `EnableDetailedErrors` 会将额外的 null 检查添加到查询中，这样一来，对于较小的性能开销，这些错误就更容易追溯到根本原因。

例如：
```CSharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    => optionsBuilder
        .EnableDetailedErrors()
        .EnableSensitiveDataLogging() // Often also useful with EnableDetailedErrors
        .UseSqlServer(Your.SqlServerConnectionString);
```

文档可通过问题 [#955](https://github.com/dotnet/EntityFramework.Docs/issues/955) 进行跟踪。

### <a name="cosmos-partition-keys"></a>Cosmos 分区键

现在可以在查询中指定用于给定查询的分区键。 例如：

```CSharp
await context.Set<Customer>()
             .WithPartitionKey(myPartitionKey)
             .FirstAsync();
```

文档可通过问题 [#2199](https://github.com/dotnet/EntityFramework.Docs/issues/2199) 进行跟踪。

### <a name="support-for-the-sql-server-datalength-function"></a>支持 SQL Server DATALENGTH 函数

可以使用新的 `EF.Functions.DataLength` 方法访问它。 例如：
```CSharp
var count = context.Orders.Count(c => 100 < EF.Functions.DataLength(c.OrderDate));
```

## <a name="preview-2"></a>预览版 2

### <a name="use-a-c-attribute-to-specify-a-property-backing-field"></a>使用 C# 特性指定属性支持字段

现在，可以使用 C# 特性指定属性的支持字段。 使用此特性，即使在不能自动找到支持字段时，EF Core 也能正常读写支持字段。 例如：

```CSharp
public class Blog
{
    private string _mainTitle;

    public int Id { get; set; }

    [BackingField(nameof(_mainTitle))]
    public string Title
    {
        get => _mainTitle;
        set => _mainTitle = value;
    }
}
```

文档可通过问题 [#2230](https://github.com/dotnet/EntityFramework.Docs/issues/2230) 进行跟踪。

### <a name="complete-discriminator-mapping"></a>完成鉴别器映射

EF Core 使用鉴别器列进行 [继承层次结构的 TPH 映射](/ef/core/modeling/inheritance)。 只要 EF Core 知道该鉴别器的所有可能的值，就可能实现某些性能改进。 EF Core 5.0 现在可实现这些改进。

例如，对于返回层次结构中所有类型的查询，旧版 EF Core 总是会生成以下 SQL：

```sql
SELECT [a].[Id], [a].[Discriminator], [a].[Name]
FROM [Animal] AS [a]
WHERE [a].[Discriminator] IN (N'Animal', N'Cat', N'Dog', N'Human')
```

配置完整的鉴别器映射后，EF Core 5.0 现在将生成以下内容：

```sql
SELECT [a].[Id], [a].[Discriminator], [a].[Name]
FROM [Animal] AS [a]
```

从预览版 3 开始，这将成为默认行为。

### <a name="performance-improvements-in-microsoftdatasqlite"></a>Microsoft.Data.Sqlite 中的性能改进

我们对 SQLIte 进行了两项性能改进：

* 现在，利用 SqliteBlob 和流，可以更高效地使用 GetBytes、GetChars 和 GetTextReader 检索二进制和字符串数据。
* SqliteConnection 的初始化现在很慢。

这些改进已实施到 ADO.NET Microsoft.Data.Sqlite 提供程序中，因此还可以在 EF Core 之外提升性能。

## <a name="preview-1"></a>预览版 1

### <a name="simple-logging"></a>简单的日志记录

此特性会添加类似于 EF6 中 `Database.Log` 的功能。 也就是说，它提供了一种从 EF Core 获取日志的简单方法，而不需要配置任何类型的外部日志记录框架。

初步文档包含在 [2019 年 12 月 5 日发布的 EF 每周状态](https://github.com/dotnet/efcore/issues/15403#issuecomment-562332863)中。

其他文档可通过问题 [#2085](https://github.com/dotnet/EntityFramework.Docs/issues/2085) 进行跟踪。

### <a name="simple-way-to-get-generated-sql"></a>获取生成的 SQL 的简单方法

EF Core 5.0 引入了 `ToQueryString` 扩展方法，该方法会返回执行 LINQ 查询时 EF Core 生成的 SQL。

初步文档包含在 [2020 年 1 月 9 日发布的 EF 每周状态](https://github.com/dotnet/efcore/issues/19549#issuecomment-572823246)中。

其他文档可通过问题 [#1331](https://github.com/dotnet/EntityFramework.Docs/issues/1331) 进行跟踪。

### <a name="use-a-c-attribute-to-indicate-that-an-entity-has-no-key"></a>使用 C# 特性指示实体没有键

现在可以将实体类型配置为不使用新 `KeylessAttribute`的键。 例如：

```CSharp
[Keyless]
public class Address
{
    public string Street { get; set; }
    public string City { get; set; }
    public int Zip { get; set; }
}
```

文档可通过问题 [#2186](https://github.com/dotnet/EntityFramework.Docs/issues/2186) 进行跟踪。

### <a name="connection-or-connection-string-can-be-changed-on-initialized-dbcontext"></a>可在初始化的 DbContext 上更改连接或连接字符串

现在可以更轻松地创建 DbContext 实例，而无需任何连接或连接字符串。 而且，现在可以在上下文实例上更改连接或连接字符串。 此功能使得同一个上下文实例能够动态地连接到不同的数据库。

文档可通过问题 [#2075](https://github.com/dotnet/EntityFramework.Docs/issues/2075) 进行跟踪。

### <a name="change-tracking-proxies"></a>更改跟踪代理

EF Core 现在可以生成自动实现 [INotifyPropertyChanging](/dotnet/api/system.componentmodel.inotifypropertychanging?view=netcore-3.1) 和 [INotifyPropertyChanged](/dotnet/api/system.componentmodel.inotifypropertychanged?view=netcore-3.1) 的运行时代理。 这些代理会将实体属性的值更改直接报告给 EF Core，从而无需扫描更改。 不过，代理有其自身的一组限制，因此并不适合所有人使用。

文档可通过问题 [#2076](https://github.com/dotnet/EntityFramework.Docs/issues/2076) 进行跟踪。

### <a name="enhanced-debug-views"></a>增强的调试视图

调试视图是调试问题时查看 EF Core 内部情况的一种简单方法。 在一段时间之前，我们就已经实现了模型的调试视图。 对于 EF Core 5.0，我们使模型视图更易于读取，并在状态管理器中为跟踪的实体添加了新的调试视图。

初步文档包含在 [2019 年 12 月 12 日发布的 EF 每周状态](https://github.com/dotnet/efcore/issues/15403#issuecomment-565196206)中。

其他文档可通过问题 [#2086](https://github.com/dotnet/EntityFramework.Docs/issues/2086) 进行跟踪。

### <a name="improved-handling-of-database-null-semantics"></a>改进了对数据库 null 语义的处理

关系数据库通常将 NULL 视为未知值，因此它不等于任何其他 NULL 值。 而 C# 将 null 视为与其他任何 null 相比均相等的定义值。 默认情况下，EF Core 会转换查询，使查询使用 C# null 语义。 EF Core 5.0 极大地提升了这些转换操作的效率。

文档可通过问题 [#1612](https://github.com/dotnet/EntityFramework.Docs/issues/1612) 进行跟踪。

### <a name="indexer-properties"></a>索引器属性

EF Core 5.0 支持 C# 索引器属性的映射。 这写属性使得实体能够充当属性包，实体中的列被映射为包中的命名属性。

文档可通过问题 [#2018](https://github.com/dotnet/EntityFramework.Docs/issues/2018) 进行跟踪。

### <a name="generation-of-check-constraints-for-enum-mappings"></a>为枚举映射生成检查约束

EF Core 5.0 迁移现可为枚举属性映射生成检查约束。 例如：

```SQL
MyEnumColumn VARCHAR(10) NOT NULL CHECK (MyEnumColumn IN ('Useful', 'Useless', 'Unknown'))
```

文档可通过问题 [#2082](https://github.com/dotnet/EntityFramework.Docs/issues/2082) 进行跟踪。

### <a name="isrelational"></a>IsRelational

除了现有 `IsSqlServer`、`IsSqlite` 和 `IsInMemory` 外，还添加了新的 `IsRelational` 方法。 此方法可用于测试 DbContext 是否使用任何关系数据库提供程序。 例如：

```CSharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    if (Database.IsRelational())
    {
        // Do relational-specific model configuration.
    }
}
```

文档可通过问题 [#2185](https://github.com/dotnet/EntityFramework.Docs/issues/2185) 进行跟踪。

### <a name="cosmos-optimistic-concurrency-with-etags"></a>使用 ETag 的 Cosmos 开放式并发

Azure Cosmos DB 数据库提供程序现在支持使用 ETag 的开放式并发。 使用 OnModelCreating 中的模型生成器配置 ETag：

```CSharp
builder.Entity<Customer>().Property(c => c.ETag).IsEtagConcurrency();
```

然后，SaveChanges 将在并发冲突上引发 `DbUpdateConcurrencyException`，[可以处理](/ef/core/saving/concurrency)它来实现重试等。

文档可通过问题 [#2099](https://github.com/dotnet/EntityFramework.Docs/issues/2099) 进行跟踪。

### <a name="query-translations-for-more-datetime-constructs"></a>查询转换以获取更多 DateTime 构造

现在，包含新 DataTime 构造的查询会被转换。

此外，现在映射了以下 SQL Server 函数：

* DateDiffWeek
* DateFromParts

例如：

```CSharp
var count = context.Orders.Count(c => date > EF.Functions.DateFromParts(DateTime.Now.Year, 12, 25));

```

文档可通过问题 [#2079](https://github.com/dotnet/EntityFramework.Docs/issues/2079) 进行跟踪。

### <a name="query-translations-for-more-byte-array-constructs"></a>查询转换以获取更多字节数组构造

现在，使用 Contains、Length、SequenceEqual 等对 byte[] 属性进行的查询会转换成 SQL。

初步文档包含在 [2019 年 12 月 5 日发布的 EF 每周状态](https://github.com/dotnet/efcore/issues/15403#issuecomment-562332863)中。

更多文档可通过问题 [#2079](https://github.com/dotnet/EntityFramework.Docs/issues/2079) 进行跟踪。

### <a name="query-translation-for-reverse"></a>反向查询转换

现在，使用 `Reverse` 的查询会被转换。 例如：

```CSharp
context.Employees.OrderBy(e => e.EmployeeID).Reverse()
```

文档可通过问题 [#2079](https://github.com/dotnet/EntityFramework.Docs/issues/2079) 进行跟踪。

### <a name="query-translation-for-bitwise-operators"></a>按位运算符的查询转换

现在，使用按位运算符的查询会在更多的情况下被转换，例如：

```CSharp
context.Orders.Where(o => ~o.OrderID == negatedId)
```

文档可通过问题 [#2079](https://github.com/dotnet/EntityFramework.Docs/issues/2079) 进行跟踪。

### <a name="query-translation-for-strings-on-cosmos"></a>Cosmos 上的字符串的查询转换

现在，在使用 Azure Cosmos DB 提供程序的情况下，使用字符串方法 Contains、StartsWith 和 EndsWith 的查询将被转换。

文档可通过问题 [#2079](https://github.com/dotnet/EntityFramework.Docs/issues/2079) 进行跟踪。
