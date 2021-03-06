---
title: EF Core 工具参考（程序包管理器控制台）-EF Core
author: bricelam
ms.author: bricelam
ms.date: 09/18/2018
uid: core/miscellaneous/cli/powershell
ms.openlocfilehash: 3893d561ccb7d97f3d9c25d9ea66509ad0f3da75
ms.sourcegitcommit: ebfd3382fc583bc90f0da58e63d6e3382b30aa22
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 06/25/2020
ms.locfileid: "85370588"
---
# <a name="entity-framework-core-tools-reference---package-manager-console-in-visual-studio"></a>Entity Framework Core 工具参考-Visual Studio 中的包管理器控制台

适用于 Entity Framework Core 的包管理器控制台（PMC）工具执行设计时开发任务。 例如，他们基于现有数据库创建[迁移](/aspnet/core/data/ef-mvc/migrations?view=aspnetcore-2.0)、应用迁移和生成模型的代码。 命令使用[包管理器控制台](/nuget/tools/package-manager-console)在 Visual Studio 中运行。 这些工具同时适用于 .NET Framework 和 .NET Core 项目。

如果未使用 Visual Studio，则建议改为使用[EF Core 命令行工具](dotnet.md)。 .NET Core CLI 工具是跨平台的，在命令提示符下运行。

## <a name="installing-the-tools"></a>安装工具

用于安装和更新这些工具的过程在 ASP.NET Core 2.1 及更早版本或其他项目类型之间有所不同。

### <a name="aspnet-core-version-21-and-later"></a>ASP.NET Core 版本2.1 及更高版本

工具将自动包含在 ASP.NET Core 2.1 + 项目中，因为 `Microsoft.EntityFrameworkCore.Tools` 包包含在[AspNetCore 元包](/aspnet/core/fundamentals/metapackage-app)中。

因此，您无需执行任何操作来安装这些工具，但必须执行以下操作：

* 在新项目中使用这些工具之前还原包。
* 安装包以将工具更新到较新版本。

为了确保获得最新版本的工具，我们建议您也执行以下步骤：

* 编辑 *.csproj*文件，并添加一行以指定最新版本的[microsoft.entityframeworkcore](https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Tools/)包。 例如， *.csproj*文件可能包含如下所示 `ItemGroup` 的：

  ```xml
  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.App" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="3.1.3" />
    <PackageReference Include="Microsoft.VisualStudio.Web.CodeGeneration.Design" Version="3.1.2" />
  </ItemGroup>
  ```

在收到类似于以下示例的消息时更新工具：

> EF Core 工具版本 "2.1.1-30846" 早于运行时 "2.1.3-32065"。 更新工具以获取最新功能和 bug 修复。

更新工具：

* 安装最新的 .NET Core SDK。
* 将 Visual Studio 更新到最新版本。
* 编辑 *.csproj*文件，使其包含对最新工具包的包引用，如前文所述。

### <a name="other-versions-and-project-types"></a>其他版本和项目类型

在**Package Manager console**中运行以下命令，安装包管理器控制台工具：

``` powershell
Install-Package Microsoft.EntityFrameworkCore.Tools
```

在**Package Manager Console**中运行以下命令，更新这些工具。

``` powershell
Update-Package Microsoft.EntityFrameworkCore.Tools
```

### <a name="verify-the-installation"></a>验证安装

通过运行以下命令验证是否已安装这些工具：

``` powershell
Get-Help about_EntityFrameworkCore
```

输出如下所示（不告诉您正在使用的工具的版本）：

```console

                     _/\__
               ---==/    \\
         ___  ___   |.    \|\
        | __|| __|  |  )   \\\
        | _| | _|   \_/ |  //|\\
        |___||_|       /   \\\/\\

TOPIC
    about_EntityFrameworkCore

SHORT DESCRIPTION
    Provides information about the Entity Framework Core Package Manager Console Tools.

<A list of available commands follows, omitted here.>
```

## <a name="using-the-tools"></a>使用工具

使用这些工具之前：

* 了解目标项目和启动项目之间的差异。
* 了解如何将工具与 .NET Standard 类库一起使用。
* 对于 ASP.NET Core 项目，请设置环境。

### <a name="target-and-startup-project"></a>目标和启动项目

命令引用*项目*和*启动项目*。

* 该*项目*也称为*目标项目*，因为它是命令在其中添加或删除文件的位置。 默认情况下，在 "**程序包管理器控制台**" 中选择的**默认项目**是目标项目。 可以使用选项指定其他项目作为目标项目 <nobr>`--project`</nobr> 。

* *启动项目*是工具生成和运行的项目。 这些工具必须在设计时执行应用程序代码，以获取有关项目的信息，例如数据库连接字符串和模型的配置。 默认情况下，**解决方案资源管理器**中的**启动项目**是启动项目。 您可以使用选项指定其他项目作为启动项目 <nobr>`--startup-project`</nobr> 。

启动项目和目标项目通常是同一个项目。 它们是不同的项目的典型方案是：

* EF Core 的上下文和实体类位于 .NET Core 类库中。
* .NET Core 控制台应用程序或 web 应用程序引用类库。

还可以[将迁移代码放在与 EF Core 上下文分离](xref:core/managing-schemas/migrations/projects)的类库中。

### <a name="other-target-frameworks"></a>其他目标框架

包管理器控制台工具适用于 .NET Core 或 .NET Framework 项目。 .NET Standard 类库中具有 EF Core 模型的应用可能没有 .NET Core 或 .NET Framework 项目。 例如，这适用于 Xamarin 和通用 Windows 平台应用。 在这种情况下，你可以创建一个 .NET Core 或 .NET Framework 的控制台应用程序项目，该项目的唯一用途是充当工具的启动项目。 项目可以是不包含实际代码的虚拟项目， &mdash; 只需为工具提供目标。

为什么需要虚拟项目？ 如前文所述，这些工具必须在设计时执行应用程序代码。 为此，需要使用 .NET Core 或 .NET Framework 运行时。 当 EF Core 模型位于面向 .NET Core 或 .NET Framework 的项目中时，EF Core 工具会借用项目中的运行时。 如果 EF Core 模型在 .NET Standard 类库中，则无法执行此操作。 .NET Standard 不是实际的 .NET 实现;它是 .NET 实现所必须支持的一组 Api 的规范。 因此 .NET Standard 不足以执行应用程序代码 EF Core 工具。 你创建的用于启动项目的虚拟项目提供了一个具体的目标平台，工具可在其中加载 .NET Standard 类库。

### <a name="aspnet-core-environment"></a>ASP.NET Core 环境

若要为 ASP.NET Core 项目指定环境，请在运行命令之前设置**env： ASPNETCORE_ENVIRONMENT** 。

## <a name="common-parameters"></a>通用参数

下表显示了所有 EF Core 命令共有的参数：

| 参数                 | 说明                                                                                                                                                                                                          |
|:--------------------------|:---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| -上下文\<String>        | 要使用的 `DbContext` 类。 仅命名空间或完全限定类名。  如果省略此参数，EF Core 将查找上下文类。 如果有多个上下文类，则此参数是必需的。 |
| -项目\<String>        | 目标项目。 如果省略此参数，则**包管理器控制台**的**默认项目**将用作目标项目。                                                                             |
| <nobr>-StartupProject</nobr>\<String> | 启动项目。 如果省略此参数，则使用**解决方案属性**中的**启动项目**作为目标项目。                                                                                 |
| -Verbose                  | 显示详细输出。                                                                                                                                                                                                 |

若要显示有关命令的帮助信息，请使用 PowerShell `Get-Help` 命令。

> [!TIP]
> 上下文、项目和 StartupProject 参数支持选项卡扩展。

## <a name="add-migration"></a>添加-迁移

添加新的迁移。

参数：

| 参数                         | 说明                                                                                                             |
|:----------------------------------|:------------------------------------------------------------------------------------------------------------------------|
| <nobr>-Name\<String><nobr>       | 迁移的名称。 这是一个位置参数，并且是必需的。                                              |
| <nobr>-OutputDir\<String></nobr> | 用于输出文件的目录。 路径相对于目标项目目录。 默认值为 "迁移"。 |
| <nobr>-命名空间\<String></nobr> | 要用于生成的类的命名空间。 默认为从输出目录生成。 （从 EFCore 5.0.0 开始提供。） |

## <a name="drop-database"></a>Drop 数据库

删除数据库。

参数：

| 参数 | 说明                                              |
|:----------|:---------------------------------------------------------|
| -WhatIf   | 显示要删除的数据库，但不删除它。 |

## <a name="get-dbcontext"></a>DbContext

获取有关类型的信息 `DbContext` 。

## <a name="remove-migration"></a>Remove-Migration

删除上一次迁移（回滚针对迁移进行的代码更改）。

参数：

| 参数 | 说明                                                                     |
|:----------|:--------------------------------------------------------------------------------|
| -Force    | 恢复迁移（回滚应用于数据库的更改）。 |

## <a name="scaffold-dbcontext"></a>基架-DbContext

为 `DbContext` 数据库的和实体类型生成代码。 为了使 `Scaffold-DbContext` 生成实体类型，数据库表必须具有主键。

参数：

| 参数                          | 说明                                                                                                                                                                                                                                                             |
|:-----------------------------------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| <nobr>-连接\<String></nobr> | 用于连接到数据库的连接字符串。 对于 ASP.NET Core 2.x 项目，值可以是*name = \<name of connection string> *。 在这种情况下，该名称来自为项目设置的配置源。 这是一个位置参数，并且是必需的。 |
| <nobr>-提供程序\<String></nobr>   | 要使用的提供程序。 通常，这是 NuGet 包的名称，例如： `Microsoft.EntityFrameworkCore.SqlServer` 。 这是一个位置参数，并且是必需的。                                                                                           |
| -OutputDir\<String>               | 要在其中放置文件的目录。 路径相对于项目目录。                                                                                                                                                                                             |
| -ContextDir\<String>              | 要在其中放置文件的目录 `DbContext` 。 路径相对于项目目录。                                                                                                                                                               |
| -命名空间\<String>               | 要用于所有生成的类的命名空间。 默认值为从根命名空间和输出目录生成。 （从 EFCore 5.0.0 开始提供。） |
| -ContextNamespace\<String>        | 要用于生成的类的命名空间 `DbContext` 。 注意：重写 `-Namespace` 。 （从 EFCore 5.0.0 开始提供。） |
| -上下文\<String>                 | `DbContext`要生成的类的名称。                                                                                                                                                                                                                          |
| -架构\<String[]>               | 要为其生成实体类型的表的架构。 如果省略此参数，则包括所有架构。                                                                                                                                                             |
| -表\<String[]>                | 要为其生成实体类型的表。 如果省略此参数，则包括所有表。                                                                                                                                                                         |
| -DataAnnotations                   | 使用属性配置模型（如果可能）。 如果省略此参数，则只使用 Fluent API。                                                                                                                                                      |
| -UseDatabaseNames                  | 使用表和列的名称与数据库中显示的名称完全相同。 如果省略此参数，则更改数据库名称以更严格地符合 c # 名称样式约定。                                                                                       |
| -Force                             | 覆盖现有文件。                                                                                                                                                                                                                                               |
| -NoOnConfiguring                   | 禁止 `OnConfiguring` 在生成的类中生成方法 `DbContext` 。 （从 EFCore 5.0.0 开始提供。） |

示例：

```powershell
Scaffold-DbContext "Server=(localdb)\mssqllocaldb;Database=Blogging;Trusted_Connection=True;" Microsoft.EntityFrameworkCore.SqlServer -OutputDir Models
```

示例：仅基架选定的表，并在具有指定名称和命名空间的单独文件夹中创建上下文：

```powershell
Scaffold-DbContext "Server=(localdb)\mssqllocaldb;Database=Blogging;Trusted_Connection=True;" Microsoft.EntityFrameworkCore.SqlServer -OutputDir Models -Tables "Blog","Post" -ContextDir Context -Context BlogContext -ContextNamespace New.Namespace
```

## <a name="script-migration"></a>脚本迁移

生成一个 SQL 脚本，该脚本将所选迁移中的所有更改应用于另一个选定的迁移。

参数：

| 参数                | 说明                                                                                                                                                                                                                |
|:-------------------------|:---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *-来自*\<String>        | 开始迁移。 可以按名称或 ID 识别迁移。 数字0是一个特殊情况，表示在*第一次迁移之前*。 默认值为 0。                                                              |
| *-到*\<String>          | 结束迁移。 默认为上次迁移。                                                                                                                                                                      |
| <nobr>-幂等</nobr> | 生成可用于任何迁移的数据库的脚本。                                                                                                                                                         |
| -输出\<String>        | 要向其写入结果的文件。 如果省略此参数，则会在创建应用的运行时文件所在的同一文件夹中创建具有生成名称的文件，例如： */obj/Debug/netcoreapp2.1/ghbkztfz.sql/*。 |

> [!TIP]
> To、From 和 Output 参数支持选项卡扩展。

以下示例使用迁移名称创建用于 InitialCreate 迁移的脚本。

```powershell
Script-Migration -To InitialCreate
```

以下示例使用迁移 ID，为 InitialCreate 迁移后的所有迁移创建一个脚本。

```powershell
Script-Migration -From 20180904195021_InitialCreate
```

## <a name="update-database"></a>Update-Database

将数据库更新到上次迁移或指定迁移。

| 参数                           | 说明                                                                                                                                                                                                                                                     |
|:------------------------------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| <nobr>*-迁移*\<String></nobr> | 目标迁移。 可以按名称或 ID 识别迁移。 数字0是一种特殊情况，表示在*第一次迁移之前*，并导致还原所有迁移。 如果未指定迁移，则该命令默认为上一次迁移。 |
| <nobr>-连接\<String></nobr>  | 用于连接到数据库的连接字符串。 默认为或中指定的 `AddDbContext` 一个 `OnConfiguring` 。 |

> [!TIP]
> 迁移参数支持选项卡扩展。

下面的示例将还原所有迁移。

```powershell
Update-Database -Migration 0
```

下面的示例将数据库更新为指定的迁移。 第一个使用迁移名称，第二个使用迁移 ID 和指定的连接：

```powershell
Update-Database -Migration InitialCreate
Update-Database -Migration 20180904195021_InitialCreate -Connection your_connection_string
```

## <a name="additional-resources"></a>其他资源

* [迁移](xref:core/managing-schemas/migrations/index)
* [反向工程](xref:core/managing-schemas/scaffolding)
