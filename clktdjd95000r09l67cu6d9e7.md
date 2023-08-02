---
title: "WebApplication.CreateBuilder()와 새로운 CreateSlimBuilder() 메서드 비교하기"
datePublished: Wed Aug 02 2023 06:55:50 GMT+0000 (Coordinated Universal Time)
cuid: clktdjd95000r09l67cu6d9e7
slug: webapplicationcreatebuilder-createslimbuilder
canonical: https://andrewlock.net/exploring-the-dotnet-8-preview-comparing-createbuilder-to-the-new-createslimbuilder-method/
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1690956682669/d325bc8a-8c06-412f-8349-eca86b555b2c.jpeg
tags: net, dotnet

---

> ***Andrew Lock님의*** [Comparing WebApplication.CreateBuilder() to the new CreateSlimBuilder() method](https://andrewlock.net/exploring-the-dotnet-8-preview-comparing-createbuilder-to-the-new-createslimbuilder-method/)***를 DeepL의 도움을 받아 번역하였습니다.***

이번 포스팅은 [.NET 8 미리 보기 살펴보기](https://dimohy.hashnode.dev/series/exploring-dotnet8-preview) 시리즈의 세 번째 포스팅입니다.

[1부 - 새로운 구성 바인더 소스 생성기 사용하기](https://dimohy.hashnode.dev/7ioi66gc7jq0ioq1royessdrsjtsnbjrjzqg7iam7iqkioydneyeseq4scdsgqzsmqk)  
[2부 - 미니멀 API AOT 컴파일 템플릿](https://dimohy.hashnode.dev/the-minimal-api-aot-compilation-template)  
3부 - WebApplication.CreateBuilder()와 새로운 CreateSlimBuilder() 메서드 비교(이 게시물)  
4부 - 새로운 미니멀 API 소스 생성기 살펴보기  
5부 - 메서드 호출을 인터셉터로 대체하기

이 게시물에서는 새로운 `CreateSlimBuilder` 메서드를 살펴봅니다. 이 메서드는 AOT 시나리오를 지원하기 위해 기존 `WebApplication.CreateBuilder` 메서드의 대안으로 .NET 8에 도입되었습니다. 이 글에서는 슬림 빌더에서 빠진 기능에 대해 개략적으로 설명한 다음 코드를 자세히 살펴보고 어떻게 구현되는지 살펴봅니다.

> 이 게시물은 모두 프리뷰 빌드를 사용하고 있으므로 2023년 11월에 .NET 8이 최종 출시되기 전에 일부 기능이 변경(또는 제거)될 수 있습니다!

## `CreateSlimBuilder`가 필요한 이유는 무엇인가요?

[이전 게시물](https://dimohy.slogs.dev/the-minimal-api-aot-compilation-template)에서 .NET 8에 도입된 Ahead-of-time(AOT) 미니멀 API `api` 템플릿을 보여드렸습니다. 해당 템플릿의 첫 번째 줄은

```csharp
var builder = WebApplication.CreateSlimBuilder(args); 
```

`web` "빈" 템플릿의 해당 줄(.NET 6-8에서 본질적으로 변경되지 않음)과 비교합니다.

```csharp
var builder = WebApplication.CreateBuilder(args);
```

그렇다면 왜 이렇게 바뀐 걸까요?

[이전 글](https://dimohy.slogs.dev/the-minimal-api-aot-compilation-template)에서 설명했듯이 AOT 컴파일의 중요한 부분은 프레임워크와 앱에서 사용되지 않는 부분을 모두 제거하는 트리밍입니다. 이렇게 하면 최종 바이너리 크기가 크게 줄어들며 합리적인 바이너리 크기를 달성하는 데 필요합니다.

"일반적인" JIT 컴파일된 .NET 앱에서는 사용하지 않는 기능이나 필요하지 않은 기능을 포함하기 위해 바이너리 크기가 약간 커지지만, 그 정도는 상대적으로 미미합니다. 사용하지 않는 메서드를 한 번도 호출하지 않으면 컴파일되지 않으며, 많은 어셈블리가 로드되지 않을 수도 있습니다.

AOT를 사용하면 모든 새로운 기능에 대해 비용을 지불해야 합니다. 필요하지 않은 기능을 제거하면 최종 AOT 바이너리 크기에 큰 영향을 미칠 수 있습니다.

> [이전 글](https://dimohy.slogs.dev/the-minimal-api-aot-compilation-template)에서 설명했듯이 컴파일러는 앱에서 실제로 사용되는 유형과 메서드를 정적으로 결정할 수 있어야 하는데, 이는 일반적으로 리플렉션 기반 API가 문제가 된다는 의미입니다.

`CreateSlimBuilder`는 AOT와 호환되지 않거나 서버리스 및 클라우드 네이티브 앱과 같이 AOT가 빛을 발하는 앱에 유용하지 않은 여러 기능을 제거합니다. 이러한 종류의 앱을 대상으로 하지 않더라도 제거된 기능이 필요하지 않은 경우 `CreateSlimBuilder`를 고려할 수 있습니다. 다음 섹션에서는 이러한 변경 사항이 무엇인지 살펴보겠습니다.

## `CreateSlimBuilder`에서 누락된 기능은 무엇인가요?

`CreateSlimBuilder` 메서드는 `CreateBuilder`와 유사합니다. 두 메서드 모두 `WebApplicationBuilder`를 초기화하지만 `CreateSlimBuilder`는 [문서에 설명된 대로](https://learn.microsoft.com/en-gb/aspnet/core/fundamentals/native-aot?view=aspnetcore-8.0#the-createslimbuilder-method) 앱을 실행하는 데 필요한 최소한의 ASP.NET Core 기능만 초기화합니다. 즉, 누락되거나 변경된 사항이 많이 있습니다.

* [startup 어셈블리 지원](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/host/platform-specific-configuration?view=aspnetcore-7.0)하지 않음 (`IHostingStartup`)
    
* `UseStartup<Startup>` 호출 미지원
    
* 더 적은 로깅 공급자
    
    * Windows 이벤트 로그에 로깅하기 위한 [EventLog 로그 공급자](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/logging/?view=aspnetcore-7.0#welog) 없음
        
    * 디버거 콘솔에 로깅하기 위한 [Debug 공급자](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/logging/?view=aspnetcore-7.0#debug) 없음
        
    * [ETW](https://learn.microsoft.com/en-us/windows/win32/etw/event-tracing-portal)(Windows) 또는 [LTTng](https://lttng.org/)(Linux)에 쓰기 위한 [EventSource 공급자](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/logging/?view=aspnetcore-7.0#event-source) 없음
        
* 누락된 웹 호스팅 기능
    
    * [참조된 프로젝트 및 패키지에서 정적 에셋을 로드하기 위한](https://learn.microsoft.com/en-us/aspnet/core/razor-pages/ui-class?view=aspnetcore-7.0&tabs=visual-studio#consume-content-from-a-referenced-rcl) `UseStaticWebAssets()`가미지원
        
    * [IIS 통합 없음](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/?view=aspnetcore-7.0)
        
* Kestrel 구성에 누락된 기능
    
    * [HTTPS 지원](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel/endpoints?view=aspnetcore-7.0) 없음
        
    * [Quic(HTTP/3) 지원](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel/http3?view=aspnetcore-7.0) 없음
        
* 라우팅에서 [`Regex` 또는 `alpha` 제약 조건 미지원](https://github.com/dotnet/aspnetcore/issues/46142)
    

앱에서 `IHostingStartup`을 직접 사용해 본 적은 없지만 Azure 앱 서비스(AAS)에 배포해 본 적이 있다면 아마 모르고 사용했을 것입니다! `IHostingStartup`을 사용하면 런타임에 어셈블리를 로드하여 예를 들어 DI 컨테이너의 서비스를 사용자 지정하여 애플리케이션 구성 방식을 변경할 수 있습니다. 제가 마지막으로 확인한 바로는 AAS가 일부 통합을 추가하는 방식입니다. 하지만 앞서 언급했듯이 AOT에서는 JIT가 없기 때문에 런타임에 임의의 dll을 로드할 수 없습니다!

리플렉션을 사용하여 앱에서 메서드를 호출하기 때문에 AOT에서는 까다로운 또 다른 디자인이므로 `UseStartup<>`에 대한 지원은 제거되었습니다. 이것은 실제로 AOT 시나리오에서 가능할 수도 있지만, `WebApplicationBuilder`와 함께 `Startup` 클래스를 사용하는 것은 사람들이 일반적으로 하는 일이 아니므로 대부분 부피만 늘릴 뿐입니다.

> 애플리케이션에서 `builder.WebHost.UseStartup();` 을 호출하려고 하면 분석기에서 컴파일 타임 오류가 발생하여 런타임에 실패할 것이라는 경고가 표시됩니다. 저는 이와 같은 런타임 문제를 포착하기 위해 분석기를 도입한 ASP.NET 팀의 열렬한 팬입니다!

제거된 나머지 기능의 대부분은 클라우드/서버리스, Linux, 환경 등 AOT가 목표로 하는 시나리오에서 일반적으로 사용되지 않기 때문에 제거되었습니다.

* 이러한 환경은 Linux인 경우가 많으므로(시작 시간이 걱정된다면 Windows의 IIS 뒤에서 호스팅하지 않을 것입니다) EventLog 및 IIS 지원을 제거하는 것이 합리적입니다.
    
* 앱에서 인증서를 관리할 필요가 없도록 [TLS 종료 프록시](https://learn.microsoft.com/en-us/azure/application-gateway/ssl-overview) 뒤에서 HTTPS 또는 HTTP/3을 처리하는 것이 일반적입니다.
    
* 프로덕션 환경에서는 (다행히도!) 디버거가 첨부되어 있지 않을 것입니다.
    
* 앱을 게시할 때 `UseStaticWebAssets()`는 필요하지 않으며, 일반적으로 API 애플리케이션에서는 사용하지 않습니다.
    

`Regex` 경로 제약 조건이 지원되지 않는 것은 순전히 `Regex` 지원으로 인해 [많은 코드가 추가되기 때문이며, 특히 .NET 7 비백트래킹 지원의 경우 더욱 그렇습니다!](https://github.com/dotnet/aspnetcore/issues/46142) 기본적으로 인라인 `Regex` 제약 조건에 대한 지원을 제거하면 바이너리 크기에서 약 1MB가 제거됩니다. 필요한 경우 다음을 사용하여 [인라인 정규식 지원을 다시 활성화](https://github.com/dotnet/aspnetcore/issues/46428)할 수 있습니다.

```csharp
var builder = WebApplication.CreateSlimBuilder();

builder.Services.AddRoutingCore().Configure<RouteOptions>(options => {
    options.SetParameterPolicy<RegexInlineRouteConstraint>("regex");
});
```

> 로깅 공급자 또는 HTTPS 지원과 같은 일부 기능을 다시 추가하려면 그렇게 할 수 있습니다. 예를 들어 HTTPS 지원을 활성화하려면 `builder.WebHost.UseKestrelHttpsConfiguration()`을 호출하세요.

`CreateSlimBuilder()`를 사용하는 데 관심이 있다면 이 정도만 알아두면 됩니다. 다음 섹션에서는 빌더의 모든 실제 변경 사항을 자세히 살펴보겠습니다.

## 구현 방법

이 섹션에서는 `CreateSlimBuilder()`가 어떻게 구현되는지, 주로 기존의 `CreateBuilder()` 메서드와 비교하여 살펴봅니다.

> 경고: 이 섹션은 초보자를 위한 것이 아니며 매우 건조합니다. 여러분이 알아야 할 것보다 훨씬 더 깊이 있는 내용이며, 대부분 이 글의 전반부를 이해하기 위해 제가 겪은 일들입니다!

`WebApplicationBuilder`는 .NET 6에 추가되었지만 이전 버전의 .NET Core에 도입된 기존의 모든 일반 `IHostBuilder` 및 `IWebHostBuilder` 추상화를 기반으로 합니다. 결과적으로 프랑켄슈타인의 괴물 같은 컴포넌트입니다! 대부분의 경우 `CreateSlimBuilder()`는 `CreateBuilder()`와 매우 유사한 웹 응용 프로그램 빌더를 생성합니다.

> [이전 포스팅](https://andrewlock.net/exploring-dotnet-6-part-3-exploring-the-code-behind-webapplicationbuilder/)에서 다양한 `HostBuilder` 인스턴스를 결합하여 `WebApplicationBuilder`가 어떻게 작동하는지 자세히 살펴봤습니다. `WebApplicationBuilder`를 전반적으로 더 잘 이해하고 싶으시다면 해당 포스팅을 읽어보시기 바랍니다!

두 메서드가 모두 `static` 헬퍼로 정의되어 있는 [`WebApplication`의](https://github.com/dotnet/aspnetcore/blob/96ba5bd21d29ef7ebd730c40942c055d3535b9db/src/DefaultBuilder/src/WebApplication.cs#L25) 메서드 정의부터 살펴보겠습니다.

```csharp
public sealed class WebApplication : IHost, IApplicationBuilder, IEndpointRouteBuilder, IAsyncDisposable
{
    public static WebApplicationBuilder CreateBuilder() =>
        new WebApplicationBuilder(new WebApplicationOptions());

    public static WebApplicationBuilder CreateSlimBuilder() =>
        new WebApplicationBuilder(new WebApplicationOptions(), slim: true);
                                                            // 👆 the only difference
}
```

보시다시피, 이 메서드들은 서로 다른 WebApplicationBuilder 생성자를 호출하며, 슬림 빌더는 `slim: true` 값을 전달합니다.

이 두 생성자를 비교해보면(아래 차이점 참조), (다소 놀랍게도) "슬림" 버전에 훨씬 더 많은 추가 코드가 포함되어 있음을 알 수 있습니다! 그 이유는 곧 알게 되겠지만 기본 빌더는 이 작업의 많은 부분을 다른 헬퍼 메서드에 위임하는 반면, 슬림 빌더는 여기서 대부분의 작업을 인라인으로 수행하기 때문입니다(그리고 필요하지 않은 것은 제거합니다). 실제로 "주요" 변경 사항은 두 가지뿐입니다:

슬림 빌더는 새로운 `HostApplicationBuilder()` 대신 `Host.CreateEmptyApplicationBuilder()`를 호출합니다. 슬림 빌더는 `ConfigureWebHostDefaults()` 대신 `ConfigureSlimWebHost()` 및 `ConfigureWebDefaultsCore()`를 호출합니다.

```diff
- internal WebApplicationBuilder(WebApplicationOptions options, Action<IHostBuilder>? configureDefaults = null)
+ internal WebApplicationBuilder(WebApplicationOptions options, bool slim, Action<IHostBuilder>? configureDefaults = null)
{
    var configuration = new ConfigurationManager();

    configuration.AddEnvironmentVariables(prefix: "ASPNETCORE_");

+   // SetDefaultContentRoot needs to be added between 'ASPNETCORE_' and 'DOTNET_' in order to match behavior of the non-slim WebApplicationBuilder.
+   SetDefaultContentRoot(options, configuration);

+   // Add the default host environment variable configuration source.
+   // This won't be added by CreateEmptyApplicationBuilder.
+   configuration.AddEnvironmentVariables(prefix: "DOTNET_");

+   _hostApplicationBuilder = Microsoft.Extensions.Hosting.Host.CreateEmptyApplicationBuilder(new HostApplicationBuilderSettings
-   _hostApplicationBuilder = new HostApplicationBuilder(new HostApplicationBuilderSettings
    {
        Args = options.Args,
        ApplicationName = options.ApplicationName,
        EnvironmentName = options.EnvironmentName,
        ContentRootPath = options.ContentRootPath,
        Configuration = configuration,
    });

+   // Ensure the same behavior of the non-slim WebApplicationBuilder by adding the default "app" Configuration sources
+   ApplyDefaultAppConfigurationSlim(_hostApplicationBuilder.Environment, configuration, options.Args);
+   AddDefaultServicesSlim(configuration, _hostApplicationBuilder.Services);

+   // configure the ServiceProviderOptions here since CreateEmptyApplicationBuilder won't.
+   var serviceProviderFactory = GetServiceProviderFactory(_hostApplicationBuilder);
+   _hostApplicationBuilder.ConfigureContainer(serviceProviderFactory);

    // Set WebRootPath if necessary
    if (options.WebRootPath is not null)
    {
        Configuration.AddInMemoryCollection(new[]
        {
            new KeyValuePair<string, string?>(WebHostDefaults.WebRootKey, options.WebRootPath),
        });
    }

    // Run methods to configure web host defaults early to populate services
    var bootstrapHostBuilder = new BootstrapHostBuilder(_hostApplicationBuilder);

    // This is for testing purposes
    configureDefaults?.Invoke(bootstrapHostBuilder);

-   bootstrapHostBuilder.ConfigureWebHostDefaults(webHostBuilder =>
+   bootstrapHostBuilder.ConfigureSlimWebHost(webHostBuilder =>
    {
+       AspNetCore.WebHost.ConfigureWebDefaultsCore(webHostBuilder);

        // Runs inline.
        webHostBuilder.Configure(ConfigureApplication);

        webHostBuilder.UseSetting(WebHostDefaults.ApplicationKey, _hostApplicationBuilder.Environment.ApplicationName ?? "");
        webHostBuilder.UseSetting(WebHostDefaults.PreventHostingStartupKey, Configuration[WebHostDefaults.PreventHostingStartupKey]);
        webHostBuilder.UseSetting(WebHostDefaults.HostingStartupAssembliesKey, Configuration[WebHostDefaults.HostingStartupAssembliesKey]);
        webHostBuilder.UseSetting(WebHostDefaults.HostingStartupExcludeAssembliesKey, Configuration[WebHostDefaults.HostingStartupExcludeAssembliesKey]);
    },
    options =>
    {
        // We've already applied "ASPNETCORE_" environment variables to hosting config
        options.SuppressEnvironmentConfiguration = true;
    });

    // This applies the config from ConfigureWebHostDefaults
    // Grab the GenericWebHostService ServiceDescriptor so we can append it after any user-added IHostedServices during Build();
    _genericWebHostServiceDescriptor = bootstrapHostBuilder.RunDefaultCallbacks();

    // Grab the WebHostBuilderContext from the property bag to use in the ConfigureWebHostBuilder. Then
    // grab the IWebHostEnvironment from the webHostContext. This also matches the instance in the IServiceCollection.
    var webHostContext = (WebHostBuilderContext)bootstrapHostBuilder.Properties[typeof(WebHostBuilderContext)];
    Environment = webHostContext.HostingEnvironment;

    Host = new ConfigureHostBuilder(bootstrapHostBuilder.Context, Configuration, Services);
    WebHost = new ConfigureWebHostBuilder(webHostContext, Configuration, Services);
}
```

[`CreateEmptyApplicationBuilder()`는 간단한 공용 헬퍼 메서드](https://github.com/dotnet/runtime/blob/1ea78a4a258fb5194443f9600632af30e1e81b3d/src/libraries/Microsoft.Extensions.Hosting/src/Host.cs#L108)로 새 `HostApplicationBuilder()`를 생성합니다.

```csharp
public static class Host
{
    public static HostApplicationBuilder CreateEmptyApplicationBuilder(HostApplicationBuilderSettings? settings)
            => new HostApplicationBuilder(settings, empty: true);
}
```

따라서 빈/슬림 버전은 이번에는 다른 생성자, 즉 `HostApplicationBuilder`를 호출합니다. [기본 생성자와 "빈" 생성자](https://github.com/dotnet/runtime/blob/1ea78a4a258fb5194443f9600632af30e1e81b3d/src/libraries/Microsoft.Extensions.Hosting/src/HostApplicationBuilder.cs)(아래)를 비교하면 많은 코드가 제거되었음을 알 수 있습니다. 이 코드의 대부분은 실제로 `WebApplicationBuilder` 생성자로 옮겨졌지만 몇 가지 주목할 만한 변경 사항이 있습니다.

* 슬림 버전은 `ApplyDefaultAppConfiguration()` 대신 `ApplyDefaultAppConfigurationSlim()`을 호출합니다. 제가 알기로 [슬림 버전](https://github.com/dotnet/aspnetcore/blob/9178b9b117e64ca75821db1927e4857714c5da97/src/DefaultBuilder/src/WebApplicationBuilder.cs#LL209C25-L209C57)은 [기본 버전](https://github.com/dotnet/runtime/blob/6149ca07d2202c2d0d518e10568c0d0dd3473576/src/libraries/Microsoft.Extensions.Hosting/src/HostingHostBuilderExtensions.cs#L229-L256)과 본질적으로 동일하지만 `ConfigurationManager`를 사용합니다.
    
* 슬림 버전은 `AddDefaultServices()` 대신 `AddDefaultServicesSlim()`을 호출합니다. [슬림 메서드](https://github.com/dotnet/aspnetcore/blob/96ba5bd21d29ef7ebd730c40942c055d3535b9db/src/DefaultBuilder/src/WebApplicationBuilder.cs#L254-L270)는 제거된 로깅 공급자를 추가하는 반면, [기본 메서드는 더 완전한 집합을 추가](https://github.com/dotnet/runtime/blob/6149ca07d2202c2d0d518e10568c0d0dd3473576/src/libraries/Microsoft.Extensions.Hosting/src/HostingHostBuilderExtensions.cs#L266-L309)합니다.
    

```diff
- public HostApplicationBuilder(HostApplicationBuilderSettings? settings)
+ internal HostApplicationBuilder(HostApplicationBuilderSettings? settings, bool empty)
{
    settings ??= new HostApplicationBuilderSettings();
    Configuration = settings.Configuration ?? new ConfigurationManager();

-    if (!settings.DisableDefaults)
-    {
-        if (settings.ContentRootPath is null && Configuration[HostDefaults.ContentRootKey] is null)
-        {
-            HostingHostBuilderExtensions.SetDefaultContentRoot(Configuration);
-        }
-
-        Configuration.AddEnvironmentVariables(prefix: "DOTNET_");
-    }

    Initialize(settings, out _hostBuilderContext, out _environment, out _logging);

-    ServiceProviderOptions? serviceProviderOptions = null;
-    if (!settings.DisableDefaults)
-    {
-        HostingHostBuilderExtensions.ApplyDefaultAppConfiguration(_hostBuilderContext, Configuration, settings.Args);
-        HostingHostBuilderExtensions.AddDefaultServices(_hostBuilderContext, Services);
-        serviceProviderOptions = HostingHostBuilderExtensions.CreateDefaultServiceProviderOptions(_hostBuilderContext);
-    }

    _createServiceProvider = () =>
    {
        // Call _configureContainer in case anyone adds callbacks via HostBuilderAdapter.ConfigureContainer<IServiceCollection>() during build.
        // Otherwise, this no-ops.
        _configureContainer(Services);
-       return serviceProviderOptions is null ? Services.BuildServiceProvider() : Services.BuildServiceProvider(serviceProviderOptions);
+       return Services.BuildServiceProvider();
    };
}
```

다음으로 `ConfigureWebHost`와 `ConfigureSlimWebHost`의 차이점을 살펴보겠습니다. 아래에서 볼 수 있듯이, [이 확장 메서드들은](https://github.com/dotnet/aspnetcore/blob/9178b9b117e64ca75821db1927e4857714c5da97/src/Hosting/Hosting/src/GenericHostWebHostBuilderExtensions.cs#L51) 각각 다른 `IWebHostBuilder` 구현을 생성합니다. `GenericWebHostBuilder`와 `SlimWebHostBuilder`를 각각 생성합니다.

```diff
- public static IHostBuilder ConfigureWebHost(this IHostBuilder builder, Action<IWebHostBuilder> configure, Action<WebHostBuilderOptions> configureWebHostBuilder)
+ public static IHostBuilder ConfigureSlimWebHost(this IHostBuilder builder, Action<IWebHostBuilder> configure, Action<WebHostBuilderOptions> configureWebHostBuilder)
{
    return ConfigureWebHost(
        builder,
-       static (hostBuilder, options) => new GenericWebHostBuilder(hostBuilder, options),
+       static (hostBuilder, options) => new SlimWebHostBuilder(hostBuilder, options),
        configure,
        configureWebHostBuilder);
}
```

이 두 클래스의 생성자를 비교해보면, [`SlimWebHostBuilder`](https://github.com/dotnet/aspnetcore/blob/9178b9b117e64ca75821db1927e4857714c5da97/src/Hosting/Hosting/src/GenericHost/SlimWebHostBuilder.cs)는 실제로는 [`GenericWebHostBuilder`](https://github.com/dotnet/aspnetcore/blob/9178b9b117e64ca75821db1927e4857714c5da97/src/Hosting/Hosting/src/GenericHost/GenericWebHostBuilder.cs) 의 슬림 버전입니다. 앞서 설명한 이유 때문에 두 가지 기능이 생략되었습니다.

* 호스팅 어셈블리(`IHostingStartup`) 지원이 제거됨
    
* `UseStartup<T>` 지원이 제거됨
    

```diff
+ public SlimWebHostBuilder(IHostBuilder builder, WebHostBuilderOptions options)
- public GenericWebHostBuilder(IHostBuilder builder, WebHostBuilderOptions options)
    : base(builder, options)
{
    _builder.ConfigureHostConfiguration(config =>
    {
        config.AddConfiguration(_config);

-       // We do this super early but still late enough that we can process the configuration
-       // wired up by calls to UseSetting
-       ExecuteHostingStartups();
    });

-   // IHostingStartup needs to be executed before any direct methods on the builder
-   // so register these callbacks first
-   _builder.ConfigureAppConfiguration((context, configurationBuilder) =>
-   {
-       if (_hostingStartupWebHostBuilder != null)
-       {
-           var webhostContext = GetWebHostBuilderContext(context);
-           _hostingStartupWebHostBuilder.ConfigureAppConfiguration(webhostContext, configurationBuilder);
-       }
-   });

    _builder.ConfigureServices((context, services) =>
    {
        var webhostContext = GetWebHostBuilderContext(context);
        var webHostOptions = (WebHostOptions)context.Properties[typeof(WebHostOptions)];

        // Add the IHostingEnvironment and IApplicationLifetime from Microsoft.AspNetCore.Hosting
        services.AddSingleton(webhostContext.HostingEnvironment);
#pragma warning disable CS0618 // Type or member is obsolete
        services.AddSingleton((AspNetCore.Hosting.IHostingEnvironment)webhostContext.HostingEnvironment);
        services.AddSingleton<IApplicationLifetime, GenericWebHostApplicationLifetime>();
#pragma warning restore CS0618 // Type or member is obsolete

        services.Configure<GenericWebHostServiceOptions>(options =>
        {
            // Set the options
            options.WebHostOptions = webHostOptions;
-           // Store and forward any startup errors
-           options.HostingStartupExceptions = _hostingStartupErrors;
        });

        // REVIEW: This is bad since we don't own this type. Anybody could add one of these and it would mess things up
        // We need to flow this differently
        services.TryAddSingleton(sp => new DiagnosticListener("Microsoft.AspNetCore"));
        services.TryAddSingleton<DiagnosticSource>(sp => sp.GetRequiredService<DiagnosticListener>());
        services.TryAddSingleton(sp => new ActivitySource("Microsoft.AspNetCore"));
        services.TryAddSingleton(DistributedContextPropagator.Current);

        services.TryAddSingleton<IHttpContextFactory, DefaultHttpContextFactory>();
        services.TryAddScoped<IMiddlewareFactory, MiddlewareFactory>();
        services.TryAddSingleton<IApplicationBuilderFactory, ApplicationBuilderFactory>();

        services.AddMetrics();
        services.TryAddSingleton<HostingMetrics>();

-       // IMPORTANT: This needs to run *before* direct calls on the builder (like UseStartup)
-       _hostingStartupWebHostBuilder?.ConfigureServices(webhostContext, services);

-       // Support UseStartup(assemblyName)
-       if (!string.IsNullOrEmpty(webHostOptions.StartupAssembly))
-       {
-           ScanAssemblyAndRegisterStartup(context, services, webhostContext, webHostOptions);
-       }
    });
}
```

다음으로 `ConfigureWebDefaults`와 `ConfigureWebDefaultsCore`의 차이점에 대해 알아보겠습니다. [전자](https://github.com/dotnet/aspnetcore/blob/9178b9b117e64ca75821db1927e4857714c5da97/src/DefaultBuilder/src/WebApplicationBuilder.cs#L253)는 [기본 `ConfigureWebHostDefaults()`](https://github.com/dotnet/aspnetcore/blob/96ba5bd21d29ef7ebd730c40942c055d3535b9db/src/DefaultBuilder/src/GenericHostBuilderExtensions.cs#L58C32-L58C56) 메서드에서 호출됩니다("일반" `WebApplicationBuilder` 생성자에서 호출됨). 후자는 "슬림" `WebApplicationBuilder` 생성자 내부에서 직접 호출됩니다.

여기서 분명한 차이점은 슬림 버전은 앞서 설명한 것처럼 IIS 통합이나 정적 웹 자산 어셈블리를 추가하지 않는다는 것입니다. 하지만 Kestrel을 구성하는 `ConfigureWebDefaultsWorker()` 메서드가 호출되는 방식에도 차이가 있습니다.

```diff
- internal static void ConfigureWebDefaults(IWebHostBuilder builder)
+ internal static void ConfigureWebDefaultsCore(IWebHostBuilder builder)
{
-   builder.ConfigureAppConfiguration((ctx, cb) =>
-   {
-       if (ctx.HostingEnvironment.IsDevelopment())
-       {
-           StaticWebAssetsLoader.UseStaticWebAssets(ctx.HostingEnvironment, ctx.Configuration);
-       }
-   });

    ConfigureWebDefaultsWorker(
-        builder.UseKestrel(ConfigureKestrel),
+        builder.UseKestrelCore().ConfigureKestrel(ConfigureKestrel), 
-        configureRouting: services => services.AddRouting()
+        configureRouting: null
    );

-   builder
-       .UseIIS()
-       .UseIISIntegration();
}
```

이 구성의 차이를 비교하기 위해 "원래" `UseKestrel(ConfigureKestrel)` 메서드를 인라인으로 사용하면 다음과 같은 결과를 얻을 수 있습니다.

```diff
- builder.UseKestrel().ConfigureKestrel(ConfigureKestrel)
+ builder.UseKestrelCore().ConfigureKestrel(ConfigureKestrel), 
```

이제 유일한 차이점은 `UseKestrel()`과 `UseKestrelCore()`라는 것이 더 분명해졌습니다. `UseKestrel()`을 살펴보면 기본 빌더와 슬림 빌더의 Kestrel 구성에서 유일한 차이점은 예상대로 HTTPS와 Quic 지원뿐이라는 것을 알 수 있습니다.

```csharp
public static IWebHostBuilder UseKestrel(this IWebHostBuilder hostBuilder)
{
    return hostBuilder
        .UseKestrelCore()
        .UseKestrelHttpsConfiguration() // 👈 missing in the "slim" builder
        .UseQuic(options => // 👈 missing in the "slim" builder
        {
            // Configure server defaults to match client defaults.
            // https://github.com/dotnet/runtime/blob/a5f3676cc71e176084f0f7f1f6beeecd86fbeafc/src/libraries/System.Net.Http/src/System/Net/Http/SocketsHttpHandler/ConnectHelper.cs#L118-L119
            options.DefaultStreamErrorCode = (long)Http3ErrorCode.RequestCancelled;
            options.DefaultCloseErrorCode = (long)Http3ErrorCode.NoError;
        });
}
```

마지막으로 빌더 간에 `ConfigureWebDefaultsWorker`가 호출되는 방식에 한 가지 차이점이 있습니다:

* 기본 버전에서는 `ConfigureWebDefaultsWorker()`에 `AddRouting()`을 호출하는 람다가 전달됩니다.
    
* 슬림 버전에서는 `null`이 전달되므로 `ConfigureWebDefaultsWorker()`는 대신 `AddRoutingCore()`를 호출합니다.
    

`AddRouting()`의 코드를 살펴보면 여기에서 기본 빌더에 정규식 경로 제약 조건이 추가되는 것을 볼 수 있습니다(또는 보는 방식에 따라 슬림 빌더에서 제거되는 것을 볼 수 있습니다!)

```csharp
public static IServiceCollection AddRouting(this IServiceCollection services)
{
    services.AddRoutingCore();
    services.TryAddEnumerable(ServiceDescriptor.Singleton<IConfigureOptions<RouteOptions>, RegexInlineRouteConstraintSetup>());
    return services;
}
```

여기까지입니다! 빌더 간의 차이점을 파악하기 위해 GitHub의 모든 코드와 Diff를 탐색하는 데 오랜 시간이 걸렸기 때문에 이 글의 후반부에서는 대부분 제가 찾은 내용을 문서화하는 데만 집중했습니다. 이 글의 전반부([또는 문서](https://learn.microsoft.com/en-gb/aspnet/core/fundamentals/native-aot?view=aspnetcore-8.0#the-createslimbuilder-method))에서 설명한 큰 수준의 차이점만 이해했다면 이 글의 후반부도 충분히 이해할 수 있습니다!

## 요약

이번 포스팅에서는 .NET 8 프리뷰에서 AOT 호환 `api` 템플릿을 지원하기 위해 도입된 `WebApplication.CreateSlimBuilder()` 메서드에 대해 살펴보았습니다. 새로운 메서드가 필요한 이유와 이 메서드를 사용할 때 기존 `WebApplication.CreateBuilder()` 메서드와 비교했을 때 어떤 차이가 있는지 살펴봤습니다. 글의 후반부에서는 이러한 변경이 내부적으로 어떻게 이루어졌는지 이해하기 위해 GitHub의 모든 실제 코드 차이점을 자세히 살펴봤습니다.