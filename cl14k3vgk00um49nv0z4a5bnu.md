---
title: "소스 생성기 만들기 8부 - 소스 생성기 '마커 특성' 문제 해결 - 2부"
datePublished: Thu Mar 24 2022 05:31:05 GMT+0000 (Coordinated Universal Time)
cuid: cl14k3vgk00um49nv0z4a5bnu
slug: 8-2
tags: dotnet

---

> **이 글은 Andrew Lock님의 [소스 생성기 만들기](https://andrewlock.net/series/creating-a-source-generator/) 연재를 번역한 글입니다.**

이것은 [소스 생성기 만들기](https://andrewlock.net/series/creating-a-source-generator/)의 여덟 번째 게시물 입니다.

이전 게시물에서 마커 특성, 소스 생성기에서 마커 특성을 사용하는 방법, 사용자 프로젝트에서 마커 특성을 참조하는 방법을 결정하는 문제에 대해 설명했습니다. 이 게시물에서는 내가 시도한 몇 가지 접근 방식과 내가 결정한 최종 접근 방식에 대해 설명합니다. 


## 외부 dll에서 마커 특성 참조 

요약하자면 마커 특성은 소스 생성기가 코드 생성에 사용해야 하는 유형을 제어하는 데 사용되는 간단한 특성이며 소스 생성기에 옵션을 전달하는 방법을 제공합니다. 

예를 들어 [내 StronglyTypedId](https://github.com/andrewlock/StronglyTypedId) 프로젝트를 사용하면 `[StronglyTypedId]` 특성으로 `struct`를 장식할 수 있습니다. 소스 생성기는 해당 특성의 존재를 사용하여 구조체에 대한 형식 변환기 및 속성 생성을 트리거합니다. 

마찬가지로 Microsoft.Extensions.Logging.Abstractions의 `[LoggerMessage]` 특성은 [효율적인 로그 인프라를 생성하는 데 사용](https://andrewlock.net/exploring-dotnet-6-part-8-improving-logging-performance-with-source-generators/)됩니다. 

문제는 마커 특성이 어디에 있어야 합니까? 이전 게시물에서 세 가지 옵션을 설명했습니다. 

- 소스 생성기에 의해 컴파일에 추가되었습니다.
- 사용자가 수동으로 생성합니다.
- 참조된 dll에 포함되어 있습니다. 

옵션 1.이 표준 접근 방식이지만 사용자가 `[InternalsVisibleTo]`를 사용하는 경우 동일한 유형을 여러 번 정의할 수 있으므로 작동하지 않습니다. 이 게시물에서는 옵션 3의 변형을 살펴봅니다. 이러한 변형은 이 문제를 스스로 해결하려고 시도하면서 시도한 것과 거의 같은 순서입니다. 


## 1. 빌드 출력을 직접 참조 

첫 번째 옵션은 단순성 측면에서 훌륭합니다. 일반적으로 분석기/소스 생성기 dll은 생성기 패키지를 프로젝트에 추가할 때 일반적인 방식으로 참조되지 않습니다. 이 접근 방식으로 우리는 그것을 바꿉니다! 

이것의 아름다움은 그것이 얼마나 단순한지입니다. 소스 생성기 프로젝트 내에 특성을 생성하고 일반적으로 소스 생성기에 있는 <IncludeBuildOutput>false</IncludeBuildOutput> 재정의를 제거하기만 하면 됩니다. 예를 들어: 

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <!-- 👇 이것을 추가하지 않음으로 dll은 빌드 출력으로 끝납니다.  -->
    <!-- <IncludeBuildOutput>false</IncludeBuildOutput> -->
  </PropertyGroup>

  <!-- 표준 소스 생성기 참조 -->
  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.3.3" PrivateAssets="all" />
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.0.1" PrivateAssets="all" />
  </ItemGroup>

  <!-- 빌드 출력을 NuGet 패키지의 "분석기" 슬롯에 패키징 -->
  <ItemGroup>
    <None Include="$(OutputPath)\$(AssemblyName).dll" Pack="true" PackagePath="analyzers/dotnet/cs" Visible="false" />
  </ItemGroup>
</Project>
```

생성기 프로젝트를 한 번만 수정하면 되는데, 지금까지는 너무 좋았습니다! 이것을 NuGet 패키지로 포함하면  analyzers/dotnet/cs 경로(소스 생성기에 필요)와 일반 lib 폴더 모두에 추가되어 소비 프로젝트에서 직접 참조할 수 있습니다. 

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1648094155071/64pXpqKL4.png)

NuGet 패키지의 소비자는 모두 생성기 dll에 포함된 마커 특성을 참조하므로 충돌하는 유형에 문제가 없습니다. 문제 해결됨! 

테스트 목적으로 또는 솔루션별 생성기가 있기 때문에 동일한 솔루션 내에서 소스 생성기 프로젝트를 참조하는 경우 소비 프로젝트의 `<ProjectReference>` 요소에서 `ReferenceOutputAssembly="true"`를 설정해야 합니다. 예를 들어: 

```xml
<ItemGroup>
  <ProjectReference Include="..\StronglyTypedId\StronglyTypedId.csproj" 
    OutputItemType="Analyzer" 
    ReferenceOutputAssembly="true" /> <!-- 👈 일반적으로 false -->
</ItemGroup>
```

그럼 문제는 해결된거 맞죠? 음… 아마도. 그러나 나는 이 접근 방식을 별로 좋아하지 않습니다. 귀하의 생성기 dll은 이제 사용자 참조의 일부가 되어 기분이 이상합니다. Microsoft.CodeAnalysis.CSharp 종속성 등에 대한 잠재적인 문제도 있습니다. 예를 들어, 내 테스트에서 내 프로젝트는 정상적으로 빌드되지만 System.Collections.Immutable의 일치하지 않는 버전에 대한 경고가 많았습니다. 

```
warning MSB3277: Found conflicts between different versions of "System.Collections.Immutable" that could not be resolved.
warning MSB3277: There was a conflict between "System.Collections.Immutable, Version=1.2.5.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a" and "System.Collections.Immutable, Version=5.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a". 
```

내 프로젝트 중 어느 것도 System.Collections.Immutable을 직접 참조하지 않았지만 생성기에서 사용하는 전이 참조이므로 문제가 발생합니다. 문제의 가능성은 내 취향에 비해 너무 커서 이 문제를 제쳐두고 다른 접근 방식을 시도했습니다. 


## 2. dll 전용 NuGet 패키지 만들기 

소스 생성기 dll 및 이에 의존하는 모든 관련 종속성을 참조하는 대신 마커 특성(및 관련 유형)만 포함하는 작은 dll이 필요합니다. 그런 다음 논리적 단계는 이러한 마커 유형만 포함하는 NuGet 패키지를 만드는 것입니다. 그런 다음 생성기 프로젝트에 종속성을 추가하여 특성 프로젝트를 소비 프로젝트에 추가할 때 생성기 프로젝트가 소비 패키지에도 자동으로 추가되도록 할 수 있습니다. 

이 접근 방식에 대한 나의 주요 관심사는 기술적인 어려움과 관련이 없습니다. 대신에 내 관심사는 이름 지정과 추악한 느낌에 더 많이 놓였습니다. 

> 밝혀진 바와 같이, 나는 이것에 약간의 기술적인 어려움이 있었지만 이것은 내 생각에 내 프로젝트의 세부 사항에 더 가깝기 때문에 이것이 진정한 장애물이라고 생각하지 않습니다. 

예를 들어 StronglyTypedId 프로젝트를 사용합니다. "마커 특성" 패키지를 StronglyTypedId.Attributes라고 하고 "생성기" 패키지를 StronglyTypedId라고 해야 합니까? 이는 사용자가 StronglyTypedId 패키지를 추가한 다음 생성기가 작동하지 않는 것으로 보이는 이유를 이해하지 못할 가능성이 높습니다(마커 특성에 대한 참조가 없기 때문에). 

또는 마커 특성 패키지 StronglyTypedId를 호출하고 소스 생성기 패키지 StronglyTypedId.Generator를 호출할 수 있습니다. 계층 구조가 더 잘 작동하는 것처럼 느껴지지만 여전히 누군가가 특성 없이 생성기 패키지를 추가하려는 것처럼 느껴집니다. 그들이 원하는 것은 결국 생성기이며 특성은 부산물입니다! 문서는 훌륭하지만 사람들은 그것을 읽지 않습니다 😉 


## 3. 추가 속성 패키지를 선택 사항으로 만들기 

이전 솔루션이 거의 맞는 것처럼 느껴졌지만 사용자가 항상 두 가지 다른 패키지에 대해 생각해야 한다는 사실이 마음에 들지 않았습니다. 이 문제를 만지작거리면서 잠재적으로 프로젝트 사용자의 작은 하위 집합에 대한 문제를 해결하려고 노력하고 있다는 것을 깨달았습니다. 

[이전 게시물](https://andrewlock.net/creating-a-source-generator-part-7-solving-the-source-generator-marker-attribute-problem-part1/)에서 언급했듯이 소스 생성기와 함께 마커 특성을 사용하는 "표준" 방법이 있습니다. 소스 생성기는 초기화 단계의 일부로 마커 속성을 자체적으로 추가합니다. 이것은 사용자에게 `[InternalsVisibleTo]` 속성이 있고 여러 프로젝트에서 소스 생성기를 사용하는 경우를 제외하고 잘 작동합니다. 

어떤 경우에 소스 생성기 초기화 단계를 사용하여 특성을 자동으로 추가하고 문제가 발생하는 사용자를 위해 별도의 속성 패키지를 제공하지 않겠습니까? 

이것은 사용자의 99%가 평소처럼 자동 추가 특성을 사용하는 단일 패키지를 갖고 다른 것에 대해 걱정할 필요가 없다는 것을 의미합니다. 기본 생성기 패키지는 StronglyTypedId라고 하고 추가 속성 패키지는 StronglyTypedId.Attributes라고 합니다. 계층 구조가 옳다고 느끼며 사람들은 (바라건대) 올바른 패키지를 향해 나아가고 있습니다. 

이 접근 방식의 문제는 `[InternalsVisibleTo]`를 실행하는 사용자가 자동 추가된 속성을 "끄는" 방법이 필요하다는 것입니다. 내가 생각할 수 있는 가장 좋은 방법은 생성된 속성 코드를 `#if/#endif`에 래핑하는 것입니다. 예를 들면 다음과 같습니다:

```csharp
#if !STRONGLY_TYPED_ID_EXCLUDE_ATTRIBUTES

using System;
namespace StronglyTypedIds
{
    [AttributeUsage(AttributeTargets.Struct, Inherited = false, AllowMultiple = false)]
    [System.Diagnostics.Conditional("STRONGLY_TYPED_ID_USAGES")]
    internal sealed class StronglyTypedIdAttribute : Attribute
    {
        public StronglyTypedIdAttribute(
            StronglyTypedIdBackingType backingType = StronglyTypedIdBackingType.Default,
            StronglyTypedIdConverter converters = StronglyTypedIdConverter.Default,
            StronglyTypedIdImplementations implementations = StronglyTypedIdImplementations.Default)
        {
            BackingType = backingType;
            Converters = converters;
            Implementations = implementations;
        }

        public StronglyTypedIdBackingType BackingType { get; }
        public StronglyTypedIdConverter Converters { get; }
        public StronglyTypedIdImplementations Implementations { get; }
    }
}
#endif
```

기본적으로 `STRONGLY_TYPED_ID_EXCLUDE_ATTRIBUTES` 변수는 설정되지 않으므로 특성은 컴파일의 일부가 됩니다. 사용자가 `[InternalsVisibleTo]` 문제에 부딪히면 프로젝트에서 이 상수를 정의할 수 있으며 포함된 생성 특성은 더 이상 컴파일의 일부가 아닙니다. 그런 다음 대신 StronglyTypedId.Attributes 패키지를 참조하여 생성기를 사용할 수 있습니다. 

```xml
 <Project Sdk="Microsoft.NET.Sdk">

   <PropertyGroup>
     <OutputType>Exe</OutputType>
     <TargetFramework>net6.0</TargetFramework>
    <!--  Define the MSBuild constant    -->
     <DefineConstants>STRONGLY_TYPED_ID_EXCLUDE_ATTRIBUTES</DefineConstants>
   </PropertyGroup>

  <PackageReference Include="StronglyTypedId" Version="1.0.0" PrivateAssets="All"/>
  <PackageReference Include="StronglyTypedId.Attributes" Version="1.0.0" PrivateAssets="All" />

 </Project>
```

이 접근 방식의 주요 이점은 대부분의 사용자가 추가 패키지에 대해 걱정할 필요가 없다는 것입니다. 문제를 파고들 필요가 있는 경우에만 문서를 읽고 싶은 동기가 더 커집니다 😉 


## 4. dll을 생성기 패키지에 포함 

내가 트릭을 놓쳤다는 것을 깨달았던 것은 이전 접근 방식을 구현하고 출시한 직후였습니다. 사용자가 문제를 해결하기 위해 별도의 패키지를 설치하도록 요구하는 대신 생성기 패키지 내부에 속성 dll을 패키징하고 마커 특성의 자동 포함을 완전히 건너뛸 수 있습니다. 

> 이것은 `[LoggerMessage]` 생성기에서 사용하는 것과 동일한 접근 방식입니다. 참고로 그 프로젝트를 참고해서 이 지경에 이르렀다는 걸 깨달았을 때 나는 얼굴을 찡그렸다. 🤦‍♂️ 

최종 결과는 analyzers/dotnet/cs  폴더에 StronglyTypedId.dll "생성기" dll이 있는 다음과 같은 NuGet 패키지 레이아웃이므로 생성에 사용되며 마커 특성 dll StronglyTypedId.Attributes.dll은 lib 폴더, 사용자 코드에서 직접 참조합니다. 

> 제 경우에는 생성기 코드 내에서 마커 특성을 참조하고 싶기 때문에 StronglyTypedId.Attributes.dll도 analyzer/dotnet/cs에 포함되어 있습니다. 이는 모든 소스 생성기 프로젝트에 필요하지 않을 가능성이 큽니다. 

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1648098888671/tuEfDjxsp.png)

이 레이아웃을 달성하려면 `dotnet pack`이 dll을 올바른 위치에 배치하도록 약간의 csproj 마법이 필요했지만 너무 모호한 것은 아닙니다. 

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <IncludeBuildOutput>false</IncludeBuildOutput>
  </PropertyGroup>

  <!-- 표준 소스 생성기 참조 -->
  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.Analyzers" Version="3.3.3" PrivateAssets="all" />
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.0.1" PrivateAssets="all" />
  </ItemGroup>


  <!-- 생성기에서 특성을 참조하여 이에 대해 컴파일 -->
  <!-- NuGet에 종속성이 없도록 PrivateAssets를 지정해야 함 -->
  <ItemGroup>
    <ProjectReference Include="..\StronglyTypedIds.Attributes\StronglyTypedIds.Attributes.csproj" PrivateAssets="All" /> 
  </ItemGroup>

  <ItemGroup>
    <!-- analyzers/dotnet/cs 경로에 생성기 dll 포함 -->
    <None Include="$(OutputPath)\$(AssemblyName).dll" Pack="true" PackagePath="analyzers/dotnet/cs" Visible="false" />

    <!-- analyzers/dotnet/cs 경로에 특성 dll 포함 -->
    <None Include="$(OutputPath)\StronglyTypedIds.Attributes.dll" Pack="true" PackagePath="analyzers/dotnet/cs" Visible="false" />

    <!-- ib\netstandard2.0 경로에 dll 특성 포함 -->
    <None Include="$(OutputPath)\StronglyTypedIds.Attributes.dll" Pack="true" PackagePath="lib\netstandard2.0" Visible="true" />
  </ItemGroup>

</Project>
```
이 작업을 수행하는 "더 나은" 방법이 있을 수 있지만 이 방법이 효과적이었습니다. 

NuGet 패키지를 참조하는 경우 특별한 작업을 수행할 필요가 없습니다. 

```xml
<ItemGroup>
  <PackageReference Include="StronglyTypedId" Version="1.0.0" PrivateAssets="all" />
</ItemGroup>
```

여기에서 `PrivateAssets="all"`을 사용하여 다운스트림 프로젝트도 소스 생성기에 대한 참조를 가져오는 것을 방지했지만 이는 전적으로 선택 사항입니다. 한 가지 유의해야 할 점은 이렇게 하면 프로젝트의 bin 폴더에 마커 속성 dll StronglyTypedId.Attributes.dll이 표시된다는 것입니다. 그러나 속성 자체는 [조건부로 장식](https://andrewlock.net/conditional-compilation-for-ignoring-method-calls-with-the-conditionalattribute/#applying-the-conditional-attribute-to-classes)되어 있으므로 dll에 대한 런타임 종속성이 없습니다. 

[`<PackageReference>` 요소에서 `ExcludeAssets="runtime"`을 설정](https://docs.microsoft.com/en-us/nuget/consume-packages/package-references-in-project-files#controlling-dependency-assets)하여 dll이 출력에 복사되지 않도록 할 수 있습니다. 

```xml
<ItemGroup>
  <PackageReference Include="StronglyTypedId" Version="1.0.0" 
    PrivateAssets="all" ExcludeAssets="runtime" />
</ItemGroup>
```

이렇게 하면 여전히 마커 특성에 대해 컴파일할 수 있지만 dll은 bin 폴더에 없습니다. 

동일한 솔루션 내부에서 소스 생성기 프로젝트를 참조하는 경우 특성 프로젝트에도 일반 `<PackageReference>`를 추가해야 합니다. 제 경우에는 특성 dll에 대한 참조가 있어야 하는 소스 생성기와 대상 프로젝트가 모두 필요했기 때문에 조금 더 복잡했습니다. 

> 소스 생성기는 참조 측면에서 자체의 작은 거품에 살고 있습니다. 소비 프로젝트에 특성 프로젝트에 대한 참조가 있더라도 소스 생성기는 해당 프로젝트 또는 소비 프로젝트의 다른 참조에 액세스할 수 없습니다. 

약간 혼란스럽긴 하지만 [소스 생성기 프로젝트가 소비 프로젝트의 속성 dll에 액세스하려면 속성 프로젝트를 분석기로 처리하도록 소비 프로젝트에 지시](https://github.com/dotnet/roslyn/discussions/47517#discussioncomment-1633510)해야 합니다. 그러면 소스 생성기 "분석기"가 이를 참조하고 올바르게 생성할 수 있습니다. 소비하는 프로젝트가 마커 속성 dll도 참조하도록 하기 때문에 `ReferenceOutputAssembly="true"`로 설정해야 합니다. 

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <!-- 소스 생성기 프로젝트 다시 참조 -->
    <ProjectReference Include="..\StronglyTypedIds\StronglyTypedIds.csproj"
        OutputItemType="Analyzer" 
        ReferenceOutputAssembly="false" /> <!-- 생성기 dll을 참조하지 않음 -->

    <!-- 특성 프로젝트 "분석기로 취급" 다시 참조 -->
    <ProjectReference Include="..\StronglyTypedIds.Attributes\StronglyTypedIds.Attributes.csproj" 
        OutputItemType="Analyzer" 
        ReferenceOutputAssembly="true" /> <!-- dll 특성을 참조 -->
  </ItemGroup>
</Project>
```

이 최종 설정을 통해 세계 최고의 것을 손에 넣을 수 있을 것 같습니다:

- 단일 NuGet 패키지만 걱정할 수 있습니다.
- 사용자가 `[InternalsVisibleTo]`를 사용할 때 문제가 없습니다.
- 사용자는 `ExcludeAssets="runtime"`을 사용하여 빌드 출력에서 마커 dll을 제외할 수 있습니다.
- 사용자는 `dotnet add package StronglyTypedId`를 수행할 수 있으며 그냥 작동합니다. 추가 `<PackageReference>` 속성은 순전히 선택 사항입니다. 


## 보너스: 원하는 경우 특성을 포함하세요! 

StronglyTypedId의 경우 실제로 한 단계 더 나아가 MSBuild 변수 `STRONGLY_TYPED_ID_EMBED_ATTRIBUTES`를 설정하여 소스 생성기를 사용하여 프로젝트의 dll에 특성을 포함하도록 사용자가 선택하도록 허용했습니다. 특성은 항상 컴파일에 추가되지만 다음과 같이 설정하지 않으면 사용할 수 없습니다:

```csharp
#if STRONGLY_TYPED_ID_EMBED_ATTRIBUTES

using System;

namespace StronglyTypedIds
{
    [AttributeUsage(AttributeTargets.Struct, Inherited = false, AllowMultiple = false)]
    [System.Diagnostics.Conditional("STRONGLY_TYPED_ID_USAGES")]
    internal sealed class StronglyTypedIdAttribute : Attribute
    {
        // ...
    }
}
#endif
```

사용자가 이 기능을 켜면 소스 생성기에 의해 포함된 "내부" 유형과 dll 특성의 공개 유형이 있기 때문에 처음에는 중복 유형 문제가 발생합니다. 이 문제를 해결하기 위해 패키지의 `ExcludeAssets`에 `compile`을 추가할 수 있습니다. 

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
    <!-- 포함된 특성이 활성화되도록 이 상수를 정의  -->
    <DefineConstants>STRONGLY_TYPED_ID_EMBED_ATTRIBUTES</DefineConstants>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="StronglyTypedId" Version="1.0.0" 
        ExcludeAssets="compile;runtime" PrivateAssets="all" />
        <!-- 마커 특성 dll에 대해 컴파일 하지 않도록 ☝ 이것을 추가합니다. -->
  </ItemGroup>
</Project>
```

이제 누군가가 왜 그렇게 하고 싶어하는지 정말로 생각할 수 없지만 원래 접근 방식을 위해 작성된 코드가 이미 있으므로 필요한 사람을 위해 남겨 두었습니다! 😄 


## 요약

이 게시물에서는 소스 생성기의 마커 특성을 처리하는 방법을 결정하는 과정을 설명합니다. 4가지 주요 접근 방식을 설명했습니다. 소비 프로젝트에서 소스 생성기 dll을 직접 참조합니다. 두 개의 독립적인 NuGet 패키지 만들기 조건부 컴파일을 사용하여 마커 특성 NuGet 패키지를 선택 사항으로 만듭니다. 마커 특성 dll 및 생성기 dll을 동일한 NuGet 패키지에 포함합니다. 최종 옵션은 최상의 접근 방식처럼 보였고 사용자에게 가장 부드러운 경험을 제공합니다. 


## 원문
Andrew Lock's [Series: Creating a source generator](https://andrewlock.net/series/creating-a-source-generator/) - [Part 8 - Solving the source generator 'marker attribute' problem - Part 2](https://andrewlock.net/creating-a-source-generator-part-8-solving-the-source-generator-marker-attribute-problem-part2/)