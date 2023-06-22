---
title: "소스 링크로 개선된 .NET 디버깅 환경 | Patrick Smacchia"
datePublished: Thu Jun 22 2023 01:31:01 GMT+0000 (Coordinated Universal Time)
cuid: clj6gvq9k000409lc1hzvc0et
slug: improved-net-debugging-experience-with-source-link
canonical: https://blog.ndepend.com/improved-net-debugging-experience-with-source-link/
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1687397482910/1f7dac37-c4ba-493c-9534-7b8bfa4a9c11.webp
tags: debugging, net, dotnet

---

> Patrick Smacchia님의 [Improved .NET Debugging Experience with Source Link](https://blog.ndepend.com/improved-net-debugging-experience-with-source-link/)를 번역하였습니다.

---

소스 링크(Source Link)는 .NET 개발자가 응용 프로그램에서 참조하는 NuGet 패키지의 **소스 코드**를 디버깅할 수 있도록 하는 Microsoft 기술입니다. 소스 코드에 밑줄을 그은 이유는 디버깅 시 사용자가 참조된 어셈블리 내의 IL 코드에서 디컴파일된 C# 코드만 가져오는 것이 아니기 때문입니다. 대신 개발자는 라이브러리를 빌드하는 데 사용된 **원본 C# 소스 코드**를 디버깅하며, 원본 주석과 서식을 사용할 수 있습니다.

소스 링크는 언어, IDE 및 소스 제어에 구애받지 않습니다. 소스 링크는 Visual Studio, VS Code 또는 Rider IDE를 통해 수행한 디버깅 세션에서 사용할 수 있습니다.

## Visual Studio에서 소스 링크를 사용한 디버깅

[Newtonsoft.Json](https://www.nuget.org/packages/Newtonsoft.Json/) 패키지는 소스 링크를 지원합니다. 즉, 아래 프로그램을 디버깅할 때 `SerializeObject()` 메서드의 원본 소스 코드에 들어갈 수 있습니다.

```csharp
var person = new Person { FirstName = "Albert", LastName = "Einstein ", YearOfBirth = 1879 };
string json = Newtonsoft.Json.JsonConvert.SerializeObject(person);
Console.WriteLine(json);

public class Person {
   public string FirstName { get; init; }
   public string LastName { get; init; }
   public int YearOfBirth { get; init; }
}
```

이 작업을 수행하기 전에 Visual Studio에서 몇 가지 설정을 설정해야 합니다:

* 활성화: 도구 &gt; 옵션 &gt; 디버깅 &gt; 기호 &gt; Nuget.org 기호 서버
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687395998947/55e6d2c8-a34d-4af5-bfa1-cf38a328839f.png align="center")

* 도구 &gt; 옵션 &gt; 디버깅 &gt; 일반
    
    * 비활성화: 내 코드만 사용
        
    * 활성화: 소스 서버 지원 활용, 소스 링크 지원 사용
        

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1687396199490/a5c2a701-4d8c-4f74-905d-b4442fbbf013.png align="center")

그런 다음 프로그램 디버깅을 시작합니다(F5). 디버깅 세션을 시작하기 전에 참조된 패키지의 심볼을 로드하는 데 몇 초 정도 걸립니다:

![Source Link Load Symbol](https://blog.ndepend.com/wp-content/uploads/Source-Link-Load-Symbol.png align="left")

이제 Newtonsoft의 원본 소스를 단계별로 살펴볼 수 있습니다:

![Source Link Setp Into](https://blog.ndepend.com/wp-content/uploads/Source-Link-Setp-Into2.gif align="left")

`JsonSerializer.cs` 파일은 GitHub에서 다운로드하여 `C:\사용자\usr\AppData\Local\SourceServer\634989e5314d385c82c8a3269e286aacd49e4527b9c315d6bd38afcf4ee504e3\Src\Newtonsoft.Json` 폴더에 저장되어 있습니다.

![Source Link repository type](https://blog.ndepend.com/wp-content/uploads/Source-Link-repository-type.png align="left")

이 시나리오에서는 압축된 패키지에 PDB 파일이 포함되지 않습니다. 디버그 시 이 위치와 커밋 해시는 소스 파일과 PDB 파일을 GitHub에서 가져오는 데 사용되며, 이 파일은 `\Temp\SymbolCache` 디렉터리에 로컬로 저장됩니다.

![Source Link symbols loaded](https://blog.ndepend.com/wp-content/uploads/Source-Link-symbols-loaded.png align="left")

## 자체 패키지에 소스 링크 활성화하기

자체 패키지에 소스 링크를 활성화하면 이를 사용하는 개발자의 작업이 훨씬 쉬워집니다.

최소한 `.csproj` 프로젝트 파일에 `<PublishRepositoryUrlsproj>` 태그와 소스 링크 패키지에 대한 참조를 추가해야 합니다.

```xml

<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net7.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
 
    <!-- Insert the tag <repositoy .../> in the .nuspec file -->
    <PublishRepositoryUrl>true</PublishRepositoryUrl>
  </PropertyGroup>
 
  <ItemGroup>
    <!-- Reference this package to enable SourceLink used with the source control GitHub -->
    <PackageReference Include="Microsoft.SourceLink.GitHub" Version="1.1.1">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
  </ItemGroup>
</Project>
```

GitLab 또는 Bitbucket을 사용하는 경우 대신 [Microsoft.SourceLink.GitLab](https://www.nuget.org/packages/Microsoft.SourceLink.GitLab) 또는 [Microsoft.SourceLink.Bitbucket.Git](https://www.nuget.org/packages/Microsoft.SourceLink.Bitbucket.Git) 패키지를 참조할 수 있습니다.

### PDB 파일 포함하기

일부 선택적 태그를 사용하면 기본 **.nupkg** 패키지 내부 또는 확장자가 **.snupkg**인 사이드 패키지에 PDB 파일을 포함할 수 있습니다. 첫 번째 옵션은 심볼이 필요하지 않은 시나리오에서도 메인 패키지의 크기가 증가하여 복원 시간이 길어지므로 권장되지 않습니다.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    ...
 
    <!-- Embed symbol files (*.pdb) in the .nupkg package -->
    <AllowedOutputExtensionsInPackageBuildOutputFolder>$(AllowedOutputExtensionsInPackageBuildOutputFolder);.pdb</AllowedOutputExtensionsInPackageBuildOutputFolder>
 
    <!-- or embed symbol package in a symbol package (.snupkg) -->
    <IncludeSymbols>true</IncludeSymbols>
    <SymbolPackageFormat>snupkg</SymbolPackageFormat>
  </PropertyGroup>
...
```

NuGet.org는 자체 심볼 서버 리포지토리를 호스팅하며, 이 리포지토리는 **.snupkg** 심볼 사이드 패키지와 함께 작동합니다. 이 글의 첫 번째 섹션에서 **도구 &gt; 옵션 &gt; 디버깅 &gt; 심볼 &gt; NuGet.org 심볼 서버**를 선택했을 때 Visual Studio에 NuGet.org 심볼 서버 리포지토리에서 심볼을 가져오도록 지시했습니다.

### PDB 파일에 소스 포함

마지막으로 이러한 옵션을 사용하여 PDB 파일 내에 소스 파일을 선택적으로 포함하도록 결정할 수 있습니다:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    ...

    <!-- Embed all project source files into the generated PDB -->
    <EmbedAllSources>true</EmbedAllSources>

    <!-- Embed project source files that are not tracked by the source control or imported from a source package to the generated PDB.
         Has no effect if EmbedAllSources is true. -->
    <EmbedUntrackedSources>true</EmbedUntrackedSources>
  </PropertyGroup>
...
```

## 마무리

소스 링크를 사용하면 디버깅 프로세스가 더욱 원활하고 직관적이며 통찰력 있게 진행되어 개발 주기를 단축할 수 있습니다. 소스 링크는 NuGet 에코시스템에 원활하게 통합되어 개발자가 레퍼런스의 내부를 자세히 살펴볼 수 있으므로 전례 없는 효율성으로 문제와 이슈를 해결할 수 있습니다.

---

%[https://blog.ndepend.com/improved-net-debugging-experience-with-source-link/]