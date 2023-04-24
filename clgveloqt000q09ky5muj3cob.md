---
title: "C#으로 .NET 프로파일러 작성 - 1부 | Kevin Gosse"
datePublished: Mon Apr 24 2023 22:22:21 GMT+0000 (Coordinated Universal Time)
cuid: clgveloqt000q09ky5muj3cob
slug: writing-a-net-profiler-in-c-part-1
canonical: https://minidump.net/writing-a-net-profiler-in-c-part-1-d3978aae9b12
tags: net, dotnet

---

> Kevin Gosse님의 [**Writing a .NET profiler in C# - Part 1**](https://minidump.net/writing-a-net-profiler-in-c-part-1-d3978aae9b12)**을 시리즈를 번역하였습니다.**

---

.NET에는 런타임을 면밀히 모니터링하고, 실행 중에 메서드를 동적으로 다시 작성하고, 임의의 시점에 스레드의 콜스택을 따라가는 등의 작업을 수행할 수 있는 매우 강력한 프로파일링 API가 있습니다. 하지만 해당 API 사용법을 배우기 위한 진입 비용이 상당히 높습니다. 첫 번째 이유는 많은 기능을 사용하려면 .NET 메타데이터 시스템의 작동 방식에 대한 충분한 지식이 필요하기 때문입니다. 또 다른 이유는 모든 문서와 예제가 C++로 작성되어 있기 때문입니다.

이론적으로는 대부분의 언어를 사용하여 .NET 프로파일러를 작성할 수 있습니다. 예를 들어 [Rust를 사용한 개념 증명](https://github.com/camdenreslink/clr-profiler)이 있습니다. 하지만 C#을 사용하는 것은 거의 불가능에 가까웠습니다. 프로파일러가 .NET 라이브러리인 경우 프로파일링된 애플리케이션과 동일한 런타임을 사용하게 되므로 몇 가지 문제가 발생할 수 있습니다:

* 프로파일러는 .NET 라이브러리이므로 결국 자체적으로 프로파일링하게 됩니다. 이것은 생각보다 문제가 많습니다. 예를 들어, 프로파일링된 애플리케이션의 메서드가 컴파일되면 런타임에서 프로파일링 이벤트 [`JITCompilationStarted`](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilercallback-jitcompilationstarted-method?WT.mc_id=DT-MVP-5003493)가 발생합니다. 그러면 프로파일러에서 콜백이 호출되는데, 이 콜백은 먼저 JIT에 의해 컴파일되어야 합니다. 그러면 콜백이 호출되고, 콜백이 호출된 콜백은 먼저 JIT에 의해 컴파일되어야 하므로 또 다른 `JITCompilationStarted` 이벤트가 발생합니다... 요점을 이해하셨겠죠?
    
* 이 문제에 대한 해결책을 찾았다고 해도 훨씬 더 실용적인 문제가 있는데, 바로 런타임 초기화 중에 시스템이 .NET 코드를 실행할 준비가 되지 않은 시점에 프로파일러가 매우 일찍 로드된다는 것입니다.
    

C#은 C# 개발자들이 가장 친숙하게 사용하는 언어이기 때문에 항상 안타깝게 생각했습니다. 다행히도 상황이 바뀌었습니다.

[이전 기사](https://minidump.net/writing-native-windbg-extensions-in-c-5390726f3cec)에서 이미 언급했지만, Microsoft는 NativeAOT를 적극적으로 개발하고 있습니다. 이 도구를 사용하면 .NET 라이브러리를 네이티브 독립형 라이브러리로 컴파일할 수 있습니다. 독립형은 자체 런타임(자체 GC, 자체 스레드풀, 자체 유형 시스템 등)과 함께 제공되므로 네이티브 라이브러리와 정확히 동일한 제한을 가진 프로세스에 로드할 수 있습니다. 즉, 이론적으로는 C#에서 .NET 프로파일러를 작성하는 데 사용할 수 있습니다.

## 설정

.NET 프로파일러를 작성하는 방법을 배우려면 [Christophe Nasarre가 작성한 기사](https://chnasarre.medium.com/start-a-journey-into-the-net-profiling-apis-40c76e2e36cc)를 참조하세요. 간단히 말해, [`IClassFactory`](https://learn.microsoft.com/en-us/windows/win32/api/unknwn/nn-unknwn-iclassfactory?WT.mc_id=DT-MVP-5003493)의 인스턴스를 반환하는 [`DllGetClassObject`](https://learn.microsoft.com/en-us/windows/win32/api/combaseapi/nf-combaseapi-dllgetclassobject) 메서드를 노출해야 합니다. .NET 런타임은 클래스 팩토리에서 `CreateInstance` 메서드를 호출하고, 이 메서드는 [`ICorProfilerCallback`](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilercallback-interface?WT.mc_id=DT-MVP-5003493)(또는 지원하려는 프로파일링 API 버전에 따라 [`ICorProfilerCallback2`](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilercallback2-interface?WT.mc_id=DT-MVP-5003493), [`ICorProfilerCallback3`](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilercallback3-interface), ...) 인스턴스를 반환할 것입니다. 마지막으로, 런타임은 프로파일링 API를 쿼리하는 데 필요한 [`ICorProfilerInfo`](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilerinfo-interface?WT.mc_id=DT-MVP-5003493) 인스턴스(또는 [ICorProfilerInfo2](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilerinfo2-interface?WT.mc_id=DT-MVP-5003493), [ICorProfilerInfo3](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilerinfo3-interface), ...)를 가져오는 데 사용할 수 있는 [`IUnknown`](https://learn.microsoft.com/en-us/windows/win32/api/unknwn/nn-unknwn-iunknown?WT.mc_id=DT-MVP-5003493) 파라미터를 사용하여 해당 인스턴스에서 [`Initialize`](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilercallback-initialize-method?WT.mc_id=DT-MVP-5003493) 메서드를 호출합니다.

정말 많네요. 첫 번째 단계인 `DllGetClassObject` 메서드 내보내기부터 시작하겠습니다. 먼저 .NET 6 클래스 라이브러리 프로젝트를 생성하고 버전 `7.0.0-preview.*`에서 [`Microsoft.DotNet.ILCompiler`](https://www.nuget.org/packages/Microsoft.DotNet.ILCompiler)에 대한 참조를 추가합니다. 그런 다음 `DllGetClassObject` 메서드가 있는 DllMain 클래스(이름은 중요하지 않음)를 생성합니다. 또한 이 메서드를 [`UnmanagedCallersOnly`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.unmanagedcallersonlyattribute?view=net-6.0&%3FWT.mc_id=DT-MVP-5003493) 속성으로 장식하여 NativeAOT 툴체인에 메서드를 내보내도록 지시합니다.

```csharp
using System;
using System.Runtime.InteropServices;

namespace ManagedDotnetProfiler;

public class DllMain
{
    [UnmanagedCallersOnly(EntryPoint = "DllGetClassObject")]
    public static unsafe int DllGetClassObject(Guid* rclsid, Guid* riid, IntPtr* ppv)
    {
        Console.WriteLine("Hello from the profiling API");

        return 0;
    }
}
```

그런 다음 `/p:NativeLib=Shared` 명령과 함께 `dotnet publish` 명령을 실행하여 네이티브 라이브러리를 생성합니다:

```bash
$ dotnet publish /p:NativeLib=Shared /p:SelfContained=true -r win-x64 -c Release
```

출력은 .dll 파일(Linux에서는 .so)입니다. 모든 것이 예상대로 작동하는지 테스트하기 위해 올바른 환경 변수를 설정한 후 .NET 콘솔 애플리케이션을 실행할 수 있습니다:

```bash
set CORECLR_ENABLE_PROFILING=1
set CORECLR_PROFILER={B3A10128-F10D-4044-AB27-A799DB8B7E4F}
set CORECLR_PROFILER_PATH=C:\git\ManagedDotnetProfiler\ManagedDotnetProfiler\bin\Release\net6.0\win-x64\publish\ManagedDotnetProfiler.dll
```

`CORECLR_ENABLE_PROFILING`은 런타임에 프로파일러를 로드하도록 지시합니다. `CORECLR_PROFILER`는 프로파일러를 고유하게 식별하는 GUID입니다(현재는 아무 값이나 사용 가능). `CORECLR_PROFILER_PATH`는 NativeAOT와 함께 게시한 dll의 경로입니다. 모든 것이 제대로 작동했다면 대상 앱을 로딩하는 동안 메시지가 표시되어야 합니다:

```bash
C:\console\bin\Debug\net6.0>console.exe
Hello from the profiling API
Hello, World!
```

훌륭하지만 아직은 유용하지 않습니다. 실제 프로파일러는 어떻게 작성할까요? 이제 `IClassFactory`의 인스턴스를 노출하는 방법을 이해해야 합니다.

## (일종의) C++ 인터페이스 노출

MSDN 문서에 따르면 `IClassFactory`는 인터페이스라고 나와 있습니다. 하지만 "인터페이스"는 C++와 C#에서 다른 의미이므로 .NET 코드에서 `IClassFactory`를 구현하고 하루를 끝낼 수는 없습니다.

사실 인터페이스라는 개념은 C++에 실제로 존재하지 않습니다. 실제로는 순수한 가상 함수만 포함하는 추상 클래스를 지정할 뿐입니다. 따라서 우리는 C++ 추상 클래스처럼 보이는 객체를 빌드하고 노출해야 합니다. 이를 위해서는 [vtable](https://en.wikipedia.org/wiki/Virtual_method_table)의 개념을 이해해야 합니다.

하나의 메서드 `DoSomething`이 있는 인터페이스 `IInterface`와 두 개의 구현 `ClassA`와 `ClassB`가 있다고 가정해 봅시다. `ClassA`와 `ClassB`는 모두 자체 구현을 선언할 수 있으므로 런타임은 `IInterface`의 인스턴스에 대한 포인터가 주어졌을 때 어느 것을 호출할지 알기 위해 어느 정도의 방향성이 필요합니다. 이러한 인디렉션을 가상 테이블 또는 vtable이라고 합니다.

관례에 따라 클래스가 가상 메서드를 구현할 때 C++ 컴파일러는 객체의 시작 부분에 숨겨진 필드를 생성합니다. 이 숨겨진 필드에는 vtable에 대한 포인터가 포함됩니다. vtable은 선언된 순서대로 각 가상 메서드의 구현 주소를 포함하는 메모리 청크입니다. 가상 메서드를 호출할 때 런타임은 먼저 vtable을 가져온 다음 이를 사용하여 구현의 주소를 가져옵니다.

예를 들어 다중 상속을 처리하는 등 vtable에는 더 많은 특수성이 있지만 이 글에서는 이에 대해 알 필요가 없습니다.

요약하자면, C++ 런타임에서 사용할 수 있는 `IClassFactory` 객체를 생성하려면 함수의 주소를 저장할 메모리 청크을 할당해야 합니다. 이것이 우리의 가상 테이블입니다. 그런 다음 vtable에 대한 포인터를 포함하는 또 다른 메모리 청크이 필요합니다. 이것이 우리의 인스턴스입니다.

![](https://miro.medium.com/v2/resize:fit:700/1*ztC0n9nKCPQRuw5pY-8Veg.png align="left")

간단하게 하기 위해 인스턴스와 vtable을 단일 메모리 청크으로 병합할 수 있습니다:

![](https://miro.medium.com/v2/resize:fit:638/1*D-z6gYzuO98n3-MJsQX7ug.png align="left")

그렇다면 C#에서는 어떻게 보일까요? 먼저 IClassFactory 인터페이스에서 각 함수에 대한 정적 메서드를 선언하고 UnmanagedCallersOnly로 장식합니다:

```csharp
    [UnmanagedCallersOnly]
    public static unsafe int QueryInterface(IntPtr self, Guid* guid, IntPtr* ptr)
    {
        Console.WriteLine("QueryInterface");
        *ptr = IntPtr.Zero;
        return 0;
    }

    [UnmanagedCallersOnly]
    public static int AddRef(IntPtr self)
    {
        Console.WriteLine("AddRef");
        return 1;
    }

    [UnmanagedCallersOnly]
    public static int Release(IntPtr self)
    {
        Console.WriteLine("Release");
        return 1;
    }

    [UnmanagedCallersOnly]
    public static unsafe int CreateInstance(IntPtr self, IntPtr outer, Guid* guid, IntPtr* instance)
    {
        Console.WriteLine("CreateInstance");
        *instance = IntPtr.Zero;
        return 0;
    }

    [UnmanagedCallersOnly]
    public static int LockServer(IntPtr self, bool @lock)
    {
        return 0;
    }
```

그런 다음 `DllGetClassObject`에서 vtable(가짜 인스턴스)에 대한 포인터를 저장하는 데 사용할 메모리 청크과 vtable 자체를 할당합니다. 이 메모리는 네이티브 코드에서 사용되므로 가비지 컬렉터에 의해 이동되지 않도록 해야 합니다. IntPtr 배열을 선언하고 고정할 수도 있지만, 저는 대신 [`NativeMemory.Alloc`](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.interopservices.nativememory.alloc?view=net-6.0&WT.mc_id=DT-MVP-5003493)을 사용하여 GC가 추적하지 않는 메모리를 할당하는 것을 선호합니다. 정적 메서드의 주소를 얻으려면 함수 포인터로 형변환한 다음 `IntPtr`로 형변환하면 됩니다. 마지막으로 함수의 `ppv` 인수를 통해 메모리 청크의 주소를 반환합니다.

```csharp
  [UnmanagedCallersOnly(EntryPoint = "DllGetClassObject")]
    public static unsafe int DllGetClassObject(Guid* rclsid, Guid* riid, IntPtr* ppv)
    {
        Console.WriteLine("Hello from the profiling API");

        // Allocate the chunk of memory for the vtable pointer + the pointers to the 5 methods
        var chunk = (IntPtr*)NativeMemory.Alloc(1 + 5, (nuint)IntPtr.Size);

        // Pointer to the vtable
        *chunk = (IntPtr)(chunk + 1);

        // Pointers to each method of the interface
        *(chunk + 1) = (IntPtr)(delegate* unmanaged<IntPtr, Guid*, IntPtr*, int>)&QueryInterface;
        *(chunk + 2) = (IntPtr)(delegate* unmanaged<IntPtr, int>)&AddRef;
        *(chunk + 3) = (IntPtr)(delegate* unmanaged<IntPtr, int>)&Release;
        *(chunk + 4) = (IntPtr)(delegate* unmanaged<IntPtr, IntPtr, Guid*, IntPtr*, int>)&CreateInstance;
        *(chunk + 5) = (IntPtr)(delegate* unmanaged<IntPtr, bool, int>)&LockServer;
        
        *ppv = (IntPtr)chunk;
        
        return HResult.S_OK;
    }
```

컴파일과 테스트가 끝나면 가짜 `IClassFactory`의 `CreateInstance` 메서드가 예상대로 호출되는 것을 확인할 수 있습니다:

```bash
C:\console\bin\Debug\net6.0> .\console.exe
Hello from the profiling API
CreateInstance
Release
Hello, World!
```

## 이제 시작에 불과합니다.

다음 단계는 `CreateInstance` 메서드를 구현하는 것입니다. 앞서 설명한 것처럼 `ICorProfilerCallback`의 인스턴스를 반환할 것으로 예상됩니다. 이 인터페이스를 구현하기 위해 방금 `IClassFactory`에서 한 것과 동일한 작업을 수행할 수 있지만 `ICorProfilerCallback`에는 거의 70개의 메서드가 포함되어 있습니다! `ICorProfilerCallback2`, `ICorProfilerCallback3` 등은 말할 것도 없고, 작성해야 할 상용구 코드가 엄청나게 많습니다. 게다가 현재 솔루션은 정적 메서드에서만 작동하므로 인스턴스 메서드와 함께 작동하는 솔루션이 있으면 정말 좋을 것입니다. 다음 연재에서는 이러한 문제를 해결하는 방법을 살펴보겠습니다.

---

%[https://minidump.net/writing-a-net-profiler-in-c-part-1-d3978aae9b12]