## Avalonia 프로젝트 템플릿 11.0 미리보기 1 실행

Avalonia(이하 아발로니아)의 버전이 0.10에서 11.0로 높은 점프를 했습니다. 아발로니아 11.0 미리보기 1이 출시가 되었는데요 아래의 링크로 그 내용을 확인하실 수 있습니다.

%[https://dev.to/avalonia/turning-it-up-to-11-34jn]

그런데 아직 [아발로니아 프로젝트 템플릿](https://github.com/AvaloniaUI/avalonia-dotnet-templates)이 `1.0.0-preview1`을 지원하지 않아 바로 프로젝트를 생성할 수 없습니다.

기존 템플릿으로 프로젝트를 생성한 후, `csproj`를 다음처럼 수정합니다.

| 기존
```xml
...
    <PackageReference Include="Avalonia" Version="0.10.18" />
    <PackageReference Include="Avalonia.Desktop" Version="0.10.18" />
    <!--Condition below is needed to remove Avalonia.Diagnostics package from build output in Release configuration.-->
    <PackageReference Condition="'$(Configuration)' == 'Debug'" Include="Avalonia.Diagnostics" Version="0.10.18" />

...
```

| 변경
```xml
...
    <PackageReference Include="Avalonia" Version="11.0.0-preview1" />
    <PackageReference Include="Avalonia.Desktop" Version="11.0.0-preview1" />
    <!--Condition below is needed to remove Avalonia.Diagnostics package from build output in Release configuration.-->
    <PackageReference Condition="'$(Configuration)' == 'Debug'" Include="Avalonia.Diagnostics" Version="11.0.0-preview1" />
    <PackageReference Include="Avalonia.Themes.Fluent" Version="11.0.0-preview1" />
...
```

이후 잘 컴파일 되고 잘 실행되는 것을 확인할 수 있습니다.

| 윈도우
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661268340139/r_jI2wsAK.png align="left")

| 우분투(WSLg)
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1661268368958/2fSgKmIyh.png align="left")

0.10에서 11.0-preview1로의 주요 변경 내용은 아래의 링크를 참고하세요.

%[https://github.com/AvaloniaUI/Avalonia/wiki/Breaking-Changes#010--110]
