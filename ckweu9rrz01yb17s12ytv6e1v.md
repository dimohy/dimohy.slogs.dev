## .NET Core 탐색 2부 - WebApplicationBuilder와 일반 호스트 비교

***본 시리즈는 Andrew Lock 님의 [.NET Core 6 탐색](https://andrewlock.net/series/exploring-dotnet-6/) 연재를 번역한 것입니다.***

`WebApplication.CreateBuilder()`를 이용해서 .NET에서 애플리케이션을 빌드하는 새로운 "기본" 방식이 있습니다. 이 글에서는 이 접근 방식과 이전의 접근 방식을 비교해서 변경 이유를 논의하고 영향을 살펴봅니다. 다음 글에서는 `WebApplication`과 `WebApplicationBuilder` 안쪽 코드를 살펴보고 작동 방식을 확인하겠습니다.

## ASP.NET Core 애플리케이션 빌드: 역사 수업

.NET 6을 살펴보기 전에 초기 설계가 오늘날 막대한 영향을 끼쳤기 때문에 ASP.NET Core 앱의 "부트스트랩" 프로세스가 지난 몇 년 동안 어떻게 발전해 왔는지 살펴보는 것이 좋습니다. 다음 글에서 `WebApplicationBuilder` 안쪽 코드를 살펴 보면 더욱 분명해 질 것입니다!

.NET Core 1.x (현재 완전히 지원되지 않음)를 무시하더라도 ASP.NET Core 애플리케이션을 구성하기 위한 세가지 패러다임이 있습니다.

- `WebHost.CreateDefaultBuilder()`: ASP.NET Core 2.x에서 ASP.NET Core 앱을 구성하는 "원래" 접근 방식입니다.
- `Host.CreateDefaultBuilder()`: 일반 호스트 위에 ASP.NET Core를 다시 빌드하여 작업자 서비스와 같은 다른 워크로드를 지원합니다. .NET Core 3.x 및 .NET 5의 기본 접근 방식입니다.
- `WebApplication.CreateBuilder()`: .NET 6의 새로운 따끈한 방식입니다.

차이점을 더 잘 느끼기 위해 다음 섹션에서 일반적인 "시작" 코드를 재현했습니다. 그러면 .NET 6의 변경 사항이 좀 더 명확해집니다.


### ASP.NET Core 2.x: WebHost.CreateDefaultBuilder()

ASP.NET Core 1.x의 첫 번째 버젼에서는 (제 기억이 맞다면) "기본" 호스트에 대한 개념이 없었습니다. ASP.NET Core의 이데올로기 중 하나는 모든 것이 "대가 지불"이어야 한다는 것입니다. 즉, 사용할 필요가 없으면 기능에 대해 비용을 지불해서는 안됩니다.

실제로 이는 "시작하기" 템플릿에 많은 상용구와 많은 NuGet 패키지가 포함되어 있음을 의미합니다. 시작하기 위해 모든 코드를 보는 어려움을 대응하기 위해 ASP.NET Core는 `WebHost.CreateDefaultBuilder()`를 도입 했습니다. 이렇게 하면 `IWebHostBuilder`를 생성하고 `IWebHost`를 빌드 하는 전체 기본 로드가 설정 됩니다.

> [2017년에 `WebHost.CreateDefaultBuilder()`의 코드를 살펴보고 ASP.NET Core 1.x와 비교](https://andrewlock.net/exploring-program-and-startup-in-asp-net-core-2-preview1-2/)했습니다. 혹시 기억의 길을 따라 내려가는 것 같은 느낌이 들 경우를 대비해서 입니다.

처음부터 ASP.NET Core는 "애플리케이션" 부트스트랩을 분리했습니다. 역사적으로 이것은 일반적으로 Program.cs와 Startup.cs라고 하는 두 파일 간에 시작 코드를  분리하는 것으로 나타납니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1637813363552/KGex0F_oh.png)
프로그램 및 시작에 대한 구성 범위의 차이입니다. 프로그램은 일반적으로 프로젝트 수명 동안 안정적으로 유지되는 인프라 구성과 관련이 있습니다. 대조적으로, 새로운 기능을 추가하고 애플리케이션 동작을 업데이트하기 위해 Startup을 수정하는 경우가 많습니다. 내 책인 [ASP.NET Core in Action, Second Edtion](https://www.manning.com/books/asp-net-core-in-action-second-edition?utm_source=aspnetcore-in-action&utm_medium=affiliate&utm_campaign=book_lock2_asp_5_19_20&a_aid=aspnetcore-in-action&a_bid=44c089ee)에서 가져옴

ASP.NET Core 2.1의 Program.cs에서는 `WebHost.CreateDefaultBuilder()`를 호출해서 애플리케이션 구성(예: appsettings.json 로딩) 로깅을 설정하고 Kestrel 및 IIS 통합을 구성 합니다.

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        BuildWebHost(args).Run();
    }

    public static IWebHost BuildWebHost(string[] args) =>
        WebHost.CreateDefaultBuilder(args)
            .UseStartup<Startup>()
            .Build();
}
```

기본 템플릿은 Startup 클래스도 참조 합니다. 이 클래스는 인터페이스를 명시적으로 구현하지 않습니다. 오히려 `IWebHostBuilder` 구현은 종속성 주입 컨테이너와 미들웨어 파이프라인을 각각 설정하기 위해 `ConfigureServices()` 및 `Configure()` 메서드를 찾는 것을 알고 있습니다.

```csharp
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddMvc();
    }

    // 이 메소드는 런타임에 의해 호출됩니다. 이 방법을 사용해서 HTTP 요청 파이프라인을 구성합니다.
    public void Configure(IApplicationBuilder app, IHostingEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseStaticFiles();
        app.UseMvc(routes =>
        {
            routes.MapRoute(
                name: "default",
                template: "{controller=Home}/{action=Index}/{id?}");
        });
    }
}
```

위의  `Startup` 클래스에서 컨테이너에 MVC 서비스를 추가하고 예외 처리 및 정적 파일 미들웨어를 추가한 다음 MVC 미들웨어를 추가했습니다. MVC 미들웨어는 서버에서 렌더링된 뷰와 RESTful API 엔드포인트를 모두 수용하여 처음에 애플리케이션을 구축하는 유일한 실제적인 방법이였습니다.


### ASP.NET Core 3.x/5: 일반 HostBuilder

ASP.NET Core 3.x는 ASP.NET Core의 시작 코드에 몇 가지 큰 변화를 가져왔습니다. 이전에는 ASP.NET Core를 웹/HTTP 워크로드에만 사용할 수 있었지만 .NET Core 3.x에서는 장기적으로 동작하는 "작업자 서비스" (예를들어 메시지 대기열 사용), gRPC 서비스, 윈도 서비스 등 다른 접근 방식을 지원하기 위해 이동했습니다. 목표는 웹 앱(구성, 로깅, DI) 구축을 위해 특별히 구성된 기본 프레임워크를 다른 앱 유형과 공유하는 것이였습니다.

결과는 "(일반 호스트(https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host?view=aspnetcore-3.0)" ([Web Host](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/web-host?view=aspnetcore-3.0)와 반대)의 생산성과 그 위에 ASP.NET Core 스택을 "다시 플랫폼화" 하였습니다. `IHostBuilder`가 `IWebHostBuilder`를 대신 했습니다.

> 다시 말하지만 관심이 있는 경우 이 [마이그레이션에 대한 동시대의 시리즈](https://andrewlock.net/exploring-the-new-project-file-program-and-the-generic-host-in-asp-net-core-3/)가 있습니다.

이 변경으로 인해 몇 가지 불가피한 변경 사항이 발생했지만 ASP.NET 팀은 `IHostBuilder`가 아닌 `IWebHostBuilder`에 대해 작성된 모든 코드에 대한 경로를 제공하기 위해 최선을 다했습니다. 이러한 해결 방법 중 하나는 Program.cs 템플릿에서 기본적으로 사용되는 `ConfigureWebHostDefaults()` 메서드 입니다.

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }
    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            }; 
    }
}
```

ASP.NET Core 앱의 Startup 클래스를 등록하기 위해 `ConfigureWebHostDefaults`가 필요하다는 것은 `IHostBuilder`로의 마이그레이션 경로를 제공하는 데 있어 .NET 팀의 과제 중 하나를 보여줍니다. `Configure()` 메서드는 미들웨어를 구성하는 것이므로 시작은 웹 앱과 뗄 수 없는 관계입니다. 그러나 작업자 서비스 및 기타 많은 앱에는 미들웨어가 없으므로 Startup 클래스가 "일반 호스트" 수준 개념이 되는 것은 이치에 맞지 않습니다.

ASP.NET Core 3.x의 또 다른 큰 변화는 끝점 라우팅의 도입입니다.  끝점 라우팅은 이전에 ASP.NET Core의 MVC 부분(이 경우 라우팅 개념)으로 제한되었던 개념을 사용할 수 있도록 하려는 첫 번째 시도 중 하나였습니다. 이를 위해서 미들웨어 파이프라인에 대한 재고가 필요했지만 대부분의 경우 필요한 변경 사항은 최소화되었습니다.

> [엔드포인트 라우팅을 사용하도록 미들웨어를 변환하는 방법을 포함하여 이전에 엔드포인트 라우팅에 대해 더 깊이 있는 게시물을 작성했습니다.](https://andrewlock.net/converting-a-terminal-middleware-to-endpoint-routing-in-aspnetcore-3/#the-evolution-of-routing)

이러한 변경에도 불구하고 ASP.NET Core 3.x의 `Startup` 클래스는 2.x 버전과 매우 유사해 보였습니다. 아래 예제는 2.x 버젼과 거의 동일합니다. (MVC 대신 Razor 페이지로 변경했음에도 불구하고)

```csharp
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddRazorPages();
    }

    // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseStaticFiles();

        app.UseRouting();
        app.UseAuthorization();
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapRazorPages();
        });
    }
}
```

ASP.NET Core 5는 3.x에서 5로 업그레이드가 일반적으로 대상 프레임워크를 변경하고 일부 NuGet 패키지를 업데이트하는 것처럼 간단하도록 기존 애플리케이션에 큰 변화를 가져오지 않았습니다.

.NET 6의 경우 기존 애플리케이션을 업그레이드하는 경우에도 여전히 유효할 것입니다. 그러나 새 앱의 경우 기본 부트스트랩 환경이 완전히 변경되었습니다...


### ASP.NET Core 6: WebApplicationBuilder

이전 모드 버젼의 ASP.NET Core에는 2개의 파일에 분할 구성이 있습니다. .NET 6에서 C#, BCL 및 ASP.NET Core에 대한 많은 변경 사항으로 인해 이제 모든 것이 단일 파일에 포함될 수 있음을 의미합니다.

> 이 스타일을 강요하는 것은 아무것도 없습니다. ASP.NET Core 3.x/5 코드에서 보여준 모든 코드는 여전히 .NET 6에서 작동합니다!

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddRazorPages();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}

app.UseStaticFiles();

app.MapGet("/", () => "Hello World!");
app.MapRazorPages();

app.Run();
```

여기에는 많은 변경 사항이 있지만 가장 눈에 띄는 몇 가지는 다음과 같습니다.
- [최상위 문](https://docs.microsoft.com/en-us/dotnet/core/tutorials/top-level-templates)은 Program.Main() 상용구가 없음을 의미합니다.
- 암시적 using 지시문은 using 문이 필요하지 않음을 의미합니다. 이전 버젼의 조각에는 포함되지 않았지만 .NET 6에는 필요하지 않습니다!
- Startup 클래스 없음 - 모든 것이 하나의 파일에 있습니다.

분명히 훨씬 적은 코드지만 이것이 필요한가요? 그냥 휘둘리기 위한 휘둘림인가요? 그리고 어떻게 작동합니까?


## 모든 코드는 어디로 갔습니까?

.NET 6의 가장 큰 초점 중 하나는 "신규 사용자" 관점 이였습니다. ASP.NET Core 초보자는 정말 빨리 이해해야 하는 개념이 많습니다. [내 책의 목차](https://www.manning.com/books/asp-net-core-in-action-second-edition?utm_source=aspnetcore-in-action&utm_medium=affiliate&utm_campaign=book_lock2_asp_5_19_20&a_aid=aspnetcore-in-action&a_bid=44c089ee)를 보십시오. 머리를 굴릴 일이 많이 있습니다! 

.NET 6의 변경 사항은 시작과 관련된 "의례적인" 것을 제거하고 신규 사용자에게 혼동을 줄 수 있는 개념을 숨기는 데 중점을 두고 있습니다. 예를 들어:
- 시작할 때 `using`문은 필요하지 않습니다. 도구를 사용하면 일반적으로 실제 이런 문제를 발생하지 않지만, 시작할 때는 분명히 필요한 개념입니다.
- 이와 유사하게 네임스페이스는 시작할 때 불필요한 개념입니다.
- `Program.Main()`... 이름이 왜 이렇게 불리죠? 왜 필요하나요? 왜냐하면 당신이 그렇게 하기 때문에. 지금은 그럴 필요가 없습니다.
- 구성은 Program.cs와 Startup.cs 두 파일로 분리되지 않습니다. 나는 그 "관심의 분리"를 좋아하지만 왜 분리가 신규 사용자에게 그러한 방식인지 설명하는 것을 놓치지 않을 것입니다.
- Startup에 대해 이야기하는 동안 인터페이스를 명시적으로 구현하지 않더라도 호출할 수 있는 "마법" 메서드를 더이상 설명할 필요가 없습니다.

추가적으로 `WebApplication`과 `WebApplicationBuilder` 유형이 있습니다. 이런 유형은 위 목표를 달성하는 데 꼭 필요한 것은 아니지만 다소 `깨끗한" 구성 경험을 제공합니다.


## 새로운 유형이 정말로 필요합니까?

글쎄, 아니요. 우리는 그것을 필요로 하지 않습니다. 대신 일반 호스트를 사용해서 위 샘플과 유사한 .NET 6 앱을 작성할 수 있습니다.

```csharp
var hostBuilder = Host.CreateDefaultBuilder(args)
    .ConfigureServices(services => 
    {
        services.AddRazorPages();
    })
    .ConfigureWebHostDefaults(webBuilder =>
    {
        webBuilder.Configure((ctx, app) => 
        {
            if (ctx.HostingEnvironment.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseStaticFiles();
            app.UseRouting();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapGet("/", () => "Hello World!");
                endpoints.MapRazorPages();
            });
        });
    }); 

hostBuilder.Build().Run();
```

.NET 6 `WebApplication` 버젼보다 훨씬 더 복잡해 보이는데 동의한다고 생각합니다. 중첩된 람다가 많으므로 구성에 액세스 할 수 있도록 올바른 오버로드를 확보해야 합니다. (예를들어), 일반적으로 말하자면 절차적 부트스트래핑 스크립트를 (대부분) 더 복잡한 것으로 바꿉니다.

> `WebApplicationBuilder`의 또 다른 이점은 시작 시 비동기 코드가 훨씬 간단하다는 것입니다. 원할 때마다 비동기 메서드를 호출할 수 있습니다. 그렇게 하면 [ASP.NET Core 3.x/5에서 이 작업을 수행하기 위해 작성한 이 시리즈](https://andrewlock.net/running-async-tasks-on-app-startup-in-asp-net-core-part-1/)가 더 이상 사용되지 않을 것입니다!

`WebApplicationBuilder` 및 `WebApplication`의 깔끔한 점은 위의 일반 호스트 설정과 본질적으로 동일하지만 틀림없이 더 간단한 API를 사용한다는 점입니다.


## 대부분의 구성은 WebApplicationBuilder에서 발생합니다.

`WebApplicationBuilder`부터 살펴봅시다.

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddRazorPages();
```

`WebApplicationBuilder`는 4가지 주요 작업을 담당합니다.
- `builder.Configuration`을 사용해서 구성 추가
- `builder.Serives`를 사용해서 서비스 추가
- `builder.Logging`를 사용해서 로깅 구성
- 일반 `IHostBuilder` 및 `IWebHostBuilder` 구성

각각 차례대로 가져가면...

`WebApplicationBuilder`는 [이전 게시물에서 설명한 대로](https://andrewlock.net/exploring-dotnet-6-part-1-looking-inside-configurationmanager-in-dotnet-6/) 새 구성 소스를 추가하고 구성 값에 액세스 하기 위한 `ConfigurationManager` 유형을 노출합니다.

또한 DI 컨테이너에 서비스를 추가하기 위해 `IServiceCollection`을 직접 노출합니다. 따라서 일반 호스트를 사용하면

```csharp
var hostBuilder = Host.CreateDefaultBuilder(args);
hostBuilder.ConfigureServices(services => 
    {
        services.AddRazorPages();
        services.AddSingleton<MyThingy>();
    })
```

던 것을 `WebApplicationBuilder`를 통해

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddRazorPages();
builder.Services.AddSingleton<MyThingy>();
```
로 할 수 있습니다. 마찬가지로 로깅을 위해 다음 대신

```csharp
var hostBuilder = Host.CreateDefaultBuilder(args);
hostBuilder.ConfigureLogging(builder => 
    {
        builder.AddFile();
    })
```

다음 처럼

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Logging.AddFile();
```
 할 수 있습니다.

이것은 사용하기 쉬운 API에서 정확히 동일한 동작을 합니다. `IHostBuilder` 또는 `IWebHostBuilder`에 직접 의존하는 확장 지점의 경우 `WebApplicationBuilder`는 각각 `Host`및 `WebHost` 속성을 노출합니다.

예를 들어 [Serilog의 ASP.NET Core 통합](https://github.com/serilog/serilog-aspnetcore)은 IHostBuilder에 연결되므로 ASP.NET Core 3.x/5에서는 다음 사용하여 추가합니다.

```csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .UseSerilog() // <-- Add this line
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        });
```

`WebApplicationBuilder`를 사용하면 빌더 자체 대신 `Host` 속성에서 `UseSerilog()` 호출을 수행합니다.

```csharp
builder.Host.UseSerilog();
```

사실 WebApplicationBuilder는 미들웨어 파이프라인을 제외한 모든 구성을 수행하는 곳입니다.


## WebApplication은 많은 모자를 쓰고 있습니다.

WebApplicationBuilder에서 필요한 모든 것을 구성했으면 `Build()`를 호출해서 `WebApplication`의 인스턴스를 생성합니다.

```csharp
var app = builder.Build();
```

`WebApplication`은 여러 다른 인터페이스를 구현하므로 흥미롭습니다.
- `IHost` - 호스트를 시작하고 중지하는데 사용
- `IApplicationBuilder` - 미들웨어 파이프라인을 구성하는데 사용
- `IEndpointRouteBuilder` - 끝점을 추가하는데 사용

후자의 두 가지 점은 매우 관련이 있습니다. ASP.NET Core 3.x 및 5에서 `IEndpointRouteBuilder`는 `UseEndpoints()`를 호출하고 람다를 전달하여 끝점을 추가하는 데 사용됩니다. 예를 들면:

```csharp
public void Configure(IApplicationBuilder app)
{
    app.UseStaticFiles();
    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapRazorPages();
    });
}
```

ASP.NET Core를 처음 사용하는 사람에겐 .NET 3.x/5 패턴에 몇 가지 복잡성이 있습니다.
- 미들웨어 파이프라인 구축은 Startup의 `Configure()` 함수에서 발생합니다.
- `app.UseEndpoints()` 전에 `app.UseRouting()`을 호출해야 합니다. (또한 다른 미들웨어를 올바른 위치에 배치해야 함)
- 끝점을 구성하려면 람다를 사용해야 합니다. (C#에 익숙한 사용자에게는 복잡하지 않지만 초보자에게는 혼란스러울 수 있음)

`WebApplication`은 이 패턴을 크게 단순화 함:
```csharp
app.UseStaticFiles();
app.MapRazorPages();
```

미들웨어와 끝점 간의 구분이 .NET 5.x 등에서 보다 훨씬 명확하지 않기 때문에 약간 혼란스럽지만 이것은 분명히 훨씬 간단합니다. 그것은 아마도 맛보기일 뿐이지만 나는 그것이 "순서의 중요함"이라는 메시지를 흐리게 한다고 생각합니다. (미들웨어에는 적용되지만 일반적으로 끝점에서는 적용되지 않음)


## 요약

이 글에서는 ASP.NET Core 앱의 부트스트랩이 버젼 2.x에서 .NET 6까지 어떻게 변화되었는지 설명헀습니다. .NET 6에 도입된 새로운 `WebApplication` 및 `WebApplicationBuilder` 유형을 보여주고 도입된 이유를 논의하고, 그리고 그들이 가져오는 몇가지 이점, 마지막으로 두 클래스가 수행하는 다양한 역할과 API가 더 간단한 시작 환경을 만드는 방법에 대해 설명 하였습니다. 다음 글에서는 어떻게 작동하는지 보기 위해 유형 뒤의 일부 코드를 살펴보겠습니다.


## 원문

[Part 2 - Comparing WebApplicationBuilder to the Generic Host](https://andrewlock.net/exploring-dotnet-6-part-2-comparing-webapplicationbuilder-to-the-generic-host/) | Andrew Lock