## 소스 생성기 만들기 6부 - 소스 제어에 소스 생성기 출력 저장

> **이 글은 Andrew Lock님의 [소스 생성기 만들기](https://andrewlock.net/series/creating-a-source-generator/) 연재를 번역한 글입니다.**

이것은 [소스 생성기 만들기](https://andrewlock.net/series/creating-a-source-generator/)의 여섯 번째 게시물 입니다.

이 게시물에서는 소스 생성기의 출력을 디스크에 유지하여 소스 제어 및 코드 검토의 일부가 될 수 있도록 하는 방법, 파일이 출력되는 위치를 제어하는 방법, 소스 생성기가 대상 프레임워크에 따라 다른 출력을 생성하는 경우를 처리하는 방법에 대해 설명합니다.


 ## 소스 생성기는 기본적으로 아티팩트를 생성하지 않습니다. 

소스 생성기의 가장 큰 장점 중 하나는 컴파일러에서 실행된다는 것입니다. 따라서 별도의 빌드 단계가 필요하지 않으므로 t4 템플릿과 같은 다른 소스 생성 기술보다 더 편리합니다. 

그러나 한 가지 잠재적인 단점은 소스 생성기가 컴파일러 내에서 실행된다는 사실에서 비롯됩니다. 이는 IDE 컨텍스트에 있지 않을 때 소스 생성기의 효과를 보기 어렵게 만들 수 있습니다. 

예를 들어, 소스 생성기를 사용하는 GitHub의 pull 요청을 검토하고 프로젝트에 코드를 추가하는 변경을 수행하는 경우 해당 출력을 PR에 표시하는 것이 유용할 수 있습니다. 이것은 "중요한" 코드에 특히 중요할 수 있습니다. 

예를 들어 [Datadog Tracer](https://github.com/DataDog/dd-trace-dotnet)에서 우리는 최근 소스 생성기를 사용하여 활성화된 통합을 제어하는 프로파일러의 "네이티브" 부분에 의해 호출되는 메서드를 생성하기 시작했습니다. 이것은 추적기의 중요한 부분이므로 변경 사항을 확인하는 것이 중요합니다. 우리는 모든 변경 사항을 PR에서 볼 수 있기를 원했기 때문에 소스 생성기 출력이 파일에 기록되었는지 확인해야 했습니다. 


## 컴파일러 생성 파일 내보내기 

소스 생성기 파일을 파일 시스템에 유지할 수 있도록 하는 간단한 스위치가 있습니다: `EmitCompilerGeneratedFiles`. 프로젝트 파일에서 이 속성을 설정할 수 있습니다. 

```xml
<PropertyGroup>
    <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
</PropertyGroup>
```

또는 다른 방법으로 예를어 빌드할 때 명령줄에서 MSBuild 속성을 설정할 수 있습니다.

```shell
dotnet build /p:EmitCompilerGeneratedFiles=true
```

이 속성을 단독으로 설정하면 컴파일러가 힌트 파일을 디스크에 출력합니다. 예를 들어 [NetEscapades.EnumGenerators](https://github.com/andrewlock/NetEscapades.EnumGenerators) 패키지를 고려하고 `EmitCompilerGeneratedFiles` 속성을 활성화하면 생성된 소스 파일이 obj 폴더에 기록되는 것을 볼 수 있습니다:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1647267108514/XQdrpAHQo.png)

특히 소스 생성기 출력은 다음과 같이 정의된 폴더에 작성됩니다:

```
{BaseIntermediateOutpath}/generated/{Assembly}/{SourceGeneratorName}/{GeneratedFile}
```

위의 예에서 우리는

- `BaseIntermediateOutpath`: obj/Debug/net6.0
- Assembly: NetEscapades.EnumGenerators
- SourceGeneratorName: NetEscapades.EnumGenerators.EnumGenerator
- GeneratedFile: ColoursExtensions_EnumExtensions.g.cs, EnumExtensionsAttribute.g.cs 

obj 폴더에 파일을 쓰는 것은 모두 훌륭하지만 bin 및 obj 폴더가 일반적으로 소스 제어에서 제외되기 때문에 실제로 문제를 해결하지는 못합니다. 소스 제어에 명시적으로 포함할 수 있지만 더 나은 옵션은 파일을 다른 곳으로 내보내는 것입니다. 


## 출력 위치 제어 

`CompilerGeneratedFilesOutputPath` 속성을 설정하여 컴파일러에서 내보낸 파일의 위치를 제어할 수 있습니다. 이것은 프로젝트 루트 폴더에 대한 상대 경로입니다. 예를 들어 프로젝트 파일에서 다음을 설정하는 경우: 

```xml
<PropertyGroup>
    <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
    <CompilerGeneratedFilesOutputPath>Generated</CompilerGeneratedFilesOutputPath>
</PropertyGroup>
```

이렇게 하면 프로젝트 폴더의 생성된 폴더에 파일이 기록됩니다:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1647267338451/RNNYiT8TU.png)

`CompilerGeneratedFilesOutputPath`에 무엇을 배치하든 파일 경로의 `{BaseIntermediateOutpath}/generated` 접두사를 대체하므로 파일이 다음 위치에 기록됩니다:

```
{CompilerGeneratedFilesOutputPath}/{Assembly}/{SourceGeneratorName}/{GeneratedFile}
```

표면적으로는 이것이 모든 문제를 해결하는 것처럼 보입니다. 소스 생성기 내용은 소스 제어에 포함된 위치로 파일 시스템으로 내보내집니다. 문제가 해결되었나요? 

어려움은 파일이 이미 작성된 후 두 번째로 빌드하고 시도할 때 여러 오류가 발생한다는 것입니다. 

```
ColoursExtensions_EnumExtensions.g.cs(31,28): 오류 CS0111: 'ColoursExtensions' 유형은 이미 동일한 매개변수 유형 ColoursExtensions_EnumExtensions.g.cs(40,28)를 사용하여 'IsDefined'라는 멤버를 정의하고 있습니다. 오류 CS0111: 유형 'Colour' 동일한 매개변수 유형으로 'TryParse'라는 멤버를 정의합니다. 
```

컴파일러가 메모리 내 소스 생성기 출력과 함께 내보낸 파일을 포함하기 때문입니다. 이로 인해 위의 유형과 오류가 중복됩니다. 대답은 컴파일에서 파일을 제외하는 것입니다. 


## 컴파일에서 내보낸 파일 제외

이 문제에 대한 간단한 해결책은 프로젝트 컴파일에서 방출된 파일을 제거하여 메모리 내 소스 생성기 출력만 컴파일의 일부가 되도록 하는 것입니다. 개별적으로 제외할 수 있습니다(예: Visual Studio에서 파일을 마우스 오른쪽 버튼으로 클릭). 또는 [와일드카드 패턴](https://www.reddit.com/r/dotnet/comments/mrgx3u/how_to_put_source_generator_code_into_source/)을 사용하여 해당 폴더의 모든 .cs 파일을 제외할 수 있습니다. 

```xml
<PropertyGroup>
    <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
    <CompilerGeneratedFilesOutputPath>Generated</CompilerGeneratedFilesOutputPath>
</PropertyGroup>

<ItemGroup>
    <!-- 컴파일에서 소스 생성기의 출력 제외 -->
    <Compile Remove="$(CompilerGeneratedFilesOutputPath)/**/*.cs" />
</ItemGroup>
```

이 변경으로 이제 소스 생성기 출력이 디스크로 내보내지고 소스 제어에 포함되어 PR 등에서 검토할 수 있으며 컴파일 자체에 영향을 미치지 않습니다. 


## 대상 프레임워크로 분할 

위의 속성은 Datadog Tracer에 첫 번째 소스 생성기를 추가할 때 [처음에 사용한 것](https://github.com/DataDog/dd-trace-dotnet/blob/69403b8873b905230faeee2b3f6284f509517ecf/tracer/src/Datadog.Trace/Datadog.Trace.csproj#L15-L27)입니다. 그러나 이것은 이후에 우리에게 약간의 문제를 일으켰습니다. 

컨텍스트의 경우 Datadog Tracer는 현재 `net461`, `netstandard2.0`, `netcoreapp3.1`과 같은 여러 대상 프레임워크를 지원합니다. 그러나 일부 통합은 특정 대상 프레임워크에만 적용됩니다. 예를 들어 [ASP.NET 통합은 `net461`에만 적용되므로 `#if NETFRAMEWORK`를 사용하여 .NET Core 어셈블리에서 제외](https://github.com/DataDog/dd-trace-dotnet/blob/master/tracer/src/Datadog.Trace/ClrProfiler/AutoInstrumentation/AspNet/ApiController_ExecuteAsync_Integration.cs)합니다. 

어려운 점은 소스 생성기의 출력이 대상 프레임워크마다 다르지만 각 대상 프레임워크 컴파일의 출력은 모든 경우에 동일한 폴더에 작성된다는 것입니다. 컴파일러가 대상 프레임워크에 대해 실행할 때마다 Generated/AssemblyName/GeneratorName/FileName.cs의 기존 파일 출력을 덮어씁니다! 소스 생성기의 세 가지 다른 출력이지만 그 중 하나만 디스크에 유지됩니다. 

이 문제를 해결하기 위해 `$(TargetFramework)` 속성을 사용하여 출력 파일 경로에 대상 프레임워크를 추가했습니다. 

```xml
<PropertyGroup>
    <!-- 소스 생성기(및 기타) 파일을 디스크에 유지 -->
    <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
    <!-- 👇 소스 생성기의 "기본" 경로 -->
    <GeneratedFolder>Generated</GeneratedFolder>
    <!-- 👇 각 대상 프레임워크에 대한 출력을 다른 하위 폴더에 작성 -->
    <CompilerGeneratedFilesOutputPath>$(GeneratedFolder)\$(TargetFramework)</CompilerGeneratedFilesOutputPath>
</PropertyGroup>

<ItemGroup>
    <!-- 👇 기본 폴더의 모든 항목 제외 -->
    <Compile Remove="$(GeneratedFolder)/**/*.cs" />
</ItemGroup>
```

이 변경으로 인해 각 프레임워크에 대한 소스 생성기의 출력이 별도의 폴더에 작성되어 어셈블리 간의 차이점을 쉽게 확인할 수 있습니다. 

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1647267857576/N2sEwumtR.png)

분명히 이 접근 방식은 다중 대상을 지정하지 않고 다른 대상 프레임워크에 대해 다른 소스 생성기 출력을 생성하지 않는 한 필요하지 않지만, 만약 그렇다면 쉬운 접근 방식입니다. 


## 요약

이 게시물에서 소스 생성기가 생성된 출력을 디스크로 내보내도록 하는 방법을 설명했습니다. 이는 소스 생성기 출력의 변경 사항을 모니터링하거나 GitHub의 pull 요청과 같이 IDE가 아닌 시나리오에서 해당 출력을 검토할 수 있도록 하려는 경우에 유용할 수 있습니다. 그런 다음 파일이 기록되는 위치를 제어하는 방법과 소스 생성기가 프로젝트의 다른 대상 프레임워크 빌드에 대해 다른 출력을 생성하는 경우를 처리하는 한 가지 접근 방식을 보여주었습니다. 


## 원문
Andrew Lock's [Series: Creating a source generator](https://andrewlock.net/series/creating-a-source-generator/) - [Part 6 - Saving source generator output in source control ](https://andrewlock.net/creating-a-source-generator-part-6-saving-source-generator-output-in-source-control/)
