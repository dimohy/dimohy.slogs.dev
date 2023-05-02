---
title: "MVVM 툴킷 및 RelayCommands로 Blazor 서버 애플리케이션 만들기 | bromix"
datePublished: Tue May 02 2023 21:34:44 GMT+0000 (Coordinated Universal Time)
cuid: clh6sf9kp00060al3272hgaus
slug: building-a-blazor-server-application-with-mvvm-toolkit-and-relaycommands
canonical: https://itnext.io/building-a-blazor-server-application-with-mvvm-toolkit-and-relaycommands-b801ad33e5c9
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1683062525532/125d377c-292d-49a8-9d0e-e442eea8c33c.webp
tags: net, dotnet, blazor

---

> [bromix](https://bromix.medium.com/?source=post_page-----b801ad33e5c9--------------------------------)님의 [**Creating a Blazor Server Application with MVVM Toolkit and RelayCommands**](https://itnext.io/building-a-blazor-server-application-with-mvvm-toolkit-and-relaycommands-b801ad33e5c9)를 번역하였습니다.

---

[이전 글](https://dimohy.slogger.today/creating-a-blazor-server-application-with-c-and-mvvm-toolkit)에서는 [Blazor](https://dotnet.microsoft.com/en-us/apps/aspnet/web-apps/blazor)와 함께 [MVVM 툴킷](https://learn.microsoft.com/en-us/dotnet/communitytoolkit/mvvm/)을 시작하기 위한 기본 사항을 살펴보았습니다. 뷰모델에서 속성과 커스텀 함수를 사용하는 방법을 시연했습니다. 하지만 명령어 사용에 대해서는 자세히 다루지 않았습니다.

이번 후속 글에서는 이전 작업을 기반으로 최적의 코드를 작성하고 웹페이지에서 로직을 분리하여 유지 관리가 훨씬 더 쉬운 코드를 생성하는 `RelayCommands`를 도입하여 이전 작업을 구축할 것입니다.

## 새 WeatherViewModel 만들기

프로젝트에서 ViewModels 폴더 아래에서 `WeatherViewModel.cs` 파일을 찾습니다. `WeatherViewModel2.cs` 라는 이름으로 ViewModel의 복사본을 만듭니다.

![](https://miro.medium.com/v2/resize:fit:229/1*8LGHnGCO2wIqNZ3nXpHmmw.png align="left")

MVVM 툴킷의 `RelayCommands`를 사용하기 위해 `CommunityToolkit.Mvvm.Input` 네임스페이스를 가져와서 `WeatherViewModel2.cs` 파일을 업데이트합니다. 그런 다음 이전의 메서드를 대체하고 날씨 데이터의 로딩을 처리하기 위해 `Load`라는 비공개 메서드를 추가하고 `RelayCommand` 특성으로 꾸밉니다. MVVM 툴킷의 소스 생성기는 `LoadCommand`라는 공용 속성을 생성하며, 이 속성은 나중에 `FetchData2.razor` 페이지에서 사용할 수 있습니다. 또한 이전에 로딩 상태를 추적하는 데 사용했던 비공개 `_isLoading` 프로퍼티를 제거할 수 있는데, 이는 나중에 `LoadCommand`에서 처리할 것이기 때문입니다.

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using ExamplesBlazorMvvmToolkit.Data;

namespace ExamplesBlazorMvvmToolkit.ViewModels;

internal sealed partial class WeatherViewModel2 : ObservableObject
{
    private readonly WeatherForecastService _weatherService;

    public WeatherViewModel2(WeatherForecastService weatherService)
    {
        _weatherService = weatherService;
    }

    [ObservableProperty] private WeatherForecast[] _forecasts = Array.Empty<WeatherForecast>();

    [RelayCommand]
    private async Task Load()
    {
        await Task.Delay(TimeSpan.FromSeconds(2)); // simulate loading
        Forecasts = await _weatherService.GetForecastAsync(DateOnly.FromDateTime(DateTime.Now));
    }
}
```

## 새 FetchData 페이지 만들기

프로젝트의 페이지 폴더에서 `FetchData.razor` 파일을 찾습니다. 파일 이름을 `FetchData2.razor`로 지정하고 복사본을 만듭니다.

![](https://miro.medium.com/v2/resize:fit:232/1*R8oziSM-vYCRpoHpATiYrg.png align="left")

`FetchData2.razor`에서 마크업과 코드 뒷부분을 약간 변경합니다. `@page` 지시문을 "/fetchdata2"로 업데이트하고 `@inject` 지시문을 업데이트하여 새 `WeatherViewModel2`를 페이지에 삽입합니다. 또한 날씨 데이터를 로드하는 명령을 트리거하는 버튼을 추가하는데, 이 버튼은 명령이 실행되는 동안에는 비활성화됩니다. 또한 `IsLoading` 속성의 사용을 제거하고 이를 `WeatherViewModel2`의 일부인 `LoadCommand`의 `IsRunning` 속성으로 대체했습니다. 코드 비하인드에서는 사용자가 버튼을 클릭할 때 `LoadCommand`를 실행하는 `Reload`라는 메서드도 추가합니다. 또한 명령이 실행 중인지 여부에 따라 버튼 텍스트를 변경하기 위해 개인 속성 `ReloadText`를 추가합니다.

```csharp
@page "/fetchdata2"
@using ExamplesBlazorMvvmToolkit.ViewModels
@inject WeatherViewModel2 ViewModel

<PageTitle>Weather forecast</PageTitle>

<h1>Weather forecast</h1>

<p>This component demonstrates fetching data from a view model by using commands.</p>

<button class="btn btn-primary" disabled="@ViewModel.LoadCommand.IsRunning" @onclick="Reload">@ReloadText</button>
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
        await ViewModel.LoadCommand.ExecuteAsync(null);
    }

    private async Task Reload()
    {
        await ViewModel.LoadCommand.ExecuteAsync(null);
    }

    private string ReloadText => ViewModel.LoadCommand.IsRunning ? "Loading..." : "Reload";
}
```

`FetchData2.razor`에서 가장 중요한 변경 사항은 ViewModel에서 `LoadCommand` 속성을 사용한다는 것입니다. 이 속성은 `WeatherViewModel2.cs`파일의 `RelayCommand` 속성을 기반으로 MVVM 툴킷의 소스 생성기에 의해 자동으로 생성됩니다.

`LoadCommand` 속성은 이전 버전의 뷰 모델에서 사용되었던 `IsLoading` 프로퍼티를 대체합니다. 이는 `LoadCommand` 속성에 명령의 실행 상태를 추적하는 `IsRunning` 속성이 내장되어 있기 때문입니다.

Razor 컴포넌트에서는 날씨 데이터의 로딩을 처리하기 위해 `LoadCommand` 속성을 사용합니다. `LoadCommand`가 실행 중일 때는 다시 로드 버튼을 비활성화하고 "로드 중..."이라는 텍스트를 표시합니다. 데이터가 로드되면 버튼을 활성화하고 "다시 로드"라는 텍스트를 표시합니다.

## 내비게이션 및 서비스 등록 업데이트

NavMenu에 새 탐색 항목을 추가하려면 `NavMenu.razor` 파일을 수정해야 합니다. FetchData 페이지로 이동하는 기존 NavLink 컴포넌트를 복사하고 새로운 `FetchData2` 페이지로 이동하도록 수정하면 됩니다:

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

프로젝트를 시작하고 '데이터 가져오기(#2)'를 선택하면 초기 로딩이 나타납니다. 이 시간 동안에는 원치 않는 클릭을 방지하기 위해 다시 로드 버튼이 비활성화됩니다. 날씨 데이터가 로드되면 다시 로드 버튼이 활성화됩니다. 그러나 이후 다시 로드 버튼을 클릭하면 다시 한 번 비활성화됩니다.

![](https://miro.medium.com/v2/resize:fit:700/1*Zxb_HzVQ-n2cOiByA5Gzlg.gif align="left")

## 결론

MVVM 툴킷을 Blazor와 함께 사용하여 보다 유지 관리가 용이하고 분리된 코드베이스를 만드는 방법을 배웠습니다. `RelayCommands`의 사용법을 소개하고 로딩 상태를 수동으로 추적할 필요성을 내장된 IsRunning 속성으로 대체하는 방법을 시연했습니다. 또한 MVVM 툴킷의 소스 제너레이터를 사용하여 뷰모델의 릴레이 명령 속성을 기반으로 공개 명령을 자동으로 생성하는 방법도 보여드렸습니다. 이러한 단계를 따라 개발자는 유지 관리 및 업데이트가 더 쉬운 강력하고 확장 가능한 Blazor 애플리케이션을 만들 수 있습니다.

**3부** [MVVM 툴킷을 사용한 블레이저에서 쉽게 비동기 명령 취소하기](https://medium.com/itnext/canceling-async-commands-made-easy-in-blazor-with-mvvm-toolkit-b592923633a6)에서는 릴레이 명령과 비동기 메서드에 대해 자세히 살펴봅니다.

*Blazor와 함께 MVVM 툴킷을 사용하기 위한 예제 프로젝트는 GitHub 리포지토리에서 확인할 수 있습니다:*

[https://github.com/bromix/ExamplesBlazorMvvmToolkit](https://github.com/bromix/ExamplesBlazorMvvmToolkit).

---

%[https://itnext.io/building-a-blazor-server-application-with-mvvm-toolkit-and-relaycommands-b801ad33e5c9]