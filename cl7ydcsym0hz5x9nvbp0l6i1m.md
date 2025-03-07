---
title: "Fusion 튜토리얼 - 11부: Fusion에서 인증"
datePublished: Mon Sep 12 2022 06:13:24 GMT+0000 (Coordinated Universal Time)
cuid: cl7ydcsym0hz5x9nvbp0l6i1m
slug: fusion-11-fusion
tags: dotnet, stlfusion

---

# 11부: Fusion에서 인증


## Fusion 세션

이 인증 시스템의 중요한 요소 중 하나는 Fusion의 자체 세션입니다. 세션은 본질적으로 HTTP 전용 쿠키에 저장되는 문자열 값입니다. 클라이언트가 요청과 함께 이 쿠키를 보내면 거기에 지정된 세션을 사용합니다. 그렇지 않은 경우 `SessionMiddleware`가 생성합니다.

Fusion 세션을 활성화하려면 `Startup` 클래스의 `Configure` 메서드 내에서 `UseFusionSession`을 호출해야 합니다. 그러면 요청 파이프라인에 `SessionMiddleware`가 추가됩니다. 실제 클래스에는 좀 더 많은 논리가 포함되어 있지만 현재 중요한 부분은 다음과 같습니다.

```csharp
public async Task InvokeAsync(HttpContext httpContext, RequestDelegate next)
    {
        // Note that now it's slightly more complex due to
        // newly introduced multitenancy support in Fusion 3.x.
        // But you'll get the idea.

        var cookies = httpContext.Request.Cookies;
        var cookieName = Cookie.Name ?? "";
        cookies.TryGetValue(cookieName, out var sessionId);
        var session = string.IsNullOrEmpty(sessionId) ? null : new Session(sessionId);

        if (session == null) {
            session = SessionFactory.CreateSession();
            var responseCookies = httpContext.Response.Cookies;
            responseCookies.Append(cookieName, session.Id, Cookie.Build(httpContext));
        }
        SessionProvider.Session = session;
        await next(httpContext).ConfigureAwait(false);
    }
```

`Session` 클래스 자체는 매우 간단하며 단일 `Symbol Id` 값을 저장합니다. `Symbol`은 캐시된 `HashCode`와 함께 문자열을 저장하는 구조체이며, 유일한 역할은 사용 시 사전 조회 속도를 높이는 것입니다. 그 외에도 `Session`은 `Id`로 비교되는 동등성을 재정의합니다.

```csharp
public sealed class Session : IHasId<Symbol>, IEquatable<Session>,
    IConvertibleTo<string>, IConvertibleTo<Symbol>
{
    public static Session Null { get; } = null!; // To gracefully bypass some nullability checks
    public static Session Default { get; } = new("~"); // We'll cover this later

    [DataMember(Order = 0)]
    public Symbol Id { get; }
    ...
}
```

`fusion.AddAuthentication()`을 호출하면 종속성 주입 컨테이너에 등록된 여러 서비스와 가장 중요한 서비스는 다음과 같습니다.

```csharp
Services.TryAddSingleton<ISessionFactory, SessionFactory>();
Services.TryAddScoped<ISessionProvider, SessionProvider>();
Services.TryAddTransient(c => (ISessionResolver) c.GetRequiredService<ISessionProvider>());
Services.TryAddTransient(c => c.GetRequiredService<ISessionProvider>().Session);
```

다음은 이러한 서비스에 대해 알아야 할 사항입니다.

- `ISessionFactory`는 새 세션을 생성합니다. 예를 들어 재정의할 수 있습니다. 모든 세션을 디지털 서명으로 만드십시오.
- `ISessionProvider`는 현재 세션을 추적합니다. `ISessionResolver`를 구현합니다.
- `ISessionResolver`는 현재 세션을 가져올 수 있습니다.
- 마지막으로 `Session`은 일회성 서비스로 등록되고 `ISessionResolver` : `c => c.GetRequiredService<ISessionProvider>().Session`에 의해 준비된 세션에 매핑됩니다.

나중에 Blazor 앱에서 사용하는 방법을 다룰 것입니다. 지금은 존재한다는 것만 기억합시다.


## 백엔드 애플리케이션의 인증 서비스

`Session`의 역할은 ASP.NET 세션과 매우 유사합니다. 현재 사용자와 관련된 모든 것을 식별할 수 있습니다. 기술적으로 무엇을 연결할지는 사용자에게 달려 있지만 Fusion의 기본 제공 서비스는 인증 정보라는 단일 종류의 정보를 처리합니다.

세션이 인증되면 사용자 정보와 사용자와 관련된 클레임 등을 가져올 수 있습니다. 서버 측에서 다음 Fusion 서비스는 인증 데이터와 상호 작용합니다.

- `InMemoryAuthService`
- `DbAuthService<...>`

동일한 인터페이스를 구현하므로 상호 교환 가능하게 사용할 수 있습니다. 둘 사이의 유일한 차이점은 데이터를 저장하는 위치(데이터베이스의 메모리)입니다. `InMemoryAuthService`는 주로 디버깅 또는 빠른 프로토타이핑을 위해 존재합니다. 실제 앱에서는 사용하고 싶지 않을 것입니다.

인터페이스에 대해 말하자면 이러한 서비스는 `IAuth` 및 `IAuthBackend`를 구현합니다. 첫 번째 것은 클라이언트에서 사용하기 위한 것입니다. 두 번째 것은 서버 측에서 사용해야 합니다.

그것들이 어떻게 보이는지 여기에서 확인할 수 있습니다.

- https://github.com/servicetitan/Stl.Fusion/blob/master/src/Stl.Fusion/Authentication/IAuth.cs

주요 차이점은 다음과 같습니다.

- `IAuth`는 현재 세션과 관련된 데이터만 읽을 수 있습니다.
- `IAuthBackend`를 사용하면 이를 수정하고 모든 사용자에 대한 정보를 읽을 수 있습니다.

다음은 Fusion 서비스를 설계하는 데 권장되는 방법입니다.

- `IXxx`는 프론트 엔드이며 첫 번째 매개 변수로 `Session`을 가져오고 현재 사용자가 액세스할 수 있는 데이터만 제공합니다.
- `IXxxBackend`는 세션이 필요하지 않으며 모든 것에 액세스할 수 있습니다.

인증을 추가하면 기본적으로 `InMemoryAuthService`가 `IAuth` 및 `IAuthBackend` 구현으로 등록됩니다. DI 컨테이너에 `DbAuthService`를 등록하려면 다음 코드 조각과 유사한 방식으로 `AddAuthentication` 메서드를 호출해야 합니다.

Operations Framework는 이러한 서비스에도 필요합니다. 이를 다루는 [10부]를 읽어보시기 바랍니다.

```csharp
services.AddDbContextServices<FusionDbContext>(dbContext => {
    db.AddOperations(operations => {
        operations.ConfigureOperationLogReader(_ => new() {
            UnconditionalCheckPeriod = TimeSpan.FromSeconds(10).ToRandom(0.05),
        });
        operations.AddFileBasedOperationLogChangeTracking();
    });
    dbContext.AddAuthentication<long>();
});
```

`DbContext`는 여기에 유형 매개변수로 제공된 클래스에 대한 `DbSet`-s를 포함해야 합니다. `DbSessionInfo` 및 `DbUser` 클래스는 인증 데이터를 저장하기 위해 Fusion에서 제공하는 매우 간단한 엔터티입니다.

```csharp
public class AppDbContext : DbContextBase
{
    // Authentication-related tables
    public DbSet<DbUser<long>> Users { get; protected set; } = null!;
    public DbSet<DbUserIdentity<long>> UserIdentities { get; protected set; } = null!;
    public DbSet<DbSessionInfo<long>> Sessions { get; protected set; } = null!;
    // Operations Framework's operation log
    public DbSet<DbOperation> Operations { get; protected set; } = null!;

    public AppDbContext(DbContextOptions options) : base(options) { }
}
```

이러한 엔터티 타입은 다음과 같습니다.

```csharp
public class DbSessionInfo<TDbUserId> : IHasId<string>, IHasVersion<long>
{
    [Key] [StringLength(32)] public string Id { get; set; }
    [ConcurrencyCheck] public long Version { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime LastSeenAt { get; set; }
    public string IPAddress { get; set; }
    public string UserAgent { get; set; }
    public string AuthenticatedIdentity { get; set; }
    public TDbUserId UserId { get; set; }
    public bool IsSignOutForced { get; set; }
    public string OptionsJson { get; set; }
}
```

`DbSessionInfo`는 세션을 저장하고 이러한 세션(인증된 경우)은 `DbUser`와 연관될 수 있습니다.

```csharp
public class DbUser<TDbUserId> : IHasId<TDbUserId>, IHasVersion<long> where TDbUserId : notnull
{
    public DbUser();

    [Key] public TDbUserId Id { get; set; }
    [ConcurrencyCheck] public long Version { get; set; }
    [MinLength(3)] public string Name { get; set; }
    public string ClaimsJson { get; set; }
    public List<DbUserIdentity<TDbUserId>> Identities { get; }
    
    [JsonIgnore, NotMapped]
    public ImmutableDictionary<string, string> Claims { get; set; }
}
```


### 권한 부여를 위해 컴퓨팅 서비스에서 세션 사용

우리의 컴퓨팅 서비스는 우리가 인증되었는지 여부와 로그인한 사용자가 누구인지 결정하는 데 사용할 수 있는 `Session` 개체를 수신할 수 있습니다.

```csharp
[ComputeMethod]
public virtual async Task<List<OrderHeaderDto>> GetMyOrders(Session session, CancellationToken cancellationToken = default)
{
    // We assume that _auth is of IAuth type here.
    var sessionInfo = await _auth.GetSessionInfo(session, cancellationToken);
    // You can use any of such methods
    var user = await _authService.RequireUser(session, true, CancellationToken);

    await using var dbContext = CreateDbContext();

    if (user.IsAuthenticated && user.Claims.ContainsKey("read_orders")) {
        // Read orders
    }
```

여기에서 `RequireUser`는 `GetUser`를 호출하고 이 호출의 결과가 `null`이면 오류를 발생시킵니다. 전달된 `true` 인수는 `ArgumentNullException`을 `ResultException`으로 래핑해야 함을 나타냅니다. `ResultException`은 Fusion에서 "정상" 결과로 간주하므로 1초 내에 이 결과를 자동 무효화하지 않습니다(이는 기본적으로 컴퓨팅 메서드에서 발생한 다른 예외에 대해 발생합니다.
Fusion은 이러한 오류가 일시적일 수 있다고 가정합니다). 여기에서 이 동작에 대해 자세히 읽을 수 있습니다.

https://discord.com/channels/729970863419424788/729971920476307554/995865256201027614

`GetSessionInfo`, `GetUser` 및 기타 모든 `IAuth` 및 `IAuthBackend` 메서드는 컴퓨팅 메서드입니다. 즉, 제공된 세션에 로그인하거나 로그아웃하면 GetMyOrders 호출의 결과가 무효화됩니다. 일반적으로 결과에 영향을 미치는 변경이 발생할 때마다입니다.


### Fusion 및 ASP.NET Core 인증 상태 동기화

`IAuth` 및 `IAuthBackend` API를 보면 인증 자체가 없다는 결론을 내리기가 쉽습니다.

- `IAuth`를 사용하면 인증 상태를 검색할 수 있습니다. 즉, 세션과 관련된 `SessionInfo`, `User` 및 세션 옵션(`ImmutableOptionSet`으로 표시되는 키-값 쌍) 가져오기
- 반대로 `IAuthBackend`는 이를 설정할 수 있습니다.

따라서 실제로 이러한 API는 인증 상태를 유지합니다. 다른 것을 사용하여 사용자를 인증하고 "Fusion 세상"에서 이러한 서비스를 사용하여 인증 정보에 액세스한다고 가정합니다. 이들은 컴퓨팅 서비스이므로 인증 정보가 변경되면 이를 호출하는 컴퓨팅 서비스가 결과를 무효화하도록 합니다.

ASP.NET Core와 Fusion 간에 인증 상태를 동기화하는 제안된 방법은 이 논리를 `Host.cshtml`에 포함하는 것입니다. 일반적으로 Blazor 앱의 매핑되지 않은 모든 경로에 매핑되며 로드될 때 바로 ASP.NET Core에서 Fusion으로 인증 상태를 전파합니다. 여기서는 사용자가 로그인하거나 로그아웃할 때 이러한 흐름이 끝날 때 `Host.cshtml`이 로드되므로 동기화하기에 가장 좋은 위치라고 가정합니다.

동기화는 `ServerAuthHelper.UpdateAuthState` 메서드에 의해 수행됩니다. `ServerAuthHelper`는 위에서 설명한 것과 정확히 일치하는 내장 Fusion 도우미입니다. 현재 `Session`에 대해 `IAuth`에 의해 노출된 인증 상태와 `HttpContext`에 노출된 상태를 비교하고 상태를 동기화하기 위해 `IAuthBackend.SignIn()` / `IAuthBackend.SignOut`을 호출합니다.

`ServerAuthHelper`의 소스 코드:

- https://github.com/servicetitan/Stl.Fusion/blob/master/src/Stl.Fusion.Server/Authentication/ServerAuthHelper.cs#L69

다음 코드 조각은 `Host.cshtml`에 포함하는 방법을 보여줍니다.

```razor
@page "/"
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
@namespace Templates.TodoApp.Host.Pages
@using Stl.Fusion.Blazor
@using Templates.TodoApp.UI
@using Stl.Fusion.Server.Authentication
@using Stl.Fusion.Server.Controllers
@inject ServerAuthHelper ServerAuthHelper
@inject BlazorCircuitContext BlazorCircuitContext
@{
    await ServerAuthHelper.UpdateAuthState(HttpContext);
    var authSchemas = await ServerAuthHelper.GetSchemas(HttpContext);
    var sessionId = ServerAuthHelper.Session.Id.Value;
    var isServerSideBlazor = BlazorModeController.IsServerSideBlazor(HttpContext);
    var isCloseWindowRequest = ServerAuthHelper.IsCloseWindowRequest(HttpContext, out var closeWindowFlowName);
    Layout = null;
}
<head>
    // This part has to be somewhere in <head> section
    <script src="_content/Stl.Fusion.Blazor/scripts/fusionAuth.js"></script>
    <script>
        window.FusionAuth.schemas = "@authSchemas";
    </script>
</head>
<body>
// And this part has to be somewhere in the beginning of <body> section
@if (isCloseWindowRequest) {
    <script>
        setTimeout(function () {
            window.close();
        }, 500)
    </script>
    <div class="alert alert-primary">
        @(closeWindowFlowName) completed, you can close this window.
    </div>
}    
```

인증 창을 열거나 리디렉션을 수행하는 `Stl.Fusion.Blazor` 어셈블리에 포함된 작은 스크립트인 `fusionAuth.js`가 있다고 가정합니다. 궁금하다면 소스 코드:

- https://github.com/servicetitan/Stl.Fusion/blob/master/src/Stl.Fusion.Blazor/wwwroot/scripts/fusionAuth.js

그 외에도 ASP.NET Core 앱 서비스 컨테이너 구성에 몇 가지 추가 사항을 추가해야 합니다.

```csharp
var fusion = services.AddFusion();
var fusionServer = fusion.AddWebServer();
var fusionAuth = fusion.AddAuthentication().AddServer(
    signInControllerOptionsFactory: _ => new() {
        // Set to the desired one
        DefaultScheme = MicrosoftAccountDefaults.AuthenticationScheme, 
        SignInPropertiesBuilder = (_, properties) => {
            properties.IsPersistent = true;
        }
    },
    serverAuthHelperOptionsFactory: _ => new() {
        // These are the claims mapped to User.Name once a new
        // User is created on sign-in; if they absent or this list
        // is empty, ClaimsPrincipal.Identity.Name is used.
        NameClaimKeys = Array.Empty<string>(),
    });

// You need this only if you plan to use Blazor WASM
var fusionClient = fusion.AddRestEaseClient();
// Configure Fusion client here

// Configure ASP.NET Core authentication providers:
services.AddAuthentication(options => {
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
}).AddCookie(options => {
    // You can use whatever you prefer to store the authentication info
    // in ASP.NET Core, this specific example uses a cookie.
    options.LoginPath = "/signIn"; // Mapped to 
    options.LogoutPath = "/signOut";
    if (Env.IsDevelopment())
        options.Cookie.SecurePolicy = CookieSecurePolicy.None;
    // This controls the expiration time stored in the cookie itself
    options.ExpireTimeSpan = TimeSpan.FromDays(7);
    options.SlidingExpiration = true;
    // And this controls when the browser forgets the cookie
    options.Events.OnSigningIn = ctx => {
        ctx.CookieOptions.Expires = DateTimeOffset.UtcNow.AddDays(28);
        return Task.CompletedTask;
    };
}).AddGitHub(options => {
    // Again, this is just an example of using GitHub account
    // OAuth provider to authenticate. There is nothing specific
    // to Fusion in the code below.
    options.ClientId = "...";
    options.ClientSecret = "..."
    options.Scope.Add("read:user");
    options.Scope.Add("user:email");
    options.CorrelationCookie.SameSite = SameSiteMode.Lax;
});
```

위의 `/signIn` 및 `/signOut` 경로를 사용한다는 점에 유의하세요. 이 경로는 이 컨트롤러에 매핑됩니다.

- https://github.com/servicetitan/Stl.Fusion/blob/master/src/Stl.Fusion.Server/Controllers/SignInController.cs

이러한 작업에 대해 다른 논리를 사용하려면 다른 컨트롤러의 유사한 작업에 매핑하고 경로를 업데이트하거나(+ JS에서도 `window.FusionAuth.signInPath` 및 `window.FusionAuth.signOutPath` 설정) 이 컨트롤러를 교체할 수 있습니다. 이를 위한 편리한 도우미가 있습니다: `services.AddFusion().AddServer().AddControllerFilter(...)`

마지막으로 앱 구성에 약간의 추가 기능이 필요합니다.

```csharp
// You need this only if you use Blazor WASM w/ Fusion client
app.UseWebSockets(new WebSocketOptions() {
    KeepAliveInterval = TimeSpan.FromSeconds(30),
});
app.UseFusionSession();

// Required by Blazor
app.UseBlazorFrameworkFiles(); 
// Required by Blazor + it serves embedded content, such as  `fusionAuth.js`
app.UseStaticFiles(); 

// API controllers
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization(); // ASP.NET Core authorization, use only if you need it
app.UseEndpoints(endpoints => {
    endpoints.MapBlazorHub();
    endpoints.MapFusionWebSocketServer(); // Needed only if you use Blazor WASM w/ Fusion client
    endpoints.MapControllers();
    endpoints.MapFallbackToPage("/_Host"); // Maps every unmapped route to _Host.cshtml
});
```


### Blazor WASM 구성 요소에서 Fusion 인증 사용

아시다시피 클라이언트 측 복제 서비스는 서버 측 컴퓨팅 서비스와 동일한 인터페이스를 가지고 있으므로 클라이언트는 `Session`을 필요로 하는 메소드에 대한 매개변수로 `Session`을 전달해야 합니다. 그러나 `Session`은 http 전용 쿠키에 저장되므로 클라이언트는 해당 값을 직접 읽을 수 없습니다. 이것은 의도적입니다. Session을 사용하면 누구든지 연결된 사용자로 가장할 수 있으므로 클라이언트 측에서 사용하지 않는 것이 가장 좋습니다.

Fusion은 소위 "기본 세션"을 사용하여 작동합니다. `Session` 클래스 코드의 시작 부분을 다시 인용하겠습니다.

```csharp
public sealed class Session : IHasId<Symbol>, IEquatable<Session>,
    IConvertibleTo<string>, IConvertibleTo<Symbol>
{
    public static Session Null { get; } = null!; // To gracefully bypass some nullability checks
    public static Session Default { get; } = new("~"); // Default session
    
    // ...
```

기본 세션은 `SessionModelBinder`에 의해 `ISessionResolver`가 제공하는 세션으로 자동 대체되는 특별히 명명된 `Session`입니다. 즉, `Session.Default`를 일부 컴퓨팅 서비스에 인수로 전달하면 서버 측에서 컨트롤러 메서드 호출 시 true 값을 얻게 됩니다.

또한 모든 `ISessionCommand`에 대해 동일한 작업을 수행하는 `UseDefaultSessionAttribute` (ASP.NET Core 필터)가 있습니다.

이 모든 것은 Blazor WASM 클라이언트가 작동하기 위해 실제 `Session` 값을 알 필요가 없다는 것을 의미합니다. 필요한 것은 `Session.Default`를 현재 세션으로 반환하도록 `ISessionResolver`를 구성하기만 하면 됩니다.

그리고 Blazor 구성 요소가 Blazor 서버에서 작동하도록 하려면 거기에서 사용할 수 있는 올바른 `Session`을 사용해야 합니다.

```csharp
Services.TryAddScoped<ISessionProvider, SessionProvider>();
Services.TryAddTransient(c => (ISessionResolver) c.GetRequiredService<ISessionProvider>());
Services.TryAddTransient(c => c.GetRequiredService<ISessionProvider>().Session);
```

따라서 `ISessionResolver`가 Blazor WASM 클라이언트에서 `Session.Default`를 확인하도록 하기만 하면 됩니다. 이를 수행하는 방법 중 하나는 이 `App.razor` (루트 Blazor 구성 요소)를 사용하는 것입니다.

```razor
@using Stl.OS
@implements IDisposable
@inject BlazorCircuitContext BlazorCircuitContext
@inject ISessionProvider SessionProvider

<CascadingAuthState>
    <Router AppAssembly="@typeof(Program).Assembly">
        <Found Context="routeData">
            <RouteView RouteData="@routeData" DefaultLayout="@typeof(MainLayout)"/>
        </Found>
        <NotFound>
            <LayoutView Layout="@typeof(MainLayout)">
                <p>Sorry, there's nothing at this address.</p>
            </LayoutView>
        </NotFound>
    </Router>
</CascadingAuthState>

@code {
    private Theme Theme { get; } = new() { IsGradient = true, IsRounded = false };

    [Parameter]
    public string SessionId { get; set; } = Session.Default.Id;

    protected override void OnInitialized()
    {
        SessionProvider.Session = OSInfo.IsWebAssembly 
            ? Session.Default 
            : new Session(SessionId);
        if (!BlazorCircuitContext.IsPrerendering)
            BlazorCircuitContext.RootComponent = this;
    }

    public void Dispose()
        => BlazorCircuitContext.Dispose();
}
```

이 구성 요소가 초기화될 때 Blazor WASM을 실행하지 않는 한 `SessionProvider.Session`을 매개 변수로 가져오는 값으로 설정하는 것을 볼 수 있습니다. 이 경우 `Session.Default`로 설정합니다. (`ISessionResolver` 또는 서비스 공급자를 통해) `Session`을 확인하려는 모든 시도는 이 값을 반환합니다.

`App.razor`가 해당 콘텐츠를 `CascadingAuthState`로 래핑하여 해당 `ChildContent`를 Blazor의 `<CascadingAuthenticationState>`에 포함하여 Blazor 인증이 예상대로 작동하도록 합니다. 세부 사항에 관심이 있는 경우 해당 출처를 참조하십시오.

- https://github.com/servicetitan/Stl.Fusion/blob/master/src/Stl.Fusion.Blazor/Authentication/CascadingAuthState.razor

이 모든 것은 서버 측에서 `App.razor`를 생성하기 위해 `_Host.cshtml`에 약간의 특별한 논리도 필요하다는 것을 의미합니다.

```razor
<app id="app">
    @{
        using var prerendering = BlazorCircuitContext.Prerendering();
        var prerenderedApp = await Html.RenderComponentAsync<App>(
            isServerSideBlazor ? RenderMode.ServerPrerendered : RenderMode.WebAssemblyPrerendered,
            isServerSideBlazor ? new { SessionId = sessionId } : null);
    }
    @(prerenderedApp)
</app>
```

여기서 가장 중요한 부분은 Blazor 서버가 사용되는 경우 `new { SessionId = sessionId }` 매개변수를 `Html.RenderComponentAsync<App>(...)` 호출에 전달하고 대신 `null`을 전달하는 것입니다.

이것은 또한 우리가 여기에서 `BlazorCircuitContext`를 사용하는 이유를 설명합니다. 이는 특히 Blazor 회로가 사전 렌더링 모드에서 실행되는지 여부를 감지할 수 있도록 하는 Fusion에 포함된 편리한 도우미입니다.

자, 이제 모든 준비가 완료되었으며 `IAuth`에 의존하는 첫 번째 Blazor 구성 요소를 작성할 준비가 되었습니다.

```razor
@page "/myOrders
@inherits ComputedStateComponent<List<OrderHeaderDto>>
@inject IOrderService OrderService
@inject IAuth Auth
@inject Session Session // We resolve the Session via DI container
@{
    var orders = State.Value;
}

// Rendering orders

@code {
    protected override async Task<List<OrderHeader>> ComputeState(CancellationToken cancellationToken)
    {
        var user = await Auth.RequireUser(Session, true, cancellationToken);
        var sessionInfo = await Auth.GetSessionInfo(Session, cancellationToken);

        if (!user.Claims.ContainsKey("required-claim"))
            return new List<OrderHeader>();

        return await OrderService.GetMyOrders(Session);
    }
}
```

Blazor WASM과 함께 작동하려면 다음과 같은 컨트롤러가 필요합니다.


```csharp
[Route("api/[controller]/[action]")]
[ApiController, UseDefaultSession] // <<< You need UseDefaultSession filter here!
public class OrderController : ControllerBase, IOrderService
{
    private readonly IOrderService _orderService;

    public OrderController(IOrderService orderService, ISessionResolver sessionResolver)
        => _orderService = orderService;

    [HttpGet, Publish]
    public async Task<List<OrderHeader>> GetMyOrders(Session session, CancellationToken cancellationToken = default)
        => await _orderService.GetMyOrders(Session, cancellationToken);
}
```


### 로그아웃

이미 알고 있듯이 `_Host.cshtml`이 요청되면 Fusion의 인증 상태가 동기화됩니다. 이는 거의 모든 요청에서 발생하므로 일반적인 로그아웃 흐름은 다음을 의미합니다.

- 먼저, 예를 들어 정기적인 로그아웃을 실행합니다. 브라우저를 `~/signOut` 페이지로 리디렉션
- 둘째, `_Host.cshtml`을 로드하는 일부 일반 페이지로 브라우저를 리디렉션합니다.

Fusion 인증 상태 변경은 모든 클라이언트에 즉시 적용되므로 예를 들어 다음에서 이 모든 작업을 수행할 수 있습니다. 별도의 창 - 동일한 세션을 공유하는 모든 브라우저 창이 로그아웃되도록 하는 데 충분합니다.

`ClientAuthHelper`는 `Stl.Fusion.Blazor`에 포함된 도우미로 `window.fusionAuth`에서 해당 메서드를 트리거하여 이러한 흐름을 실행하는 데 도움이 됩니다.

- https://github.com/servicetitan/Stl.Fusion/blob/master/src/Stl.Fusion.Blazor/Authentication/ClientAuthHelper.cs

`TodoApp` 템플릿의 `Authentication.razor` 페이지에서 사용하는 방법은 다음과 같습니다.

```razor
<Button Color="Color.Warning"
        @onclick="_ => ClientAuthHelper.SignOut()">Sign out</Button>
<Button Color="Color.Danger"
        @onclick="_ => ClientAuthHelper.SignOutEverywhere()">Sign out everywhere</Button>
```

그리고 궁금하다면 `SignOutEverywhere()`는 현재 사용자의 모든 세션을 로그아웃합니다. `IAuthBackend`에는 실제로 이러한 세션을 열거할 수 있는 메서드가 있기 때문에 가능합니다. 왜냐하면... 안될 이유가 없죠?


### ASP.NET Core Identity로 고유한 등록/로그인 시스템 만들기

ASP.NET Core Identity 위에 Fusion의 인증을 사용할 수 있습니다. 이 접근 방식을 따르면 두 프레임워크 간에 인증 상태를 동기화해야 합니다. 다음 코드는 이에 대한 매우 기본적인 구현을 보여줍니다.

`SignIn` 메서드는 로그인하려는 사용자의 사용자 이름과 비밀번호와 현재 Fusion 세션을 수신해야 합니다. 그런 다음 기본적으로 확인하는 예제에서 제공된 데이터가 유효한지 확인할 수 있습니다.

- 사용자가 존재하는지
- 사용자가 이미 Fusions 인증 상태에 로그인했는지
- 비밀번호/이메일 쌍이 올바른지

```csharp
public async Task SignIn(Session session, EmailPasswordDto signInDto, CancellationToken cancellationToken)
{
    var user = await _userManager.FindByNameAsync(signInDto.Email);
    if (user is ApplicationUser) {
        var sessionInfo = await _authService.GetSessionInfo(session, cancellationToken);
        if (sessionInfo.IsAuthenticated)
            throw new InvalidOperationException("You are already signed in!");

        var signInResult = await _signInManager.CheckPasswordSignInAsync(user, signInDto.Password, lockoutOnFailure: false);
    }
}
```

모든 것이 정확하면 사용자 로그인을 진행할 수 있습니다. 여기서 기본 아이디어는 Identity 프레임워크를 사용하여 각 사용자의 클레임과 역할을 데이터베이스 내부에 저장하고 로그인 프로세스 중에 이러한 역할을 쿼리한다는 것입니다. Identity가 제공하는 `UserManager` 서비스를 사용하여 여기에서 클레임을 가져오고 Fusion SignIn 메서드에 전달할 수 있는 이러한 값에서 `ClaimsPrincipal`을 만들 수 있습니다.

```csharp
if (signInResult.Succeeded) {
    var claims = await _userManager.GetClaimsAsync(user);
    var roles = await _userManager.GetRolesAsync(user);
    var identity = new ClaimsIdentity(claims, CookieAuthenticationDefaults.AuthenticationScheme);
    foreach (var role in roles)
        identity.AddClaim(new Claim(ClaimTypes.Role, role));

    identity.AddClaim(new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()));
    identity.AddClaim(new Claim(ClaimTypes.Name, user.UserName));
    var principal = new ClaimsPrincipal(identity);

    var ipAddress = _httpContextAccessor.HttpContext.Connection.RemoteIpAddress?.ToString() ?? "";
    var userAgent = _httpContextAccessor.HttpContext.Request.Headers.TryGetValue("User-Agent", out var userAgentValues)
                    ? userAgentValues.FirstOrDefault() ?? ""
                    : "";

    var mustUpdateSessionInfo =
        !StringComparer.Ordinal.Equals(sessionInfo.IPAddress, ipAddress)
        || !StringComparer.Ordinal.Equals(sessionInfo.UserAgent, userAgent);
    if (mustUpdateSessionInfo) {
        var setupSessionCommand = new SetupSessionCommand(session, ipAddress, userAgent);
        await _auth.SetupSession(setupSessionCommand, cancellationToken);
    }

    var fusionUser = new User(session.Id);
    var (newUser, authenticatedIdentity) = CreateFusionUser(fusionUser, principal, CookieAuthenticationDefaults.AuthenticationScheme);
    var signInCommand = new SignInCommand(session, newUser, authenticatedIdentity);
    signInCommand.IsServerSide = true;
    await _authBackend.SignIn(signInCommand, cancellationToken);
    }
    
    protected virtual (User User, UserIdentity AuthenticatedIdentity) CreateFusionUser(User user, ClaimsPrincipal httpUser, string schema)
    {
        var httpUserIdentityName = httpUser.Identity?.Name ?? "";
        var claims = httpUser.Claims.ToImmutableDictionary(c => c.Type, c => c.Value);
        var id = FirstClaimOrDefault(claims, IdClaimKeys) ?? httpUserIdentityName;
        var name = FirstClaimOrDefault(claims, NameClaimKeys) ?? httpUserIdentityName;
        var identity = new UserIdentity(schema, id);
        var identities = ImmutableDictionary<UserIdentity, string>.Empty.Add(identity, "");

        user = new User("", name) {
            Claims = claims,
            Identities = identities
        };
        return (user, identity);
    }

    protected static string? FirstClaimOrDefault(IReadOnlyDictionary<string, string> claims, string[] keys)
    {
        foreach (var key in keys)
            if (claims.TryGetValue(key, out var value) && !string.IsNullOrEmpty(value))
                return value;
        return null;
    }
```

Fusion의 `IAuthBackend.SignIn()`을 호출하면 인증 상태가 Fusion의 저장소에 저장되고 쿠키도 생성되므로 평소대로 진행할 수 있습니다.

한 가지 주의해야 할 점은 Identity 내부에서 특정 사용자의 역할/클레임을 편집하는 경우 Fusion의 저장소 내에서 이를 무효화하거나 두 프레임워크를 동기화 상태로 유지하기 위해 사용자가 강제로 로그아웃해야 한다는 것입니다. Fusion 내부의 인증 상태를 업데이트하려면 업데이트된 역할/클레임이 포함된 새로 구성된 `ClaimsPrincipal` 개체를 사용하여 `IAuthBackend.SignIn`을 호출하기만 하면 됩니다.