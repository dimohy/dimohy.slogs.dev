---
title: "새로운 계측 도구로 Visual Studio 성능 향상"
datePublished: Sat Apr 15 2023 12:04:31 GMT+0000 (Coordinated Universal Time)
cuid: clghxkhi8000j09mj4t2s24b5
slug: improving-visual-studio-performance-with-the-new-instrumentation-tool
tags: net, dotnet

---

> 본 글은 Nik Karpinsky님의 [Improving Visual Studio performance with the new Instrumentation Tool](https://devblogs.microsoft.com/visualstudio/improving-visual-studio-performance-with-the-new-instrumentation-tool/) 글을 번역한 글입니다.

---

## 개요

Visual Studio 2022 버전 17.6 출시와 함께 성능 프로파일러에 새롭게 개선된 [계측 도구](https://learn.microsoft.com/ko-kr/visualstudio/profiling/instrumentation?view=vs-2022)가 제공됩니다. CPU 사용량 도구와 달리 계측 도구는 정확한 타이밍과 호출 수를 제공하므로 차단된 시간과 평균 함수 시간을 파악하는 데 매우 유용합니다. 이 도구를 사용하여 Visual Studio의 성능을 개선해 보겠습니다.

## 잠깐, Visual Studio에 이미 계측 도구가 있는 줄 알았는데요?

"Visual Studio에 이미 계측 도구가 있지 않나요?"라고 생각하셨다면 제대로 생각하신 것입니다! 그럼, 뭐가... 새로워졌나요? 글쎄요, 전체 목록은 다음과 같습니다.

* **더 빠르고 더 적은 리소스**: 이 도구는 훨씬 더 빠르고 디스크 공간을 적게 사용하며, 리포지토리를 복제하고 측정값을 직접 확인할 수 있습니다. 샘플 앱: [ScabbleFinderDotNet](https://github.com/karpinsn/ScrabbleFinderDotNet)
    
    ![Graph of instrumentation tool performance. 35x smaller file and 7x less overhead](https://devblogs.microsoft.com/visualstudio/wp-content/uploads/sites/4/2023/04/InstrumentationToolPerformance.png align="left")
    
* **.NET에 대한 향상된 타겟팅**: 이 도구는 .NET 시나리오에 대한 타겟팅이 개선되어 계측 범위를 특정 함수까지 좁혀서 오버헤드를 줄이고 더 나은 데이터를 얻을 수 있습니다.
    
* **플레임 그래프**: 플레임 그래프를 사용하면 애플리케이션에서 가장 많은 시간이 소요되는 부분을 그래픽으로 확인하고 개선해야 할 부분을 빠르게 좁힐 수 있습니다.
    
* **더 나은 오류 처리**: 이 도구는 C++ 프로젝트의 "/profiler" 링커 플래그 누락과 같은 일반적인 문제를 해결하는 데 도움이 됩니다. 해결하지 못한 문제가 발생하면 [개발자 커뮤니티](https://developercommunity.visualstudio.com/home)에서 도움을 받을 수 있습니다.
    

## 성능을 위해 마이닝 하러 가자!

우선, 성능 프로파일러에서 진단 세션을 가져와서 분석 백엔드를 실행한 다음 종료하는 AnalyzerBench라는 콘솔 애플리케이션이 있습니다. 이를 통해 반복 가능한 벤치마크를 측정하고 변경의 효과를 확인할 수 있습니다. 제가 가지고 있는 진단 세션은 .NET 개체 할당 도구를 사용하여 Visual Studio 시작의 모든 할당, 430만 개 이상의 할당을 추적한 것입니다. 성능 프로파일러(Alt+F2)에서 계측 도구를 실행하면 다음과 같은 대화 상자가 표시됩니다.

![Targeted instrumentation dialog of Visual Studio Instrumentation tool](https://devblogs.microsoft.com/visualstudio/wp-content/uploads/sites/4/2023/03/TargetedInstrumentationDialog.png align="left")

이렇게 하면 계측할 프로젝트를 선택할 수 있으므로 오버헤드를 줄이기 위해 계측 대상을 지정할 수 있습니다. 제 경우에는 .NET 할당 도구에 대한 분석을 보고 싶었기 때문에 DataWarehouse 및 DotNetAllocAnalyzer 프로젝트를 선택했지만, 그다지 신경 쓰지 않기 때문에 AnalyzerBench는 선택하지 않았습니다. 무엇을 프로파일링할지 확실하지 않은 경우, CPU 사용량 도구를 사용하면 시간이 어디에 소비되고 있는지 대략적으로 파악한 다음 특정 영역을 대상으로 하는 계측 도구를 사용하여 더 자세히 조사할 수 있습니다. 도구를 실행하면 다음이 표시됩니다.

![Summary view of Visual Studio Instrumentation tool](https://devblogs.microsoft.com/visualstudio/wp-content/uploads/sites/4/2023/03/InstrumentationToolSummaryPage.png align="left")

상위 함수는 가장 많은 시간이 소요되는 함수를 표시하고, 핫 경로는 가장 비용이 많이 드는 코드 경로를 표시합니다. 세부 정보 패널을 열고 다음과 같이 표시되는 플레임 그래프로 전환합니다.

![Flame chart from Visual Studio Instrumentation Tool](https://devblogs.microsoft.com/visualstudio/wp-content/uploads/sites/4/2023/03/InstrumentationToolFlameChart.png align="left")

플레임 그래프를 보면 `System.Threading.Monitor.Enter`가 약 20%의 시간을 소비하는 것을 볼 수 있는데, 이는 매우 흥미롭습니다. 노드를 마우스 오른쪽 버튼으로 클릭하면 호출 트리에서 이 문제가 발생하는 위치를 상호 참조할 수 있습니다.

![Call tree and source view of Visual Studio Instrumentation tool showing lock contention](https://devblogs.microsoft.com/visualstudio/wp-content/uploads/sites/4/2023/03/InstrumentationToolLockContention.png align="left")

`Monitor.Enter` 함수가 핫 함수로 표시되고 있으며, 그 부모 `ImportDataSource`가 전체 시간의 약 17%를 차지하고 있는 것으로 나타났습니다. 호출 트리에서 헤더의 컨텍스트 메뉴에 더 많은 열이 숨겨져 있는 새로운 열이 몇 개 있는 것을 볼 수 있습니다. 계측 도구가 정확한 호출 수를 제공하기 때문에 최소, 최대 및 평균 함수 시간과 같은 통계를 계산할 수 있습니다. 계측 도구는 정확한 통화 수를 제공할 뿐만 아니라 벽시계 시간도 측정합니다. 이를 통해 경합과 같은 CPU와 관련이 없는 문제를 확인할 수 있습니다. 이 경우 호출 수에 따라 `ImportDataSource`에 대한 호출이 세 번 있으며 잠금 대기 시간으로 평균 약 5초가 소요된다는 것을 알 수 있습니다. 실제로는 첫 번째 호출이 잠금을 얻고 나머지 두 호출은 첫 번째 데이터 소스가 완료될 때까지 약 8초를 기다렸다가 가져올 수 있습니다. 즉, 2개의 스레드 풀 스레드가 동기적으로 차단되어 어떤 작업도 수행할 수 없어 스레드 풀 고갈로 이어질 수 있습니다.

이 문제를 해결하려면 일부 병렬 데이터 구조를 사용하여 잠금이 필요하지 않도록 하거나 메서드 서명을 비동기화하도록 변경하고 [SemaphoreSlim.WaitAsync](https://learn.microsoft.com/ko-kr/dotnet/api/system.threading.semaphoreslim.waitasync?view=net-7.0)를 사용하여 최소한 스레드 풀 스레드를 차단하지 않도록 하는 방법을 조사할 수 있습니다. 두 가지 변경 사항 모두 조금 더 복잡하므로 코드에 TODO를 추가하고 나중에 다시 돌아올 수 있습니다.

다시 플레임 그래프로 돌아가서, 다음으로 눈에 띄는 것은 `List.Sort`인데, 한눈에 보기에는 일부 데이터를 정렬하는 데 약 20%의 시간을 소비하고 있는 것처럼 보입니다. 다시 노드를 마우스 오른쪽 버튼으로 클릭하고 호출 트리로 상호 참조하면 세부 통계를 볼 수 있습니다. 여기에는 데이터를 정렬하는 데 20초를 소비하면서 24,000회 이상 sort를 호출하고 있음을 보여줍니다!

![Call tree and source view of Visual Studio Instrumentation tool showing time spent in sort](https://devblogs.microsoft.com/visualstudio/wp-content/uploads/sites/4/2023/03/InstrumentationToolSortIssue.png align="left")

이 코드에서는 프로파일러의 그래프에서 시간 범위를 선택할 때 빠르게 필터링하는 데 필요한 각 고유 유형에 대한 할당을 정렬하고 있습니다. 대부분의 경우 이러한 할당은 각 할당에 대한 콜백을 가져와서 진단 세션 파일에 기록할 때 정렬되어야 합니다. 서로 다른 스레드에 동시에 많은 할당이 있는 경우 순서가 맞지 않을 수 있지만 목록에 추가할 때 이를 확인한 다음 정렬되지 않은 목록만 여기에서 정렬할 수 있습니다. 이렇게 변경하고 계측 도구를 다시 실행하면 이제 여기에 소요되는 시간이 거의 모두 제거되었으며, 이 추적의 경우 모든 할당이 이미 파일에서 정렬되어 있으므로 아무것도 정렬할 필요가 없음을 알 수 있습니다.

```csharp
/// <summary>
/// Add an allocation instance to the current type
/// </summary>
/// <param name="allocationObject">allocation object</param>
internal void AddAllocation(AllocationObject allocationObject)
{
    if (this.Allocations.Count > 0)
    {
        this.allocationsSorted &= allocationObject.AllocTimeNs >= this.Allocations[this.Allocations.Count - 1].AllocTimeNs;
    }

    this.Allocations.Add(allocationObject);
}

/// <summary>
/// Finalizes the data for fast retrieval
/// </summary>
public void FinalizeData()
{
    if (!this.allocationsSorted)
    {
        this.Allocations.Sort(TypeObject.comparer);
        this.allocationsSorted = true;
    }
}
```

![Call tree and source view in Visual Studio instrumentation tool showing improved sort performance](https://devblogs.microsoft.com/visualstudio/wp-content/uploads/sites/4/2023/03/InstrumentationToolFixedSort.png align="left")

여기서 목록의 정렬 상태를 추적하는 작은 변경으로 110초의 추적 시간을 20초로 단축하여 약 20%의 성능 향상을 가져왔습니다.

## 결론

새로운 계측 도구는 정말 훌륭하며(적어도 저는 그렇게 생각합니다 😊) 약간의 성능 조사만으로도 큰 도움이 될 수 있습니다. 한 시간도 채 안 되는 시간 동안 코드를 프로파일링하고 들여다본 결과 .NET 할당 도구의 로드 성능이 약 20% 향상되었습니다. 코드를 프로파일링하면서 어떤 점을 발견했는지, 새로운 계측 도구로 어떤 개선 요소를 달성할 수 있었는지 알려주세요!

---

https://devblogs.microsoft.com/visualstudio/improving-visual-studio-performance-with-the-new-instrumentation-tool/