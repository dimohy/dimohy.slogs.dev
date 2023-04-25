---
title: "C#으로 .NET 프로파일러 작성 - 3부 | Kevin Gosse"
datePublished: Tue Apr 25 2023 05:06:41 GMT+0000 (Coordinated Universal Time)
cuid: clgvt1o2p000z09k2et7a67bd
slug: writing-a-net-profiler-in-c-part-3
canonical: https://minidump.net/writing-a-net-profiler-in-c-part-3-7d2c59fc017f
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1682399165384/7c188cdd-256b-4c4c-a9d9-e451c90c2847.jpeg
tags: net, dotnet

---

> ***Kevin Gosse님의*** [**Writing a .NET profiler in C# — Part 3**](https://minidump.net/writing-a-net-profiler-in-c-part-3-7d2c59fc017f)***을 번역하였습니다.***

---

[1부에서](https://minidump.net/writing-a-net-profiler-in-c-part-1-d3978aae9b12)는 NativeAOT를 사용하여 C#으로 프로파일러를 작성하는 방법과 프로파일링 API를 사용하기 위해 가짜 COM 객체를 노출하는 방법을 살펴봤습니다. [2부에서](https://minidump.net/writing-a-net-profiler-in-c-part-2-8039da001e43)는 정적 메서드 대신 인스턴스 메서드를 사용하도록 솔루션을 개선했습니다. 이제 프로파일링 API와 상호 작용하는 방법을 알았으므로 [`ICorProfilerCallback`](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilercallback-interface?WT.mc_id=DT-MVP-5003493) 인터페이스에 선언된 70개 이상의 메서드를 구현하는 데 필요한 상용구 코드를 자동으로 생성하는 소스 생성기를 작성해 보겠습니다.

먼저 수동으로 [`ICorProfilerCallback` 인터페이스를 C#으로](https://github.com/kevingosse/ManagedDotnetProfiler/blob/a204080e9299b9bb774f0f5d291640fe081fc04c/ManagedDotnetProfiler/ICorProfilerCallback.cs) 변환해야 합니다. 기술적으로는 C++ 헤더 파일에서 자동으로 생성할 수도 있지만, 동일한 C++ 코드가 C#에서는 다른 방식으로 변환될 수 있으므로 함수의 목적을 이해하여 올바른 의미로 변환하는 것이 중요합니다.

실제 예를 들어 [`JITInlining`](https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/profiling/icorprofilercallback-jitinlining-method?WT.mc_id=DT-MVP-5003493) 함수를 살펴보겠습니다. C++의 프로토타입은 다음과 같습니다:

```cpp
HRESULT JITInlining(FunctionID callerId, FunctionID calleeId, BOOL *pfShouldInline);
```

C#에서 순수한 변환은 다음과 같습니다:

```csharp
HResult JITInlining(FunctionId callerId, FunctionId calleeId, bool* pfShouldInline);
```

하지만 여기서는 실제로 포인터가 필요하지 않습니다. pfShouldInline 인수가 읽기 전용이면 다음과 같이 변환할 수 있습니다:

```csharp
HResult JITInlining(FunctionId callerId, FunctionId calleeId, in bool pfShouldInline);
```

하지만 함수 설명서를 보면 `pfShouldInline`은 함수 자체에서 설정해야 하는 값이라는 것을 알 수 있습니다. 따라서 대신 `out` 키워드를 사용해야 합니다:

```csharp
HResult JITInlining(FunctionId callerId, FunctionId calleeId, out bool pfShouldInline);
```

다른 경우에는 의도에 따라 `in` 또는 `ref`를 사용합니다. 그렇기 때문에 프로세스를 완전히 자동화할 수 없습니다.

인터페이스를 C#으로 번역한 후에는 소스 생성기 생성을 진행할 수 있습니다. API가 매우 복잡하기 때문에 최첨단 소스 생성기를 작성할 생각은 없으며(예, C#으로 프로파일러를 작성하는 방법을 설명하는 사람에게서 나온 말입니다), 이에 대한 자세한 내용은 [Andrew Lock의 훌륭한 글](https://andrewlock.net/creating-a-source-generator-part-1-creating-an-incremental-source-generator/)을 확인하실 수 있습니다.

## 소스 생성기 작성

소스 생성기를 생성하기 위해 솔루션에 `netstandard2.0`을 대상으로 하는 클래스 라이브러리 프로젝트를 추가하고 `Microsoft.CodeAnalysis.CSharp` 및 `Microsoft.CodeAnalysis.Analyzers`에 대한 참조를 추가합니다:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <LangVersion>latest</LangVersion>
    <IsRoslynComponent>true</IsRoslynComponent>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.0.1" PrivateAssets="all" />
    <PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.3.3">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
  </ItemGroup>

</Project>
```

그런 다음 `ISourceGenerator` 를 구현하는 클래스를 추가하고 `[Generator]` 특성으로 장식합니다:

```csharp
[Generator]
public class NativeObjectGenerator : ISourceGenerator
{
    public void Initialize(GeneratorInitializationContext context)
    {
    }

    public void Execute(GeneratorExecutionContext context)
    {
    }
}
```

가장 먼저 할 일은 `[NativeObject]` 특성을 방출하는 것입니다. 이 특성을 사용하여 소스 생성기를 실행할 인터페이스를 꾸밀 것입니다. 이 코드를 파이프라인 초기에 실행하기 위해 `RegisterForPostInitialization`을 사용합니다:

```csharp
[Generator]
public class NativeObjectGenerator : ISourceGenerator
{
    public void Initialize(GeneratorInitializationContext context)
    {
        context.RegisterForPostInitialization(EmitAttribute);

    }

    public void Execute(GeneratorExecutionContext context)
    {
    }

    private void EmitAttribute(GeneratorPostInitializationContext context)
    {
        context.AddSource("NativeObjectAttribute.g.cs", """
    using System;

    [AttributeUsage(AttributeTargets.Interface, Inherited = false, AllowMultiple = false)]
    internal class NativeObjectAttribute : Attribute { }
    """);
    }
}
```

이제 유형을 검사하고 어떤 유형이 `[NativeObject]` 특성으로 장식되어 있는지 감지하기 위해 `ISyntaxContextReceiver`를 등록해야 합니다.

```csharp
public class SyntaxReceiver : ISyntaxContextReceiver
{
    public List<INamedTypeSymbol> Interfaces { get; } = new();

    public void OnVisitSyntaxNode(GeneratorSyntaxContext context)
    {
        if (context.Node is InterfaceDeclarationSyntax classDeclarationSyntax
            && classDeclarationSyntax.AttributeLists.Count > 0)
        {
            var symbol = (INamedTypeSymbol)context.SemanticModel.GetDeclaredSymbol(classDeclarationSyntax);

            if (symbol.GetAttributes().Any(a => a.AttributeClass.ToDisplayString() == "NativeObjectAttribute"))
            {
                Interfaces.Add(symbol);
            }
        }
    }
}
```

기본적으로 구문 수신기는 구문 트리의 모든 노드에 대해 호출됩니다. 해당 노드가 인터페이스 선언인지 확인하고, 인터페이스 선언인 경우 특성을 검사하여 `NativeObjectAttribute`를 찾습니다. 개선할 수 있는 부분이 많겠지만, 특히 `NativeObjectAttribute`인지 확인하기 위해 개선할 수 있는 부분이 많겠지만, 우리의 목적에는 이 정도면 충분하다고 하겠습니다.

구문 수신기는 소스 생성기를 초기화하는 동안 등록해야 합니다:

```csharp
    public void Initialize(GeneratorInitializationContext context)
    {
        context.RegisterForPostInitialization(EmitAttribute);
        context.RegisterForSyntaxNotifications(() => new SyntaxReceiver());
    }
```

마지막으로 `Execute` 메서드에서는 구문 수신기에 저장된 인터페이스 목록을 검색하고 해당 인터페이스에 대한 코드를 생성합니다:

```csharp
    public void Execute(GeneratorExecutionContext context)
    {
        if (!(context.SyntaxContextReceiver is SyntaxReceiver receiver))
        {
            return;
        }

        foreach (var symbol in receiver.Interfaces)
        {
            EmitStubForInterface(context, symbol);
        }
    }
```

![](https://miro.medium.com/v2/resize:fit:700/1*WWnCdVg1-k0_HesCz4YJRQ.png align="left")

## 네이티브 래퍼 생성

`EmitStubForInterface` 메서드의 경우 템플릿 엔진을 사용할 수도 있지만, 대신 기존의 `StringBuilder`와 `Replace` 호출에 의존할 것입니다.

먼저 템플릿을 만듭니다:

```csharp
        var sourceBuilder = new StringBuilder("""
    using System;
    using System.Runtime.InteropServices;

    namespace NativeObjects
    {
        {visibility} unsafe class {typeName} : IDisposable
        {
            private {typeName}({interfaceName} implementation)
            {
                const int delegateCount = {delegateCount};

                var obj = (IntPtr*)NativeMemory.Alloc((nuint)2 + delegateCount, (nuint)IntPtr.Size);
    
                var vtable = obj + 2;

                *obj = (IntPtr)vtable;
    
                var handle = GCHandle.Alloc(implementation);
                *(obj + 1) = GCHandle.ToIntPtr(handle);

    {functionPointers}

                Object = (IntPtr)obj;
            }

            public IntPtr Object { get; private set; }

            public static {typeName} Wrap({interfaceName} implementation) => new(implementation);

            public static implicit operator IntPtr({typeName} stub) => stub.Object;

            ~{typeName}()
            {
                Dispose();
            }

            public void Dispose()
            {
                if (Object != IntPtr.Zero)
                {
                    NativeMemory.Free((void*)Object);
                    Object = IntPtr.Zero;
                }

                GC.SuppressFinalize(this);
            }

            private static class Exports
            {
    {exports}
            }
        }
    }
    """);
```

이해가 안 되는 부분이 있다면 [이전 글](https://minidump.net/writing-a-net-profiler-in-c-part-2-8039da001e43)을 꼭 확인하시기 바랍니다. 여기서 새로 추가된 것은 파이널라이저와 `Dispose` 메서드뿐이며, 여기서 `NativeMemory.Free`를 호출하여 해당 객체에 할당된 메모리를 해제합니다. 그런 다음 템플릿화된 모든 부분인 `{visibility}`, `{typeName}`, `{interfaceName}`, `{delegateCount}`, `{functionPointers}` 및 `{exports}`를 채워야 합니다.

먼저 쉬운 것부터 시작하세요:

```csharp
        var interfaceName = symbol.ToString();
        var typeName = $"{symbol.Name}";
        var visibility = symbol.DeclaredAccessibility.ToString().ToLower();

        // To be filled later
        int delegateCount = 0;
        var exports = new StringBuilder();
        var functionPointers = new StringBuilder();
```

`MyProfiler.ICorProfilerCallback` 인터페이스의 경우, `NativeObjects.ICorProfilerCallback` 유형의 래퍼를 생성합니다. 그렇기 때문에 정규화된 이름은 `interfaceName`(= `MyProfiler.ICorProfilerCallback`)에 저장하고 유형 이름만 `typeName`(= `ICorProfilerCallback`)에 저장합니다.

그런 다음 내보내기 목록과 해당 함수 포인터를 생성하고 싶습니다. 소스 생성기가 상속을 지원하여 `ICorProfilerCallback13`이 `ICorProfilerCallback12`를 구현하고, 그 자체로 `ICorProfilerCallback11`을 구현하는 등의 코드 중복을 피하고 싶습니다. 따라서 대상 인터페이스가 상속하는 인터페이스 목록을 추출하고 각 인터페이스에 대한 메서드를 추출합니다:

```csharp
        var interfaceList = symbol.AllInterfaces.ToList();
        interfaceList.Reverse();
        interfaceList.Add(symbol);

        foreach (var @interface in interfaceList)
        {
            foreach (var member in @interface.GetMembers())
            {
                if (member is not IMethodSymbol method)
                {
                    continue;
                }

                // TODO: Inspect the method
            }
        }
```

`QueryInterface(in Guid guid, out IntPtr ptr)` 메서드의 경우, 생성할 내보내기는 다음과 같습니다:

```csharp
[UnmanagedCallersOnly]
public static int QueryInterface(IntPtr* self, Guid* __arg1, IntPtr* __arg2)
{
    var handleAddress = *(self + 1);
    var handle = GCHandle.FromIntPtr(handleAddress);
    var obj = (IUnknown)handle.Target;

    var result = obj.QueryInterface(*__arg1, out var __local2);

    *__arg2 = __local2;

    return result;
}
```

메서드가 인스턴스 메서드이므로 `IntPtr* self` 인수를 추가합니다. 또한 관리 인터페이스의 함수가 `in`/`out`/`ref` 키워드로 장식된 경우, `UnmanagedCallersOnly` 메서드는 `in`/`out`/`ref`를 지원하지 않으므로 인수를 포인터 타입으로 선언합니다.

내보내기를 생성하는 결과 코드는 다음과 같습니다:

```csharp
var parameterList = new StringBuilder();

parameterList.Append("IntPtr* self");

foreach (var parameter in method.Parameters)
{
    var isPointer = parameter.RefKind == RefKind.None ? "" : "*";
    parameterList.Append($", {parameter.Type}{isPointer} __arg{parameter.Ordinal}");
}

exports.AppendLine($"            [UnmanagedCallersOnly]");
exports.AppendLine($"            public static {method.ReturnType} {method.Name}({parameterList})");
exports.AppendLine($"            {{");
exports.AppendLine($"                var handle = GCHandle.FromIntPtr(*(self + 1));");
exports.AppendLine($"                var obj = ({interfaceName})handle.Target;");
exports.Append($"                ");

if (!method.ReturnsVoid)
{
    exports.Append("var result = ");
}

exports.Append($"obj.{method.Name}(");

for (int i = 0; i < method.Parameters.Length; i++)
{
    if (i > 0)
    {
        exports.Append(", ");
    }

    if (method.Parameters[i].RefKind == RefKind.In)
    {
        exports.Append($"*__arg{i}");
    }
    else if (method.Parameters[i].RefKind is RefKind.Out)
    {
        exports.Append($"out var __local{i}");
    }
    else
    {
        exports.Append($"__arg{i}");
    }
}

exports.AppendLine(");");

for (int i = 0; i < method.Parameters.Length; i++)
{
    if (method.Parameters[i].RefKind is RefKind.Out)
    {
        exports.AppendLine($"                *__arg{i} = __local{i};");
    }
}

if (!method.ReturnsVoid)
{
    exports.AppendLine($"                return result;");
}

exports.AppendLine($"            }}");

exports.AppendLine();
exports.AppendLine();
```

함수 포인터의 경우 이전과 동일한 방법이 주어지면 생성하려고 합니다:

```csharp
*(vtable + 1) = (IntPtr)(delegate* unmanaged<IntPtr*, Guid*, IntPtr*>)&Exports.QueryInterface;
```

이를 생성하는 코드는 다음과 같습니다:

```csharp
var sourceArgsList = new StringBuilder();
sourceArgsList.Append("IntPtr _");

for (int i = 0; i < method.Parameters.Length; i++)
{
    sourceArgsList.Append($", {method.Parameters[i].OriginalDefinition} a{i}");
}

functionPointers.Append($"            *(vtable + {delegateCount}) = (IntPtr)(delegate* unmanaged<IntPtr*");

for (int i = 0; i < method.Parameters.Length; i++)
{
    functionPointers.Append($", {method.Parameters[i].Type}");

    if (method.Parameters[i].RefKind != RefKind.None)
    {
        functionPointers.Append("*");
    }
}

if (method.ReturnsVoid)
{
    functionPointers.Append(", void");
}
else
{
    functionPointers.Append($", {method.ReturnType}");
}

functionPointers.AppendLine($">)&Exports.{method.Name};");

delegateCount++;
```

인터페이스의 모든 메서드에 대해 이 작업을 수행한 후에는 템플릿의 값을 바꾸고 생성된 소스 파일을 추가하기만 하면 됩니다:

```csharp
sourceBuilder.Replace("{typeName}", typeName);
sourceBuilder.Replace("{visibility}", visibility);
sourceBuilder.Replace("{exports}", exports.ToString());
sourceBuilder.Replace("{interfaceName}", interfaceName);
sourceBuilder.Replace("{delegateCount}", delegateCount.ToString());
sourceBuilder.Replace("{functionPointers}", functionPointers.ToString());

context.AddSource($"{symbol.ContainingNamespace?.Name ?? "_"}.{symbol.Name}.g.cs", sourceBuilder.ToString());
```

이제 소스 생성기가 준비되었습니다.

## 생성된 코드 사용

소스 생성기를 사용하려면 `IUnknown`, `IClassFactory` 및 `ICorProfilerCallback` 인터페이스를 선언하고 `[NativeObject]` 특성으로 장식하면 됩니다:

```csharp
[NativeObject]
public interface IUnknown
{
    HResult QueryInterface(in Guid guid, out IntPtr ptr);
    int AddRef();
    int Release();
}
```

```csharp
[NativeObject]
internal interface IClassFactory : IUnknown
{
    HResult CreateInstance(IntPtr outer, in Guid guid, out IntPtr instance);
    HResult LockServer(bool @lock);
}
```

```csharp
[NativeObject]
public unsafe interface ICorProfilerCallback : IUnknown
{
    HResult Initialize(IntPtr pICorProfilerInfoUnk);

    // 70+ methods, stripped for brevity
}
```

그런 다음 `IClassFactory`를 구현하고 `NativeObjects.IClassFactory.Wrap`을 호출하여 네이티브 래퍼를 생성하고 `ICorProfilerCallback`의 인스턴스를 노출합니다:

```csharp
public unsafe class ClassFactory : IClassFactory
{
    private NativeObjects.IClassFactory _classFactory;
    private CorProfilerCallback2 _corProfilerCallback;

    public ClassFactory()
    {
        _classFactory = NativeObjects.IClassFactory.Wrap(this);
    }

    // The native wrapper has an implicit cast operator to IntPtr
    public IntPtr Object => _classFactory;

    public HResult CreateInstance(IntPtr outer, in Guid guid, out IntPtr instance)
    {
        Console.WriteLine("[Profiler] ClassFactory - CreateInstance");

        _corProfilerCallback = new();
        
        instance = _corProfilerCallback.Object;
        return HResult.S_OK;
    }

    public HResult LockServer(bool @lock)
    {
        return default;
    }

    public HResult QueryInterface(in Guid guid, out IntPtr ptr)
    {
        Console.WriteLine("[Profiler] ClassFactory - QueryInterface - " + guid);

        if (guid == KnownGuids.ClassFactoryGuid)
        {
            ptr = Object;
            return HResult.S_OK;
        }

        ptr = IntPtr.Zero;
        return HResult.E_NOTIMPL;
    }

    public int AddRef()
    {
        return 1; // TODO: do actual reference counting
    }

    public int Release()
    {
        return 0; // TODO: do actual reference counting
    }
}
```

그리고 `DllGetClassObject`에 노출합니다:

```csharp
public class DllMain
{
    private static ClassFactory Instance;

    [UnmanagedCallersOnly(EntryPoint = "DllGetClassObject")]
    public static unsafe int DllGetClassObject(void* rclsid, void* riid, nint* ppv)
    {
        Console.WriteLine("[Profiler] DllGetClassObject");

        Instance = new ClassFactory();
        *ppv = Instance.Object;

        return 0;
    }
}
```

마지막으로 `ICorProfilerCallback`의 인스턴스를 구현할 수 있습니다:

```csharp
public unsafe class CorProfilerCallback2 : ICorProfilerCallback2
{
    private static readonly Guid ICorProfilerCallback2Guid = Guid.Parse("8a8cc829-ccf2-49fe-bbae-0f022228071a");

    private readonly NativeObjects.ICorProfilerCallback2 _corProfilerCallback2;

    public CorProfilerCallback2()
    {
        _corProfilerCallback2 = NativeObjects.ICorProfilerCallback2.Wrap(this);
    }

    public IntPtr Object => _corProfilerCallback2;

    public HResult Initialize(IntPtr pICorProfilerInfoUnk)
    {
        Console.WriteLine("[Profiler] ICorProfilerCallback2 - Initialize");

        // TODO: To be implemented in next article

        return HResult.S_OK;
    }

    public HResult QueryInterface(in Guid guid, out IntPtr ptr)
    {
        if (guid == ICorProfilerCallback2Guid)
        {
            Console.WriteLine("[Profiler] ICorProfilerCallback2 - QueryInterface");

            ptr = Object;
            return HResult.S_OK;
        }

        ptr = IntPtr.Zero;
        return HResult.E_NOTIMPL;
    }

    // Stripped for brevity: the default implementation of all 70+ methods of the interface
    // Automatically generated by the IDE
}
```

테스트 애플리케이션으로 실행하면 기능이 예상대로 작동하는 것을 확인할 수 있습니다:

```bash
[Profiler] DllGetClassObject
[Profiler] ClassFactory - CreateInstance
[Profiler] ICorProfilerCallback2 - QueryInterface
[Profiler] ICorProfilerCallback2 - Initialize
Hello, World!
```

다음 단계에서는 퍼즐의 마지막 퍼즐 조각인 `ICorProfilerCallback.Initialize` 메서드를 구현하고 `ICorProfilerInfo` 인스턴스를 검색하는 작업을 처리합니다. 그러면 프로파일러 API와 실제로 상호 작용하는 데 필요한 모든 것을 갖추게 됩니다.

---

%[https://minidump.net/writing-a-net-profiler-in-c-part-3-7d2c59fc017f]