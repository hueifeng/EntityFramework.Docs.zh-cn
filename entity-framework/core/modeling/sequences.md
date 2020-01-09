---
title: 序列-EF Core
author: roji
ms.date: 12/18/2019
ms.assetid: 94f81a92-3c72-4e14-912a-f99310374e42
uid: core/modeling/sequences
ms.openlocfilehash: 6343e58dd79837cc69308a070c9814030505d7dd
ms.sourcegitcommit: 32c51c22988c6f83ed4f8e50a1d01be3f4114e81
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 12/27/2019
ms.locfileid: "75502437"
---
# <a name="sequences"></a><span data-ttu-id="503a8-102">序列</span><span class="sxs-lookup"><span data-stu-id="503a8-102">Sequences</span></span>

> [!NOTE]  
> <span data-ttu-id="503a8-103">序列是通常仅由关系数据库支持的功能。</span><span class="sxs-lookup"><span data-stu-id="503a8-103">Sequences are a feature typically supported only by relational databases.</span></span> <span data-ttu-id="503a8-104">如果使用的是非关系数据库（如 Cosmos），请查看数据库文档以生成唯一值。</span><span class="sxs-lookup"><span data-stu-id="503a8-104">If you're using a non-relational database such as Cosmos, check your database documentation on generating unique values.</span></span>

<span data-ttu-id="503a8-105">序列将在数据库中生成唯一的顺序数值。</span><span class="sxs-lookup"><span data-stu-id="503a8-105">A sequence generates unique, sequential numeric values in the database.</span></span> <span data-ttu-id="503a8-106">序列不与特定表相关联，并且可以将多个表设置为从同一序列中绘制值。</span><span class="sxs-lookup"><span data-stu-id="503a8-106">Sequences are not associated with a specific table, and multiple tables can be set up to draw values from the same sequence.</span></span>

## <a name="basic-usage"></a><span data-ttu-id="503a8-107">基本用法</span><span class="sxs-lookup"><span data-stu-id="503a8-107">Basic usage</span></span>

<span data-ttu-id="503a8-108">您可以在模型中设置一个序列，然后使用它来生成属性值：</span><span class="sxs-lookup"><span data-stu-id="503a8-108">You can set up a sequence in the model, and then use it to generate values for properties:</span></span>

[!code-csharp[Main](../../../samples/core/Modeling/FluentAPI/Sequence.cs?name=Sequence&highlight=3,7)]

<span data-ttu-id="503a8-109">请注意，用于从序列生成值的特定 SQL 是特定于数据库的;上面的示例可在 SQL Server 上运行，但在其他数据库中将失败。</span><span class="sxs-lookup"><span data-stu-id="503a8-109">Note that the specific SQL used to generate a value from a sequence is database-specific; the above example works on SQL Server but will fail on other databases.</span></span> <span data-ttu-id="503a8-110">有关详细信息，请参阅特定数据库的文档。</span><span class="sxs-lookup"><span data-stu-id="503a8-110">Consult your specific database's documentation for more information.</span></span>

## <a name="configuring-sequence-settings"></a><span data-ttu-id="503a8-111">配置序列设置</span><span class="sxs-lookup"><span data-stu-id="503a8-111">Configuring sequence settings</span></span>

<span data-ttu-id="503a8-112">你还可以配置序列的其他方面，例如其架构、起始值、增量等：</span><span class="sxs-lookup"><span data-stu-id="503a8-112">You can also configure additional aspects of the sequence, such as its schema, start value, increment, etc.:</span></span>

[!code-csharp[Main](../../../samples/core/Modeling/FluentAPI/SequenceConfiguration.cs?name=SequenceConfiguration&highlight=3-5)]