---
title: "MVVM 툴킷을 사용한 Blazor에서 쉽게 비동기 명령 취소하기 | bromix"
datePublished: Fri May 05 2023 01:06:08 GMT+0000 (Coordinated Universal Time)
cuid: clh9uutrb000709l244kahe5p
slug: canceling-async-commands-made-easy-in-blazor-with-mvvm-toolkit
canonical: https://itnext.io/canceling-async-commands-made-easy-in-blazor-with-mvvm-toolkit-b592923633a6
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1683247454777/8fb66db3-15d5-4a25-b989-57a2a21fcba0.webp
tags: net, dotnet, blazor

---

> [bromix](https://bromix.medium.com/?source=post_page-----b592923633a6--------------------------------)님의 [**Canceling Async Commands Made Easy in Blazor with MVVM Toolkit**](https://itnext.io/canceling-async-commands-made-easy-in-blazor-with-mvvm-toolkit-b592923633a6)을 번역하였습니다.

---

[이전 글](https://dimohy.slogger.today/building-a-blazor-server-application-with-mvvm-toolkit-and-relaycommands)에서는 날씨 데이터를 다시 로드하고 웹 페이지에서 로직을 분리하는 RelayCommand를 구현하여 예제를 확장했습니다. 그 결과 훨씬 더 유지 관리하기 쉬운 코드가 만들어졌습니다. 또한 이 글에서는 [MVVM 툴킷](https://learn.microsoft.com/en-us/dotnet/communitytoolkit/mvvm/)이 어떻게 비동기 명령어 작업을 간소화하는지 살펴보겠습니다.

비동기 메서드를 사용할 때는 가능하면 취소 토큰을 전달하는 것이 좋습니다. MVVM 툴킷에서 제공하는 RelayCommands를 사용하면 특히 이미 실행 중인 명령을 중지하거나 취소해야 하는 경우 많은 작업이 자동으로 수행됩니다.

## 새 WeatherViewModel 만들기

프로젝트에서 ViewModels 폴더 아래에서 `WeatherViewModel2.cs` 파일을 찾습니다. `WeatherViewModel3.cs` 라는 이름으로 ViewModel의 복사본을 만듭니다.

![](https://miro.medium.com/v2/resize:fit:243/1*CroGFaMcmVsnSHf-rp_P3A.png align="left")

이제 `Load` 메서드는 두 개의 매개변수, 즉 `TimeSpan`과 `CancellationToken`을 받습니다. 지연 매개변수는 로딩 작업을 시뮬레이션할 시간을 지정하고, `CancellationToken`은 필요한 경우 메서드의 실행을 취소하는 데 사용됩니다.

`Load` 메서드 내에서 `CancellationToken`은 `Task.Delay` 메서드로 전달됩니다. `Task.Delay` 실행 중에 `CancellationToken`되면, 이 메서드는 `OperationCanceledException`을 던집니다. 이 예외를 처리하기 위해 `Load` 메서드는 `OperationCanceledException`을 잡는 `try-catch` 블록을 사용합니다. 예외가 잡히면 예외는 무시되고 메서드는 추가 작업을 수행하지 않고 종료됩니다.

취소 요청이 있을 때 애플리케이션이 충돌하거나 예기치 않은 동작을 하지 않도록 하려면 이러한 방식으로 `OperationCanceledException`을 처리하는 것이 필수적입니다. 작업 취소는 애플리케이션에서 추가 조치가 필요하지 않은 유효하고 예상되는 조건으로 간주됩니다. `OperationCanceledException`에 대한 추가 정보는 [여기](https://learn.microsoft.com/en-us/dotnet/api/system.operationcanceledexception)에서 확인할 수 있습니다.

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using ExamplesBlazorMvvmToolkit.Data;

namespace ExamplesBlazorMvvmToolkit.ViewModels;

internal sealed partial class WeatherViewModel3 : ObservableObject
{
    private readonly WeatherForecastService _weatherService;

    public WeatherViewModel3(WeatherForecastService weatherService)
    {
        _weatherService = weatherService;
    }

    [ObservableProperty] private WeatherForecast[] _forecasts = Array.Empty<WeatherForecast>();

    [RelayCommand]
    private async Task Load(TimeSpan delay, CancellationToken cancellationToken)
    {
        try
        {
            await Task.Delay(delay, cancellationToken); // simulate loading
            Forecasts = await _weatherService.GetForecastAsync(DateOnly.FromDateTime(DateTime.Now));
        }
        catch (OperationCanceledException)
        {
            /*
             * This is okay to catch and in this case to ignore. The Cancellation will throw an
             * OperationCanceledException. For information read the following link:
             * 
             * https://learn.microsoft.com/en-us/dotnet/api/system.operationcanceledexception
             */
        }
    }
}
```

## 새 FetchData 페이지 만들기

프로젝트의 페이지 폴더에서 `FetchData2.razor` 파일을 찾습니다. 파일 이름을 `FetchData3.razor`로 지정하고 복사본을 만듭니다.

![](https://miro.medium.com/v2/resize:fit:237/1*_c8KR2myp0DZ8kVw-Lx8gw.png align="left")

업데이트된 버전의 페이지는 이제 "/fetchdata3"에서 확인할 수 있습니다. 이제 사용자가 서로 다른 속도로 날씨 데이터를 다시 로드할 수 있는 두 개의 버튼이 있습니다. "로드(빠르게)" 버튼은 데이터를 빠르게 다시 로드하고, "로드(느리게)" 버튼은 데이터를 느린 속도로 다시 로드합니다.

취소를 지원하기 위해 로드 명령을 취소할 수 있는 경우에만 표시되는 "Cancel" 버튼도 추가했습니다. 이 기능은 MVVM 툴킷에서 생성된 명령에 의해 제공됩니다. "Cancel" 버튼을 사용하면 사용자가 원할 경우 로딩 프로세스를 중지할 수 있어 애플리케이션의 사용자 편의성과 견고성이 향상됩니다.

하나의 재로드 버튼을 사용할 수도 있었지만, 여러 개의 릴레이 명령이 사용될 때 MVVM 툴킷이 어떻게 작동하는지 보여주기 위해 두 가지를 모두 포함하기로 결정했습니다.

```xml
@page "/fetchdata3"
@using ExamplesBlazorMvvmToolkit.ViewModels
@inject WeatherViewModel3 ViewModel

<PageTitle>Weather forecast</PageTitle>

<h1>Weather forecast</h1>

<p>This component demonstrates fetching data from a view model by using commands and also support cancellation.</p>

<button class="btn btn-primary" disabled="@ViewModel.LoadCommand.IsRunning" @onclick="ReloadFast">@ReloadText (Fast)</button>
<button class="btn btn-primary" disabled="@ViewModel.LoadCommand.IsRunning" @onclick="ReloadSlow">@ReloadText (Slow)</button>
<button class="btn btn-danger" hidden="@(!ViewModel.LoadCommand.CanBeCanceled)" @onclick="@ViewModel.LoadCommand.Cancel">Cancel</button>
<table class="table">
    <thead>
    <tr>
        <th>Date</th>
        <th>Temp. (C)</th>
        <th>Temp. (F)</th>
        <th>Summary</th>
    </tr>
    </thead>
    <tbody>
    @foreach (var forecast in ViewModel.Forecasts)
    {
        <tr>
            <td>@forecast.Date.ToShortDateString()</td>
            <td>@forecast.TemperatureC</td>
            <td>@forecast.TemperatureF</td>
            <td>@forecast.Summary</td>
        </tr>
    }
    </tbody>
</table>

@code {

    protected override async Task OnInitializedAsync()
    {
        await ReloadFast();
    }

    private async Task ReloadFast()
    {
        await ViewModel.LoadCommand.ExecuteAsync(TimeSpan.FromSeconds(1));
    }

    private async Task ReloadSlow()
    {
        await ViewModel.LoadCommand.ExecuteAsync(TimeSpan.FromSeconds(5));
    }

    private string ReloadText => ViewModel.LoadCommand.IsRunning ? "Loading..." : "Reload";
}
```

## 네비게이션 및 서비스 등록 업데이트

NavMenu에 새 탐색 항목을 추가하려면 `NavMenu.razor` 파일을 수정해야 합니다. FetchData 페이지로 이동하는 기존 `NavLink` 컴포넌트를 복사하고 새로운 `FetchData3` 페이지로 이동하도록 수정하면 됩니다:

```xml
<div class="nav-item px-3">
    <NavLink class="nav-link" href="counter">
        <span class="oi oi-plus" aria-hidden="true"></span> Counter
    </NavLink>
</div>
<div class="nav-item px-3">
    <NavLink class="nav-link" href="fetchdata">
        <span class="oi oi-list-rich" aria-hidden="true"></span> Fetch data (#1)
    </NavLink>
</div>
<div class="nav-item px-3">
    <NavLink class="nav-link" href="fetchdata2">
        <span class="oi oi-list-rich" aria-hidden="true"></span> Fetch data (#2)
    </NavLink>
</div>
<div class="nav-item px-3">
    <NavLink class="nav-link" href="fetchdata3">
        <span class="oi oi-list-rich" aria-hidden="true"></span> Fetch data (#3)
    </NavLink>
</div>
```

`FetchData3.razor` 파일의 `@page` 지시문과 일치하도록 `href` 속성을 "fetchdata3"로 설정해야 합니다.

`Program.cs` 파일에 `WeatherViewModel3.cs`를 등록하려면 `Main` 메서드에서 `builder.Services` 호출을 수정해야 합니다. 이렇게 새 뷰 모델을 범위가 지정된 서비스로 추가하기만 하면 됩니다:

```csharp
using ExamplesBlazorMvvmToolkit.Data;
using ExamplesBlazorMvvmToolkit.ViewModels;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddRazorPages();
builder.Services.AddServerSideBlazor();
builder.Services.AddSingleton<WeatherForecastService>();
builder.Services.AddScoped<WeatherViewModel>();
builder.Services.AddScoped<WeatherViewModel2>();
builder.Services.AddScoped<WeatherViewModel3>();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    // The default HSTS value is 30 days. You may want to change this for production scenarios, see https://aka.ms/aspnetcore-hsts.
    app.UseHsts();
}

app.UseHttpsRedirection();

app.UseStaticFiles();

app.UseRouting();

app.MapBlazorHub();
app.MapFallbackToPage("/_Host");

app.Run();
```

## 쇼타임

프로젝트를 시작하고 'Fetch Data (#3)'를 선택하면 초기 로딩 화면이 나타납니다. 이 시간 동안에는 의도하지 않은 클릭을 방지하기 위해 다시 로드 버튼이 모두 비활성화되고 취소 버튼이 표시됩니다. 날씨 데이터가 로드되면 다시 로드 버튼이 활성화되고 취소 버튼은 보이지 않게 됩니다.

사용자가 취소 버튼을 클릭하면 실행 중인 다시 로드 명령이 중단되고 버튼이 올바른 상태로 돌아갑니다. MVVM 툴킷 덕분에 이 기능의 로직이 자동으로 생성되어 원활한 사용자 경험을 제공합니다.

![](https://miro.medium.com/v2/resize:fit:700/1*ndCSvCNuM3pZJi1pWcI2kw.gif align="left")

## 결론

이 글에서는 MVVM 툴킷을 사용하여 Blazor 애플리케이션에서 비동기 명령 작업을 간소화하는 방법에 대해 알아보았습니다. 툴킷은 뷰모델 내의 비동기 작업에만 집중함으로써 호출 사이트에서 실행 중인 명령의 취소를 처리하여 코드를 보다 유지 관리하기 쉽고 효율적으로 만듭니다.

MVVM 툴킷을 활용하면 블레이저 프로젝트를 개선하고자 하는 개발자에게 유용한 도구가 될 수 있습니다.

*Blazor와 함께 MVVM 툴킷을 사용하기 위한 예제 프로젝트는 GitHub 리포지토리에서 확인할 수 있습니다:*

[https://github.com/bromix/ExamplesBlazorMvvmToolkit](https://github.com/bromix/ExamplesBlazorMvvmToolkit).

---

%[https://itnext.io/canceling-async-commands-made-easy-in-blazor-with-mvvm-toolkit-b592923633a6]