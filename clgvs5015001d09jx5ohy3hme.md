---
title: "C#으로 .NET 프로파일러 작성 - 2부 | Kevin Gosse"
datePublished: Tue Apr 25 2023 04:41:17 GMT+0000 (Coordinated Universal Time)
cuid: clgvs5015001d09jx5ohy3hme
slug: c-net-2-kevin-gosse
canonical: https://minidump.net/writing-a-net-profiler-in-c-part-2-8039da001e43
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1682396923491/3894fd0d-84b1-460a-82c7-db826f108f6b.jpeg
tags: net, dotnet

---

> Kevin Gosse님의 [**Writing a .NET profiler in C# — Part 2**](https://minidump.net/writing-a-net-profiler-in-c-part-2-8039da001e43)을 번역하였습니다.

[첫 번째 파트](https://minidump.net/writing-a-net-profiler-in-c-part-1-d3978aae9b12)에서는 COM 객체의 레이아웃을 모방하고 이를 사용하여 IClassFactory의 가짜 인스턴스를 노출하는 방법을 살펴봤습니다. 이 방법은 잘 작동했지만 정적 메서드를 사용했기 때문에 여러 인스턴스가 예상될 때마다 객체의 상태를 추적하는 것이 편리하지 않았습니다. COM 개체를 .NET의 실제 개체 인스턴스에 매핑할 수 있다면 정말 좋을 것 같습니다.

이 시점에서 코드는 다음과 같습니다:

```csharp
public class DllMain
{
    private static ClassFactory Instance;

    [UnmanagedCallersOnly(EntryPoint = "DllGetClassObject")]
    public static unsafe int DllGetClassObject(void* rclsid, void* riid, nint* ppv)
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
}
```

이상적으로는 다음과 같이 인스턴스 메서드가 있는 실제 객체를 원합니다:

```csharp
public class ClassFactory
{
    public unsafe int QueryInterface(IntPtr self, Guid* guid, IntPtr* ptr)
    {
        Console.WriteLine("QueryInterface");
        *ptr = IntPtr.Zero;
        return 0;
    }

    public int AddRef(IntPtr self)
    {
        Console.WriteLine("AddRef");
        return 1;
    }

    public int Release(IntPtr self)
    {
        Console.WriteLine("Release");
        return 1;
    }

    public unsafe int CreateInstance(IntPtr self, IntPtr outer, Guid* guid, IntPtr* instance)
    {
        Console.WriteLine("CreateInstance");
        *instance = IntPtr.Zero;
        return 0;
    }

    public int LockServer(IntPtr self, bool @lock)
    {
        return 0;
    }
}
```

그러나 네이티브 측에서는 `UnmanagedCallersOnly` 속성으로 장식된 메서드만 호출할 수 있으며, 이 속성은 정적 메서드에만 적용될 수 있습니다. 따라서 정적 메서드 집합과 이러한 정적 메서드에서 객체의 인스턴스를 검색할 수 있는 방법이 필요합니다.

이를 달성하기 위한 핵심은 해당 메서드의 `self` 인자입니다. C++ 객체의 레이아웃을 모방하고 있기 때문에 네이티브 객체의 인스턴스 주소가 첫 번째 인자로 전달됩니다. 이를 사용하여 관리되는 객체를 검색하고 정적이지 않은 버전의 메서드를 호출할 수 있습니다. 예를 들어:

```csharp
public unsafe class ClassFactory
{
    private static Dictionary<IntPtr, ClassFactory> _instances = new();

    public ClassFactory()
    {
        // Allocate the chunk of memory for the vtable pointer + the pointers to the 5 methods
        var chunk = (IntPtr*)NativeMemory.Alloc(1 + 5, (nuint)IntPtr.Size);

        // Pointer to the vtable
        *chunk = (IntPtr)(chunk + 1);

        // Pointers to each method of the interface
        *(chunk + 1) = (IntPtr)(delegate* unmanaged<IntPtr, Guid*, IntPtr*, int>)&QueryInterfaceNative;

        // [...] (stripped for brevity

        _instances.Add((IntPtr)chunk, this);
    }

    public int QueryInterface(Guid* guid, IntPtr* ptr)
    {
        Console.WriteLine("QueryInterface");
        *ptr = IntPtr.Zero;
        return 0;
    }

    // [...] (same for other instance methods of ClassFactory)

    [UnmanagedCallersOnly]
    public static int QueryInterfaceNative(IntPtr self, Guid* guid, IntPtr* ptr)
    {
        var instance = _instances[self];

        return instance.QueryInterface(guid, ptr);
    }

    // [...] (same for other static methods of ClassFactory)
}
```

생성자에서는 연결된 네이티브 객체의 주소와 함께 정적 딕셔너리에 `ClassFactory`의 인스턴스를 추가합니다. 정적 메서드인 `QueryInterfaceNative`에서는 정적 딕셔너리에서 해당 인스턴스를 검색하고 비정적 메서드인 `QueryInterface`를 호출합니다.

작동은 하지만 메서드를 호출할 때마다 사전을 조회해야 한다는 점이 아쉽습니다. 게다가 동시성을 처리해야 합니다(아마도 `ConcurrentDictionary`를 사용해서). 더 나은 해결책이 있을까요?

이미 네이티브 객체에 대한 포인터가 있으므로 해당 네이티브 객체가 관리 객체에 대한 포인터를 저장할 수 있다면 좋을 것입니다. 이런 식으로요:

```csharp
public ClassFactory()
{
    // Allocate the chunk of memory for the vtable pointer + the address of the managed object + the pointers to the 5 methods
    var chunk = (IntPtr*)NativeMemory.Alloc(2 + 5, (nuint)IntPtr.Size);

    // Pointer to the vtable
    *chunk = (IntPtr)(chunk + 2);

    // Pointer to the managed object
    *(chunk + 1) = &this;

    // [...]
}
```

정적 메서드가 있다면 관리되는 객체에 대한 포인터를 가져오는 것만 하면 됩니다:

```csharp
[UnmanagedCallersOnly]
public static unsafe int QueryInterfaceNative(IntPtr* self, Guid* guid, IntPtr* ptr)
{
    var instance = *(ClassFactory*)(self + 1);

    return instance.QueryInterface(guid, ptr);
}
```

하지만 `&this`은 이 컴파일\*되지 않습니다. 그럴만한 이유가 있습니다: 관리되는 객체는 가비지 수집기에 의해 언제든지 이동할 수 있으므로 다음 가비지 수집 시 포인터가 유효하지 않게 될 수 있습니다.

> \*: 거짓말입니다. 최신 버전의 C#을 사용하는 경우 `this` 주소를 가져올 수 있습니다:
> 
> *var classFactory = this;  
> \*(chunk + 1) = (nint)(nint\*)&classFactory;*
> 
> 하지만 앞서 언급한 이유로 안전하지 않으므로 자신이 무엇을 하고 있는지 잘 알지 못한다면 하지 마세요.

이 문제를 해결하기 위해 개체를 고정하고 싶을 수도 있지만, 다른 관리되는 개체에 대한 참조가 있는 개체는 고정할 수 없으므로 이 역시 좋지 않습니다.

우리에게 필요한 것은 관리되는 오브젝트에 대한 일종의 고정된 참조인데, 다행히도 `GCHandle`이 이를 정확히 제공합니다. 관리되는 객체를 가리키는 `GCHandle`을 할당하면 `GCHandle.ToIntPtr`을 사용해 해당 핸들에 연결된 고정 주소를 가져오고, `GCHandle.FromIntPtr`을 사용해 해당 주소에서 핸들을 검색할 수 있습니다. 따라서 우리가 할 수 있는 일은:

```csharp
public ClassFactory()
{
    // Allocate the chunk of memory for the vtable pointer + the address of the managed object + the pointers to the 5 methods
    var chunk = (IntPtr*)NativeMemory.Alloc(2 + 5, (nuint)IntPtr.Size);

    // Pointer to the vtable
    *chunk = (IntPtr)(chunk + 2);

    // Pointer to the managed object
    var handle = GCHandle.Alloc(this);
    *(chunk + 1) = GCHandle.ToIntPtr(handle);

    // [...]
}
```

그런 다음 정적 메서드에서 핸들 및 관련 객체를 검색할 수 있습니다:

```csharp
[UnmanagedCallersOnly]
public static unsafe int QueryInterfaceNative(IntPtr* self, Guid* guid, IntPtr* ptr)
{
    var handleAddress = *(self + 1);
    var handle = GCHandle.FromIntPtr(handleAddress);
    var instance = (ClassFactory)handle.Target;

    return instance.QueryInterface(guid, ptr);
}
```

모든 것을 종합하면 이제 `ClassFactory`의 모습은 다음과 같습니다:

```csharp
public unsafe class ClassFactory
{
    public ClassFactory()
    {
        // Allocate the chunk of memory for the vtable pointer + the address of the managed object + the pointers to the 5 methods
        var chunk = (IntPtr*)NativeMemory.Alloc(2 + 5, (nuint)IntPtr.Size);

        // Pointer to the vtable
        *chunk = (IntPtr)(chunk + 2);

        // Pointer to the managed object
        var handle = GCHandle.Alloc(this);
        *(chunk + 1) = GCHandle.ToIntPtr(handle);

        *(chunk + 2) = (IntPtr)(delegate* unmanaged<IntPtr*, Guid*, IntPtr*, int>)&Exports.QueryInterface;
        *(chunk + 3) = (IntPtr)(delegate* unmanaged<IntPtr*, int>)&Exports.AddRef;
        *(chunk + 4) = (IntPtr)(delegate* unmanaged<IntPtr*, int>)&Exports.Release;
        *(chunk + 5) = (IntPtr)(delegate* unmanaged<IntPtr*, IntPtr, Guid*, IntPtr*, int>)&Exports.CreateInstance;
        *(chunk + 6) = (IntPtr)(delegate* unmanaged<IntPtr*, bool, int>)&Exports.LockServer;

        Object = (IntPtr)chunk;
    }

    public IntPtr Object { get; }

    public int QueryInterface(Guid* guid, IntPtr* ptr)
    {
        Console.WriteLine("QueryInterface");
        *ptr = IntPtr.Zero;
        return 0;
    }

    public int AddRef()
    {
        Console.WriteLine("AddRef");
        return 1;
    }

    public int Release()
    {
        Console.WriteLine("Release");
        return 1;
    }

    public int CreateInstance(IntPtr outer, Guid* guid, IntPtr* instance)
    {
        Console.WriteLine("CreateInstance");
        *instance = IntPtr.Zero;
        return 0;
    }

    public int LockServer(bool @lock)
    {
        Console.WriteLine("LockServer");
        return 0;
    }

    private class Exports
    {
        [UnmanagedCallersOnly]
        public static int QueryInterface(IntPtr* self, Guid* guid, IntPtr* ptr)
        {
            var handleAddress = *(self + 1);
            var handle = GCHandle.FromIntPtr(handleAddress);
            var obj = (ClassFactory)handle.Target;

            return obj.QueryInterface(guid, ptr);
        }


        [UnmanagedCallersOnly]
        public static int AddRef(IntPtr* self)
        {
            var handleAddress = *(self + 1);
            var handle = GCHandle.FromIntPtr(handleAddress);
            var obj = (ClassFactory)handle.Target;

            return obj.AddRef();
        }

        [UnmanagedCallersOnly]
        public static int Release(IntPtr* self)
        {
            var handleAddress = *(self + 1);
            var handle = GCHandle.FromIntPtr(handleAddress);
            var obj = (ClassFactory)handle.Target;

            return obj.Release();
        }
        
        [UnmanagedCallersOnly]
        public static unsafe int CreateInstance(IntPtr* self, IntPtr outer, Guid* guid, IntPtr* instance)
        {
            var handleAddress = *(self + 1);
            var handle = GCHandle.FromIntPtr(handleAddress);
            var obj = (ClassFactory)handle.Target;

            return obj.CreateInstance(outer, guid, instance);
        }

        [UnmanagedCallersOnly]
        public static int LockServer(IntPtr* self, bool @lock)
        {
            var handleAddress = *(self + 1);
            var handle = GCHandle.FromIntPtr(handleAddress);
            var obj = (ClassFactory)handle.Target;

            return obj.LockServer(@lock);
        }
    }
}
```

(이름 충돌을 피하기 위해 정적 메서드를 중첩 클래스로 옮겼습니다)

그리고 진입 지점부터 사용할 수 있습니다:

```csharp
public class DllMain
{
    private static ClassFactory Instance;

    [UnmanagedCallersOnly(EntryPoint = "DllGetClassObject")]
    public static unsafe int DllGetClassObject(void* rclsid, void* riid, nint* ppv)
    {
        Instance = new ClassFactory();

        Console.WriteLine("Hello from the profiling API");

        *ppv = Instance.Object;

        return HResult.S_OK;
    }
}
```

남은 것은 `ICorProfilerCallback`과 약 70개의 메서드에 대해 이 작업을 수행하는 것입니다. 이 작업을 수작업으로 수행하지 않을 것이므로 다음 글에서는 프로세스를 자동화하는 소스 생성기를 작성하겠습니다.