---
title: ASP.NET Core 中的 Razor 页面的筛选方法
author: Rick-Anderson
description: 了解如何为 ASP.NET Core 中的 Razor 页面创建筛选方法。
monikerRange: '>= aspnetcore-2.1'
ms.author: riande
ms.date: 12/28/2019
uid: razor-pages/filter
ms.openlocfilehash: 02771219454556b236080c2668243f788693b2c1
ms.sourcegitcommit: 077b45eceae044475f04c1d7ef2d153d7c0515a8
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 12/29/2019
ms.locfileid: "75542717"
---
# <a name="filter-methods-for-razor-pages-in-aspnet-core"></a>ASP.NET Core 中的 Razor 页面的筛选方法

::: moniker range=">= aspnetcore-3.0"

作者：[Rick Anderson](https://twitter.com/RickAndMSFT)

Razor 页面筛选器 [IPageFilter](/dotnet/api/microsoft.aspnetcore.mvc.filters.ipagefilter?view=aspnetcore-2.0) 和 [IAsyncPageFilter](/dotnet/api/microsoft.aspnetcore.mvc.filters.iasyncpagefilter?view=aspnetcore-2.0) 允许 Razor 页面在运行 Razor 页面处理程序的前后运行代码。 Razor 页面筛选器与 [ASP.NET Core MVC 操作筛选器](xref:mvc/controllers/filters#action-filters)类似，但它们不能应用于单个页面处理程序方法。

Razor 页面筛选器：

* 在选择处理程序方法后但在模型绑定发生前运行代码。
* 在模型绑定完成后，执行处理程序方法前运行代码。
* 在执行处理程序方法后运行代码。
* 可在页面或全局范围内实现。
* 无法应用于特定的页面处理程序方法。
* 可以具有通过[依赖关系注入](xref:fundamentals/dependency-injection)（DI）填充的构造函数依赖项。 有关详细信息，请参阅[ServiceFilterAttribute](/aspnet/core/mvc/controllers/filters#servicefilterattribute)和[TypeFilterAttribute](/aspnet/core/mvc/controllers/filters#typefilterattribute)。

可以在处理程序方法使用页构造函数或中间件执行之前运行代码，但只有 Razor 页面筛选器有权访问 <xref:Microsoft.AspNetCore.Mvc.RazorPages.PageModel.HttpContext>。 筛选器具有 <xref:Microsoft.AspNetCore.Mvc.Filters.FilterContext> 的派生参数，该参数提供对 `HttpContext`的访问。 例如，[实现筛选器属性](#ifa)示例将标头添加到响应中，而使用构造函数或中间件则无法做到这点。

[查看或下载示例代码](https://github.com/aspnet/AspNetCore.Docs/tree/master/aspnetcore/razor-pages/filter/3.1sample)（[如何下载](xref:index#how-to-download-a-sample)）

Razor 页面筛选器提供的以下方法可在全局或页面级应用：

* 同步方法：

  * [OnPageHandlerSelected](/dotnet/api/microsoft.aspnetcore.mvc.filters.ipagefilter.onpagehandlerselected?view=aspnetcore-2.0)：在选择处理程序方法后但在模型绑定发生前调用。
  * [OnPageHandlerExecuting](/dotnet/api/microsoft.aspnetcore.mvc.filters.ipagefilter.onpagehandlerexecuting?view=aspnetcore-2.0)：在执行处理器方法前，模型绑定完成后调用。
  * [OnPageHandlerExecuted](/dotnet/api/microsoft.aspnetcore.mvc.filters.ipagefilter.onpagehandlerexecuted?view=aspnetcore-2.0)：在执行处理器方法后，生成操作结果前调用。

* 异步方法：

  * [OnPageHandlerSelectionAsync](/dotnet/api/microsoft.aspnetcore.mvc.filters.iasyncpagefilter.onpagehandlerselectionasync?view=aspnetcore-2.0)：在选择处理程序方法后，但在模型绑定发生前，进行异步调用。
  * [OnPageHandlerExecutionAsync](/dotnet/api/microsoft.aspnetcore.mvc.filters.iasyncpagefilter.onpagehandlerexecutionasync?view=aspnetcore-2.0)：在调用处理程序方法前，但在模型绑定结束后，进行异步调用。

筛选器接口的同步和异步版本任意实现一个，而不是同时实现。 该框架会先查看筛选器是否实现了异步接口，如果是，则调用该接口。 如果不是，则调用同步接口的方法。 如果实现了这两个接口，则只调用异步方法。 对页面中的替代应用相同的规则，同步替代或异步替代只能任选其一实现，不可二者皆选。

## <a name="implement-razor-page-filters-globally"></a>全局实现 Razor 页面筛选器

以下代码实现了 `IAsyncPageFilter`：

[!code-csharp[Main](filter/3.1sample/PageFilter/Filters/SampleAsyncPageFilter.cs?name=snippet1)]

在前面的代码中，`ProcessUserAgent.Write` 是用户提供的与用户代理字符串一起使用的代码。

以下代码启用 `Startup` 类中的 `SampleAsyncPageFilter`：

[!code-csharp[Main](filter/3.1sample/PageFilter/Startup.cs?name=snippet2)]

下面的代码调用 <xref:Microsoft.AspNetCore.Mvc.ApplicationModels.PageConventionCollection.AddFolderApplicationModelConvention*> 将 `SampleAsyncPageFilter` 仅应用于 */Movies*中的页面：

[!code-csharp[Main](filter/3.1sample/PageFilter/Startup2.cs?name=snippet2)]

以下代码实现同步的 `IPageFilter`：

[!code-csharp[Main](filter/3.1sample/PageFilter/Filters/SamplePageFilter.cs?name=snippet1)]

以下代码启用 `SamplePageFilter`：

[!code-csharp[Main](filter/3.1sample/PageFilter/StartupSync.cs?name=snippet2)]

## <a name="implement-razor-page-filters-by-overriding-filter-methods"></a>通过重写筛选器方法实现 Razor 页面筛选器

以下代码将重写异步 Razor 页面筛选器：

[!code-csharp[Main](filter/3.1sample/PageFilter/Pages/Index.cshtml.cs?name=snippet)]

<a name="ifa"></a>

## <a name="implement-a-filter-attribute"></a>实现筛选器属性

基于属性的内置筛选器 <xref:Microsoft.AspNetCore.Mvc.Filters.IAsyncResultFilter.OnResultExecutionAsync*> 筛选器可以为子类。 以下筛选器向响应添加标头：

[!code-csharp[Main](filter/3.1sample/PageFilter/Filters/AddHeaderAttribute.cs)]

以下代码应用 `AddHeader` 属性：

[!code-csharp[Main](filter/3.1sample/PageFilter/Pages/Movies/Test.cshtml.cs)]

使用浏览器开发人员工具等工具来检查标头。 在 "**响应标头**" 下，将显示 `author: Rick`。

有关重写顺序的说明，请参阅[重写默认顺序](xref:mvc/controllers/filters#overriding-the-default-order)。

有关将筛选器管道与筛选器短路的说明，请参阅[取消和设置短路](xref:mvc/controllers/filters#cancellation-and-short-circuiting)。

<a name="auth"></a>

## <a name="authorize-filter-attribute"></a>授权筛选器属性

[Authorize](/dotnet/api/microsoft.aspnetcore.authorization.authorizeattribute?view=aspnetcore-2.0) 属性可应用于 `PageModel`：

[!code-csharp[Main](filter/sample/PageFilter/Pages/ModelWithAuthFilter.cshtml.cs?highlight=7)]

::: moniker-end

::: moniker range="< aspnetcore-3.0"

作者：[Rick Anderson](https://twitter.com/RickAndMSFT)

Razor 页面筛选器 [IPageFilter](/dotnet/api/microsoft.aspnetcore.mvc.filters.ipagefilter?view=aspnetcore-2.0) 和 [IAsyncPageFilter](/dotnet/api/microsoft.aspnetcore.mvc.filters.iasyncpagefilter?view=aspnetcore-2.0) 允许 Razor 页面在运行 Razor 页面处理程序的前后运行代码。 Razor 页面筛选器与 [ASP.NET Core MVC 操作筛选器](xref:mvc/controllers/filters#action-filters)类似，但它们不能应用于单个页面处理程序方法。

Razor 页面筛选器：

* 在选择处理程序方法后但在模型绑定发生前运行代码。
* 在模型绑定完成后，执行处理程序方法前运行代码。
* 在执行处理程序方法后运行代码。
* 可在页面或全局范围内实现。
* 无法应用于特定的页面处理程序方法。

代码可在使用页面构造函数或中间件执行处理程序方法前运行，但仅 Razor 页面筛选器可访问 [HttpContext](/dotnet/api/microsoft.aspnetcore.mvc.razorpages.pagemodel.httpcontext?view=aspnetcore-2.0#Microsoft_AspNetCore_Mvc_RazorPages_PageModel_HttpContext)。 筛选器具有 [FilterContext](/dotnet/api/microsoft.aspnetcore.mvc.filters.filtercontext?view=aspnetcore-2.0) 派生参数，可用于访问 `HttpContext`。 例如，[实现筛选器属性](#ifa)示例将标头添加到响应中，而使用构造函数或中间件则无法做到这点。

[查看或下载示例代码](https://github.com/aspnet/AspNetCore.Docs/tree/master/aspnetcore/razor-pages/filter/sample/PageFilter)（[如何下载](xref:index#how-to-download-a-sample)）

Razor 页面筛选器提供的以下方法可在全局或页面级应用：

* 同步方法：

  * [OnPageHandlerSelected](/dotnet/api/microsoft.aspnetcore.mvc.filters.ipagefilter.onpagehandlerselected?view=aspnetcore-2.0)：在选择处理程序方法后但在模型绑定发生前调用。
  * [OnPageHandlerExecuting](/dotnet/api/microsoft.aspnetcore.mvc.filters.ipagefilter.onpagehandlerexecuting?view=aspnetcore-2.0)：在执行处理器方法前，模型绑定完成后调用。
  * [OnPageHandlerExecuted](/dotnet/api/microsoft.aspnetcore.mvc.filters.ipagefilter.onpagehandlerexecuted?view=aspnetcore-2.0)：在执行处理器方法后，生成操作结果前调用。

* 异步方法：

  * [OnPageHandlerSelectionAsync](/dotnet/api/microsoft.aspnetcore.mvc.filters.iasyncpagefilter.onpagehandlerselectionasync?view=aspnetcore-2.0)：在选择处理程序方法后，但在模型绑定发生前，进行异步调用。
  * [OnPageHandlerExecutionAsync](/dotnet/api/microsoft.aspnetcore.mvc.filters.iasyncpagefilter.onpagehandlerexecutionasync?view=aspnetcore-2.0)：在调用处理程序方法前，但在模型绑定结束后，进行异步调用。

> [!NOTE]
> 筛选器接口的同步和异步版本**任意**实现一个，而不是同时实现。 该框架会先查看筛选器是否实现了异步接口，如果是，则调用该接口。 如果不是，则调用同步接口的方法。 如果实现了这两个接口，则只调用异步方法。 对页面中的替代应用相同的规则，同步替代或异步替代只能任选其一实现，不可二者皆选。

## <a name="implement-razor-page-filters-globally"></a>全局实现 Razor 页面筛选器

以下代码实现了 `IAsyncPageFilter`：

[!code-csharp[Main](filter/sample/PageFilter/Filters/SampleAsyncPageFilter.cs?name=snippet1)]

在前面的代码中，[ILogger](/dotnet/api/microsoft.extensions.logging.ilogger?view=aspnetcore-2.0) 不是必需的。 它在示例中用于提供应用程序的跟踪信息。

以下代码启用 `Startup` 类中的 `SampleAsyncPageFilter`：

[!code-csharp[Main](filter/sample/PageFilter/Startup.cs?name=snippet2&highlight=11)]

以下代码显示完整的 `Startup` 类：

[!code-csharp[Main](filter/sample/PageFilter/Startup.cs?name=snippet1)]

以下代码调用 `AddFolderApplicationModelConvention` 将 `SampleAsyncPageFilter` 仅应用于 /subFolder 中的页面：

[!code-csharp[Main](filter/sample/PageFilter/Startup2.cs?name=snippet2)]

以下代码实现同步的 `IPageFilter`：

[!code-csharp[Main](filter/sample/PageFilter/Filters/SamplePageFilter.cs?name=snippet1)]

以下代码启用 `SamplePageFilter`：

[!code-csharp[Main](filter/sample/PageFilter/StartupSync.cs?name=snippet2&highlight=11)]

## <a name="implement-razor-page-filters-by-overriding-filter-methods"></a>通过重写筛选器方法实现 Razor 页面筛选器

以下代码替代同步的 Razor 页面筛选器：

[!code-csharp[Main](filter/sample/PageFilter/Pages/Index.cshtml.cs)]

<a name="ifa"></a>

## <a name="implement-a-filter-attribute"></a>实现筛选器属性

基于属性的内置筛选器 [OnResultExecutionAsync](/dotnet/api/microsoft.aspnetcore.mvc.filters.iasyncresultfilter.onresultexecutionasync?view=aspnetcore-2.0#Microsoft_AspNetCore_Mvc_Filters_IAsyncResultFilter_OnResultExecutionAsync_Microsoft_AspNetCore_Mvc_Filters_ResultExecutingContext_Microsoft_AspNetCore_Mvc_Filters_ResultExecutionDelegate_) 可以进行子类化。 以下筛选器向响应添加标头：

[!code-csharp[Main](filter/sample/PageFilter/Filters/AddHeaderAttribute.cs)]

以下代码应用 `AddHeader` 属性：

[!code-csharp[Main](filter/sample/PageFilter/Pages/Contact.cshtml.cs?name=snippet1)]

有关重写顺序的说明，请参阅[重写默认顺序](xref:mvc/controllers/filters#overriding-the-default-order)。

有关将筛选器管道与筛选器短路的说明，请参阅[取消和设置短路](xref:mvc/controllers/filters#cancellation-and-short-circuiting)。 

<a name="auth"></a>

## <a name="authorize-filter-attribute"></a>授权筛选器属性

[Authorize](/dotnet/api/microsoft.aspnetcore.authorization.authorizeattribute?view=aspnetcore-2.0) 属性可应用于 `PageModel`：

[!code-csharp[Main](filter/sample/PageFilter/Pages/ModelWithAuthFilter.cshtml.cs?highlight=7)]

::: moniker-end
