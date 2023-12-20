---
title: "ASP.NET Core의 multipart/form-data 데이터 섹션에서 JSON 및 바이너리 데이터 읽기 | Andrew Lock"
datePublished: Wed Dec 20 2023 08:01:51 GMT+0000 (Coordinated Universal Time)
cuid: clqdhjiy0000r08jrbtsn8aie
slug: aspnet-core-multipartform-data-json-andrew-lock
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1703078228810/cf6984a6-d15c-4a52-96da-37f23f784adb.png
tags: dotnet

---

> 본 글은 Andrew Lock님의 글 [Reading JSON and binary data from multipart/form-data sections in ASP.NET Core](https://andrewlock.net/reading-json-and-binary-data-from-multipart-form-data-sections-in-aspnetcore/)을 번역한 것입니다.

이 글에서는 ASP.NET Core의 `multipart/form-data` 요청에서 JSON과 바이너리 데이터를 모두 읽는 방법을 설명합니다. 직장 동료가 이 기능이 필요했는데, ASP.NET Core의 "일반적인" 메커니즘을 사용하여 이 기능을 수행할 방법을 찾을 수 없었습니다. 이 게시물은 우리가 결국 찾은 접근 방식에 대해 설명합니다.

## `multipart/form-data`가 무엇인가요?

자세한 내용을 살펴보기 전에 `multipart/form-data`가 어떻게 생겼는지, 다른 요청 유형과 어떻게 다른지 알아볼 필요가 있습니다. 최신 웹에서는 기존 HTTP 요청을 전송하는 두 가지 주요 접근 방식이 있습니다.

* JSON 데이터
    
* 양식 데이터
    

> 예, 압니다. XML, 웹소켓을 통한 바이너리 데이터, gRPC 등 다른 형식도 많이 있다는 것을 알고 있습니다. 하지만 대부분의 사람들이 일상적인 앱에서 JSON 또는 양식 데이터를 사용하고 있을 것입니다.

JSON 데이터는 일반적으로 프로그래밍 방식으로 API에 데이터를 전송할 때 사용됩니다. 백엔드 앱에 요청을 보내는 JavaScript 또는 Blazor 클라이언트일 수도 있고, 서버 측 앱이 다른 API에 `HttpClient` 요청을 보내는 것일 수도 있습니다.

반면에 양식 데이터는 MVC, 레이저 페이지 또는 일반 HTML로 빌드하는 것과 같이 "전통적인" 서버 렌더링 애플리케이션의 일반적인 데이터 형식입니다.

HTTP 요청으로 데이터를 전송할 때는 데이터 유형을 지정해야 합니다. JSON 요청의 경우 `application/json`이지만 양식 데이터의 경우 두 가지 가능성이 있습니다.

* `application/x-www-form-urlencoded`
    
* `multipart/form-data`
    

기본적으로 HTML `<form>` 요소를 만들고 기본 제공 브라우저 기능을 사용하여 서버에 게시하는 경우 양식은 `application/x-www-form-urlencoded` 데이터로 전송됩니다. 예를 들어 다음과 같은 양식입니다.

```xml
<form action="/handle-form" method="POST">
  <input type="text" name="Name" />
  <input type="date" name="DueDate" />
  <input type="checkbox" name="IsCompleted" />
  <input type="submit" />
</form>
```

전송하면 다음과 같은 HTTP 요청이 전송됩니다.

```http
POST /handle-form HTTP/2
Host: andrewlock.net
accept: text/html,*/*
upgrade-insecure-requests: 1
content-type: application/x-www-form-urlencoded
content-length: 50

Name=Andrew+Lock&DueDate=2024-08-01&IsCompleted=on
```

데이터는 "URL 인코딩" 형식으로 본문에 포함되므로 공백은 `+`로 인코딩되고 필드는 `&`를 사용하여 연결됩니다. 이 형식은 데이터를 전송하는 데 매우 간결한 형식이지만 몇 가지 제한 사항이 있습니다. 가장 눈에 띄게 누락된 기능 중 하나는 양식을 사용하여 파일을 제출하는 기능입니다. 요청 본문에 파일을 제출하려면 대신 `multipart/form-data` 인코딩으로 전환해야 합니다.

예를 들어 `enctype` 속성을 지정하여 HTML 양식에서 이 형식으로 전환할 수 있습니다.

```xml
<!-- multipart/form-data를 사용하여 POST 인코딩하기 👇 -->
<form action="/handle-form" method="POST" enctype="multipart/form-data">
  <input type="text" name="Name" />
  <input type="date" name="DueDate" />
  <input type="checkbox" name="IsCompleted" />
  <input type="file" name="myFile" /> <!-- 👈 파일도 보내기 -->
  <input type="submit" />
</form>
```

이렇게 하면 다음과 같은 HTTP 요청이 생성됩니다.

```http
POST /handle-form HTTP/2
Host: andrewlock.net
accept: text/html,*/*
upgrade-insecure-requests: 1
content-type: multipart/form-data; boundary=----WebKitFormBoundaryYCxEzUfoh3oKUrnX
content-length: 3107

------WebKitFormBoundary8Vq2TDi66aYi2H5d
Content-Disposition: form-data; name="Name"

Andrew Lock
------WebKitFormBoundary8Vq2TDi66aYi2H5d
Content-Disposition: form-data; name="DueDate"

2024-08-01
------WebKitFormBoundary8Vq2TDi66aYi2H5d
Content-Disposition: form-data; name="IsCompleted"

on
------WebKitFormBoundary8Vq2TDi66aYi2H5d
Content-Disposition: form-data; name="myFile"; filename="some-file-to-send.json"
Content-Type: application/octet-stream

<binary data>

------WebKitFormBoundary8Vq2TDi66aYi2H5d--
```

이것은 간단한 `application/x-www-form-urlencoded` 요청보다 훨씬 더 장황하지만 훨씬 더 유연합니다. 여기서 주목해야 할 몇 가지 중요한 기능이 있습니다.

* 각 "파트"는 경계 마커로 구분됩니다: `------WebKitFormBoundary8Vq2TDi66aYi2H5d`. 사용할 경계는 `content-type` 헤더에 선언됩니다.
    
* 각 "파트"에는 `form-data` 유형이 있고 파트가 정의하는 필드 `name`을 지정하는 `Content-Disposition` 헤더가 포함됩니다.
    
* 각 "파트"에는 선택적 `Content-Type`이 포함됩니다. 지정되지 않은 경우 `text/plain`이 가정되지만 변경할 수 있습니다. 파일의 경우 파일이 바이너리 파일(`application/octet-stream`)로 제출된 것을 볼 수 있습니다.
    

HTML 양식 제출에서 오는 일반적인 HTTP 요청의 경우, 이 두 가지 형식이 거의 정확히 맞습니다. 하지만 요청이 앱 코드에서 오는 경우, `multipart/form-data` 요청에 다른 유형의 데이터를 전송하는 것을 막을 수 있는 방법은 없습니다.

## JSON 및 바이너리 데이터를 [`multipart/form-data`](https://andrewlock.net/reading-json-and-binary-data-from-multipart-form-data-sections-in-aspnetcore/#sending-json-and-binary-data-as-multipart-form-data)로 보내기

서버에 요청을 보내려고 한다고 가정해 보겠습니다. 바이너리 데이터의 전체 로드와 바이너리에 대한 일부 JSON 메타데이터도 포함해야 합니다. 어떻게 처리하시겠습니까?

JSON 데이터가 충분히 작다면 요청 헤더나 쿼리 문자열에서 인코딩한 다음 요청 본문에서 바이너리를 보낼 수 있습니다. 이 접근 방식은 효과가 있을 수 있지만 이 방식에는 여러 가지 잠재적인 문제가 있습니다. 이 방법을 사용하지 않는 간단한 이유는 헤더와 쿼리 문자열 값이 종종 로그에 남기 때문에 민감한 정보가 유출될 수 있기 때문입니다. 또한 허용되는 JJSON의 크기에 제한을 두어 문제가 될 수 있습니다.

이 경우 우리가 결정한 접근 방식은 `multipart/form-data` 요청을 보내는 것이었습니다. 여기에는 다음 처럼 한 파트에는 JSON 데이터가, 다른 파트에는 바이너리 데이터가 포함됩니다.

```http
--73dc24e0-b350-48f8-931e-eab338df00e1
Content-Type: application/json; charset=utf-8
Content-Disposition: form-data; name=myJson

{"Name":"Andrew","Type":"Engineer"}
--73dc24e0-b350-48f8-931e-eab338df00e1
Content-Type: application/octet-stream
Content-Disposition: form-data; name=binary

<binary data>
--73dc24e0-b350-48f8-931e-eab338df00e1--
```

두 파트 모두에 `content-type`이 있다는 점에 유의하세요. 첫 번째 파트는 `application/json`이고 두 번째 파트는 바이너리인 `application/octet-stream`입니다.

다음 `HttpClient` 코드를 사용하여 이와 같은 HTTP 요청을 보낼 수 있습니다. `multipart/form-data`를 하나씩 작성해야 하므로 일반적인 `HttpClient` 요청보다 복잡하지만, 중요한 단계를 설명하기 위해 코드에 주석을 달았습니다.

```csharp
var client = new HttpClient();

// 다음은 전송할 JSON 데이터입니다.
var myData = new MyData("Andrew", "Engineer");
// 전송할 임의의 데이터로 바이트 배열을 생성합니다.
var myBinaryData = new byte[100];
Random.Shared.NextBytes(myBinaryData);

// "최상위" 콘텐츠를 만듭니다. 이렇게 하면 content-type이
// multipart/form-data 데이터로 설정되고 컨테이너 역할을 합니다.
using var content = new MultipartFormDataContent();

// JsonContent 파트를 생성하고 "myJson"이라는 이름으로
// multipart/form-data에 추가합니다.
content.Add(JsonContent.Create(myData), "myJson");

// 바이너리 파트를 생성하고 multipart/form-data에 "binary" 이름으로 추가합니다.
// 명시적으로 Content-type, 그렇지 않으면 기본값(text/plain)을 사용합니다.
var binaryContent = new ByteArrayContent(myBinaryData);
binaryContent.Headers.ContentType = new("application/octet-stream");
content.Add(binaryContent, "binary");

// 상대방이 데이터를 제대로 수신했는지 확인하기 위해 전송하는 데이터를 기록하세요!
app.Logger.LogInformation("Sending Json {Data} and binary data {Binary}", myData, Convert.ToBase64String(myBinaryData));

// 요청 보내기
var response = await client.PostAsync("https://localhost:8080/", content);
response.EnsureSuccessStatusCode();

// JSON record 정의
record MyData(string Name, string Type);
```

이 작업을 실행하면 다음과 같은 내용이 기록됩니다.

```plaintext
Sending Json MyData { Name = Andrew, Type = Engineer } and binary data qe8gFK3/PdDIZQ3MrpQBB0o9ymSs8Azk6ALo0raP2qn0mB2BKqlB0DkXuJHG79OyvdwabLgMCdr2a8U1txABVo3pxN0ik8oT6P4zIlfgAwDH/ZcV118tqdqITvE8B2NmMdayIA==
```

HTTP 요청의 본문이 위와 비슷해집니다. 이제 데이터를 전송할 수 있으므로 이 데이터를 수신하는 다른 부분인 ASP.NET Core 엔드포인트를 작성해야 합니다.

## ASP.NET Core에서 파일이 아닌 복잡한 `multipart/form-data`를 읽는 것에 대한 어려움

[.NET 8에 최소 API 엔드포인트 매개 변수에 대한 양식 바인딩 지원이 추가 되었습니다.](https://andrewlock.net/exploring-the-dotnet-8-preview-form-binding-in-minimal-apis/) 대부분 간단한 경우, 이 바인딩은 이 글의 첫 번째 부분에서 보여드린 `multipart/form-data`와 같은 양식 파일과 양식 데이터에 쉽게 액세스할 수 있는 방법을 제공합니다.

```csharp
app.MapPost("/handle-form", (
    [FromForm] string name,  // 👈 name 파트에 바인딩
    [FromForm] DateOnly dueDate, // 👈 dueDate 파트에 바인딩
     IFormFile myFile) // 👈 스트림을 myFile 파트에 바인딩
    => Results.Ok(comment));
```

`[FromForm]` 매개 변수는 다음과 같이 양식 데이터 부분에 간단하게 바인딩됩니다.

```http
------WebKitFormBoundary8Vq2TDi66aYi2H5d
Content-Disposition: form-data; name="Name"

Andrew Lock
```

`IFormFile` 매개변수는 다음과 같이 파일 파트에 바인딩됩니다.

```http
------WebKitFormBoundary8Vq2TDi66aYi2H5d
Content-Disposition: form-data; name="myFile"; filename="some-file-to-send.json"
Content-Type: application/octet-stream

<binary data>
```

`IFormFile` 매개변수는 바이너리 파일 데이터를 읽기 위한 `Stream`을 제공하는데, 이 스트림이 바로 우리가 필요로 하는 기능입니다.

안타깝게도 우리 요청에는 해당 기능이 작동하지 않습니다.😢

JSON과 바이너리 요청에는 간단한 양식 값이 없으므로 `[FromForm]`을 사용하여 바인딩할 수 없습니다. 또한 `IFormFile`은 바이너리 부분에 바인딩되지 않으며, `IFormFile` 또는 `IFormFileCollection`을 바인딩하려고 하면 항상 `null`이 됩니다.🙁

> 그 이유에 대해서는 나중에 자세히 설명해 드리겠습니다!

좋아요... 항상 `HttpRequest.GetFormAsync()`의 폴백이 있지 않나요? 불행히도 아닙니다.

![Image showing the result of running IFormCollection form = await request.ReadFormAsync() on a request](https://andrewlock.net/content/images/2023/multipart_01.png align="left")

위 이미지에서 볼 수 있듯이 `GetFormAsync()`를 호출하면 `multipart/form-data` 데이터가 포함된 `IFormCollection`이 반환됩니다. 그러나 이 이미지는 응답의 각 부분이 `StringValues` 객체(즉, 문자열)로 변환되었음을 보여줍니다. 이 API를 통해 파일의 "원시" 바이너리 데이터를 가져올 수 있는 방법은 없습니다.🙁

이 기능을 테스트한 후 약간 난관에 봉착했습니다. `HttpRequest`의 "원시" 데이터를 확보할 수 있는 방법이 없는 것 같았습니다.

## `MultipartReader`로 `multipart/form-data` 수동 읽기

이 시점에서 저는 ASP.NET Core 소스 코드를 자세히 살펴보기로 결정했습니다. ASP.NET Core는 분명히 요청을 파싱하고 "파일"에 대한 `Stream`을 가져올 수 있으므로 재사용할 수 있는 무언가가 있을 것입니다.

물론 약간의 검색 끝에 요청 본문을 파싱하는 [`FormFeature` 구현](https://github.com/dotnet/aspnetcore/blob/6c1b3dfb66a1b66cc32ee26bbc23f6472f1dc985/src/Http/Http/src/Features/FormFeature.cs#L193)을 발견했습니다. 그리고 이 구현은 `public` 클래스인 `MultipartReader`를 사용하여 이 작업을 수행합니다!🥳

예상할 수 있듯이 `MultipartReader`는 요청 본문 읽기를 용이하게 합니다. 이는 엔드포인트에서 일반적으로 상호 작용하는 `IFormCollection` 또는 `IFormFile` 유형보다 낮은 수준의 API로, 일반적으로 이러한 유형을 채우는 데 사용됩니다.

응답 본문에 대한 지원을 추가하는 작업은 대부분 `FormFeature`에서 사용되는 광범위한 접근 방식을 복사하되 사용 사례에 맞게 조정하는 것이었습니다. 다음 엔드포인트는 `MultipartReader`를 사용하여 JSON 및 바이너리 응답을 파싱하고 요청 스트림에서 직접 역직렬화하는 방법을 보여줍니다. 여기에는 꽤 많은 코드가 있지만 최대한 주석을 달았습니다!

```csharp
// POST를 처리하는 최소한의 API 엔드포인트를 만듭니다.
// JSON을 역직렬화할 때 기본 설정된 JsonOptions을 사용합니다.
app.MapPost("/", async (HttpContext ctx, IOptions<JsonOptions> jsonOptions) =>
{
    // 올바른 헤더 유형이 있는지 확인합니다.
    if (!MediaTypeHeaderValue.TryParse(ctx.Request.ContentType, out MediaTypeHeaderValue? contentType)
        || !contentType.MediaType.Equals("multipart/form-data", StringComparison.OrdinalIgnoreCase))
    {
        return Results.BadRequest("Incorrect mime-type");
    }

    // 응답에서 파싱한 데이터를 보관하는 변수
    MyData? jsonData = null;
    byte[]? binaryData = null;

    // content-type에서 multipart/form-boundary 헤더를 가져옵니다.
    // Content-Type: multipart/form-data; boundary="--73dc24e0-b350-48f8-931e-eab338df00e1"
    // 사양에 따르면 70자가 적당한 제한이라고 나와 있습니다.
    string boundary = GetBoundary(contentType, lengthLimit: 70);
    var multipartReader = new MultipartReader(boundary, ctx.Request.Body);
    
    // multipart reader를 사용하여 각 섹션을 읽습니다.
    while (await multipartReader.ReadNextSectionAsync(ct) is { } section)
    {
        // 섹션에 대한 content-type이 있는지 확인합니다.
        if(!MediaTypeHeaderValue.TryParse(section.ContentType, out MediaTypeHeaderValue? sectionType))
        {
            return Results.BadRequest("Invalid content type in section " + section.ContentType);
        }

        if (sectionType.MediaType.Equals("application/json", StringComparison.OrdinalIgnoreCase))
        {
            // 섹션이 JSON인 경우 앱에 구성된 기본 JSON 직렬화 옵션을 사용하여
            // 섹션 스트림에서 직접 역직렬화합니다.
            jsonData = await JsonSerializer.DeserializeAsync<MyData>(
                section.Body,
                jsonOptions.Value.JsonSerializerOptions,
                cancellationToken: ctx.RequestAborted);
        }
        else if (sectionType.MediaType.Equals("application/octet-stream", StringComparison.OrdinalIgnoreCase))
        {
            // 섹션이 바이너리 데이터인 경우, 데이터가 필요한 방식에 따라
            // 배열로 역직렬화하면 더 효율적인 작업을 수행할 수 있습니다.
            using var ms = new MemoryStream();
            await section.Body.CopyToAsync(ms, ctx.RequestAborted);
            binaryData = ms.ToArray();
        }
        else
        {
            return Results.BadRequest("Invalid content type in section " + section.ContentType);
        }
    }

    // 디버깅 목적으로 인쇄
    app.Logger.LogInformation("Receive Json {JsonData} and binary data {BinaryData}",
        jsonData, Convert.ToBase64String(binaryData));

    return Results.Ok();
    
    // 따옴표 등을 처리하여 content-type에서 경계 마커를 검색합니다.
    // 다음에서 가져옴: https://github.com/dotnet/aspnetcore/blob/4eef6a1578bb0d8a4469779798fe9390543d15c0/src/Http/Http/src/Features/FormFeature.cs#L318-L320
    static string GetBoundary(MediaTypeHeaderValue contentType, int lengthLimit)
    {
        var boundary = HeaderUtilities.RemoveQuotes(contentType.Boundary);
        if (StringSegment.IsNullOrEmpty(boundary))
        {
            throw new InvalidDataException("Missing content-type boundary.");
        }
        if (boundary.Length > lengthLimit)
        {
            throw new InvalidDataException($"Multipart boundary length limit {lengthLimit} exceeded.");
        }
        return boundary.ToString();
    }
});
```

휴. 많은 코드처럼 보이지만 작동하며, 우리가 수신하는 페이로드를 역직렬화할 수 있는 유일한 방법이기도 합니다.

```plaintext
info: testApp[0]
      Sending Json MyData { Name = Andrew, Type = Engineer } and binary data JtlpOBcTftaEIUSfXO1X3K5ubjE09ewqsappxBj6ok0n0K8dUIexc7RXLhymJsb9eErSoCnZ7+ZLsooe9cQ5gWAG3wKJFpfIRP7qnUceJpes45hJksBTA91J5bqLJIZfVnWagQ==
info: testApp[0]
      Receive Json MyData { Name = Andrew, Type = Engineer } and binary data JtlpOBcTftaEIUSfXO1X3K5ubjE09ewqsappxBj6ok0n0K8dUIexc7RXLhymJsb9eErSoCnZ7+ZLsooe9cQ5gWAG3wKJFpfIRP7qnUceJpes45hJksBTA91J5bqLJIZfVnWagQ==
```

하지만 이 페이로드를 약간만 조정하면 ASP.NET Core의 기본 제공 기능을 사용하여 자동으로 응답을 읽을 수 있지 않을까 하는 생각이 들었습니다. 🤔

## `multipart/form-data` 요청을 조정하여 ASP.NET Core를 만족스럽게 유지

본문 읽기를 위한 ASP.NET Core의 기본 제공 지원을 더 많이 활용하려면 각 multipart/form-data 파트/섹션이 파트의 이름과 파일명을 모두 선언하는 `Content-Disposition`과 함께 전송되도록 하는 것이 핵심입니다.

```http
--883a97cb-5025-47c0-8d5f-e238687cfc5e
Content-Type: application/json; charset=utf-8
Content-Disposition: form-data; name=myJson; filename=some_json.json

{"name":"Andrew","type":"Engineer"}
--883a97cb-5025-47c0-8d5f-e238687cfc5e
Content-Type: application/octet-stream
Content-Disposition: form-data; name=binary; filename=my_binary.log

<binary data>
--883a97cb-5025-47c0-8d5f-e238687cfc5e--
```

앞서 보여드린 양식 데이터와 비교해보면 두 경우 모두 `Content-Disposition`에 `filename`을 추가한 것을 알 수 있습니다. 해당 파일명을 추가하기 위해 `HttpClient` 코드를 업데이트하는 것은 간단합니다. `MultipartFormDataContent`를 만들 때 파일명을 지정하기만 하면 됩니다.

```csharp
content.Add(
    content: JsonContent.Create(myData),
    name: "myJson",
    filename: "some_json.json"); //👈 이 매개변수 추가
```

이러한 접근 방식을 취하면 "전송" 코드를 적절히 조정할 수 있습니다. 아래 코드는 원본과 동일하지만 `Content-Disposition` `filename`이 설정되어 있습니다.

```csharp
var client = new HttpClient();

var myData = new MyData("Andrew", "Engineer");
var myBinaryData = new byte[100];
Random.Shared.NextBytes(myBinaryData);

using var content = new MultipartFormDataContent();
content.Add(JsonContent.Create(myData), "myJson", "some_json.json"); // 👈 added filename

var binaryContent = new ByteArrayContent(myBinaryData);
binaryContent.Headers.ContentType = new("application/octet-stream");
content.Add(binaryContent, "binary", "my_binary.log"); // 👈 Added filename

app.Logger.LogInformation("Sending Json {Data} and binary data {Binary}", myData, Convert.ToBase64String(myBinaryData));

var response = await client.PostAsync("https://localhost:8080/", content);
response.EnsureSuccessStatusCode();
```

그렇다면 ASP.NET Core에서 어떻게 하면 작업이 더 쉬워질까요?

주요 차이점은 `FormFeature`가 섹션의 `Content-Disposition` 헤더에 `filename`이 있는지 확인한다는 것입니다. 헤더가 발견되면 ASP.NET Core는 [자동으로 파일 본문을 메모리(또는 대용량 파일의 경우 디스크)에 버퍼링](https://github.com/dotnet/aspnetcore/blob/6c1b3dfb66a1b66cc32ee26bbc23f6472f1dc985/src/Http/Http/src/Features/FormFeature.cs#L221-L251)하고 `IFormFile` 인스턴스를 구성하여 `HttpRequest.Form.Files` 속성에서 사용할 수 있을 뿐만 아니라 엔드포인트에 직접 삽입할 수도 있습니다.

즉, 최소한의 API를 획기적으로 간소화할 수 있습니다. 더 이상 `MultipartReader`를 사용하여 요청을 직접 파싱할 필요가 없습니다. 대신 `IFormFile`을 사용하여 거기에서 직접 데이터를 가져올 수 있습니다.

```csharp
// 최소 API 엔드포인트에서 파일을 직접 참조합니다.
// 매개변수의 이름은 콘텐츠의 파일 이름과 일치해야 합니다.
app.MapPost("/", async (IFormFile myJson, IFormFile binary, IOptions<JsonOptions> jsonOptions) =>
{
    // 'myJson' 파일에서 JSON 데이터 역직렬화하기
    var jsonData = await JsonSerializer.DeserializeAsync<MyData>(
        myJson.OpenReadStream(), jsonOptions.Value.JsonSerializerOptions);
    
    // '바이너리' 파일에서 바이너리 데이터 복사하기
    var binaryData = new byte[binary.Length]; // 👈 We know the length because it's already buffered
    using var ms = new MemoryStream(binaryData);
    await binary.CopyToAsync(ms);

    // 시연용
    app.Logger.LogInformation("Receive Json {JsonData} and binary data {BinaryData}",
        jsonData, Convert.ToBase64String(binaryData));

    return Results.Ok();
}).DisableAntiforgery(); // .NET 8은 CSRF 보호 기능을 자동으로 추가하므로 이 예제에서는 이 기능을 비활성화합니다.
```

본질적으로 동일한 작업을 수행하는 데 훨씬 적은 코드가 필요합니다!🎉 각 파일에 대해 콘텐츠 유형 등이 올바른지 확인하지 않아서 약간의 속임수를 썼지만 원한다면 언제든지 다시 추가할 수 있습니다.

두 접근 방식에는 약간의 미묘한 차이가 있습니다. 이 접근 방식을 사용하면 ASP.NET Core가 자동으로 본문을 읽고 메모리에 버퍼링한 다음 작업할 수 있도록 `IFormFile` 인스턴스를 생성합니다. `MultipartReader`를 사용하는 원래의 저수준 접근 방식에서는 요청 스트림으로 직접 작업하므로 나중에 데이터로 정확히 무엇을 하느냐에 따라 버퍼링과 개체 생성을 일부 생략할 수 있을 것입니다.

또한 다음과 같이 최소한의 API 엔드포인트에서 `MyData` 객체를 직접 요청하면 JSON 역직렬화를 직접 수행할 필요가 없는지 궁금할 수도 있습니다.

```csharp
// ⚠ 작동하지 않습니다.
app.MapPost("/", async (MyData myJson, IFormFile binary) => { /* */ });
```

안타깝게도 불가능합니다. 요청 본문의 "JSON-in-form"은 표준이 아니며, ASP.NET Core는 이를 지원하지 않습니다. 위의 접근 방식을 시도하면 ASP.NET Core가 요청 본문 전체에 `MyData` JSON 개체를 바인딩하고 본문을 양식 데이터로 읽으려고 시도하므로 오류가 발생합니다.

> 최소한의 API가 바인딩할 대상을 결정하는 방법에 대해 자세히 알아보려면 [최소한의 API에서 바인딩하기 시리즈](https://andrewlock.net/series/behind-the-scenes-of-minimal-apis/)를 읽어보세요.

이 접근 방식을 사용하는 경우 ASP.NET Core는 HTML `<form>` 요청이 데이터를 전송하는 형식에 따라 `MyData` 개체의 각 개별 속성을 양식 데이터의 별도 섹션에 바인딩하려고 시도합니다. 이번에는 오류가 발생하지 않으며, `myJson` 매개변수에 데이터가 없을 뿐입니다!

여기까지입니다. multipart/form-data 읽기에 대한 두 가지 다른 접근 방식입니다. 첫 번째 접근 방식을 사용하지 않고 표준 파일 접근 방식을 고수할 수 있기를 바랍니다.

## 요약

이 글에서는 `multipart/form-data` 형식과 `application/x-www-form-urlencoded` 양식 데이터와 어떻게 다른지, 그리고 요청이 어떻게 보이는지에 대해 설명했습니다. HTML `<form>` 요소가 `multipart/form-data` 형식을 사용하여 파일을 전송하는 방법을 보여준 다음, `HttpClient`를 사용하여 `multipart/form-data` 요청에서 데이터를 전송하는 방법을 보여주었습니다. 예제에서는 동일한 요청에 JSON과 바이너리 데이터를 모두 보냈습니다.

문제는 파일이나 "일반" 양식 데이터로 전송되지 않는 한 ASP.NET Core가 `multipart/form-data` 요청을 자동으로 읽을 수 없다는 것입니다. JSON 또는 바이너리 데이터를 자동으로 읽을 수 있는 방법이 없습니다. ASP.NET Core에서 백그라운드에서 사용하는 `MultipartReader` 유형을 사용하여 이 데이터를 읽는 방법을 보여드렸습니다. 이렇게 하면 요청 본문에서 각 섹션의 읽기를 완벽하게 제어할 수 있습니다.

마지막으로 요청에 전송된 데이터를 조정하여 `Content-Disposition`에 파일 이름을 포함시키는 방법을 보여드렸습니다. 이렇게 변경한 후 ASP.NET Core는 표준 `IFormFile` 메커니즘을 사용하여 요청 본문을 읽을 수 있었습니다. 이렇게 하면 코드가 크게 간소화되었지만 여전히 JSON 데이터를 수동으로 역직렬화해야 했습니다.

---

%[https://andrewlock.net/reading-json-and-binary-data-from-multipart-form-data-sections-in-aspnetcore/]