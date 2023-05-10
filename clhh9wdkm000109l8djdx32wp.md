---
title: "콘솔에서 Azure OpenAI 및 C#으로 챗봇 만들기 | Niels Swimberghe"
datePublished: Wed May 10 2023 05:41:37 GMT+0000 (Coordinated Universal Time)
cuid: clhh9wdkm000109l8djdx32wp
slug: create-a-chatbot-in-the-console-with-azure-openai-and-csharp
canonical: https://swimburger.net/blog/dotnet/create-a-chatbot-in-the-console-with-azure-openai-and-csharp
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1683695526990/fa10ef35-7e23-4d16-9737-874dcae0751c.webp
tags: net, dotnet, azure-openai

---

> Niels Swimberghe님의 [**Create a chatbot in the console with Azure OpenAI and C#**](https://swimburger.net/blog/dotnet/create-a-chatbot-in-the-console-with-azure-openai-and-csharp)글을 번역하였습니다.

---

OpenAI에서 [ChatGPT](https://openai.com/blog/chatgpt)가 출시하자마자 입소문을 타기 시작했습니다. 불과 몇 주 만에 ChatGPT는 컴퓨터 기술자들만 알고 있던 것에서 어린이와 청소년들이 학교 숙제를 '컨닝'하는 데 도움을 주기 위해 빠르게 채택되었습니다. 얼마 지나지 않아 기술 전문가가 아닌 제 친구들조차도 제가 언급하지 않았는데도 ChatGPT와 Bing 채팅을 실험하기 시작했습니다.

그 이후로 OpenAI는 ChatGPT용 API를 공개했지만, 그 외에도 Azure는 OpenAI와 협력하여 Azure 클라우드 플랫폼에서 기대할 수 있는 보안, 통합 및 관리 기능을 추가한 자체 버전의 [OpenAI 모델을 Azure를 통해 제공](https://azure.microsoft.com/en-us/products/cognitive-services/openai-service)하고 있습니다.

Azure OpenAI Service의 흥미로운 점은 OpenAI와 동일한 모델을 사용하지만 Azure 구독 내에 배포되므로 OpenAI 및 다른 Azure OpenAI 서비스와 완전히 격리된다는 것입니다. 실수로 다른 사람과 데이터를 공유할 염려가 없습니다.

> Azure는 Azure OpenAI 서비스를 "일반 사용 가능"으로 출시했지만 실제로 서비스에 액세스하려면 [이 양식을 작성](https://aka.ms/oai/access?azure-portal=true)하고 Microsoft의 승인을 받아야 합니다. (제발 Microsoft, 수동으로 승인을 받아야 하는 서비스는 GA가 아니지만 본론으로 들어가겠습니다.)

이 자습서에서는 콘솔 내에서 챗봇을 만들기 위해 OpenAI .NET 클라이언트 라이브러리를 사용하여 Azure OpenAI 서비스를 만들고 API를 사용하는 방법을 배웁니다.

## 전제 조건

다음 사항을 준수하려면 다음 사항이 필요합니다:

* [.NET 7 SDK](https://dotnet.microsoft.com/en-us/download/dotnet/7.0)(이전 버전 및 최신 버전도 작동할 수 있음)
    
* 코드 편집기 또는 IDE(C# 플러그인이 포함된 [JetBrains Rider](https://www.jetbrains.com/rider/), [Visual Studio](https://visualstudio.microsoft.com/downloads/) 또는 [C# 플러그인](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csharp)과 함께 [VS Code](https://code.visualstudio.com/Download) 권장)
    
* Azure 구독 및 Azure OpenAI 서비스 액세스([여기에서 액세스 요청](https://aka.ms/oai/access?azure-portal=true))
    

[이 자습서의 소스 코드는 이 GitHub 리포지토리](https://github.com/Swimburger/ConsoleChatGptOnAzure)에서 찾을 수 있습니다. 문제가 발생하거나 질문이 있는 경우 언제든지 GitHub 이슈를 제출하세요.

## Azure OpenAI 서비스 만들기

Azure OpenAI 서비스를 만들려면 먼저 브라우저를 열고 [Azure Portal로 이동](https://portal.azure.com/)합니다. 왼쪽의 탐색을 열고 **리소스 만들기(Create a resource)**를 클릭합니다.

![User clicks on the &quot;Create a resource&quot; link in the Azure Portal navigation](https://swimburger.net/media/jdde5uan/1-azure-portal-create-resource-nav.png align="left")

그러면 Azure 마켓플레이스가 열립니다. 검색 상자를 사용하여 "**OpenAI**"를 검색하고 Azure OpenAI 제품을 클릭합니다.

![User searching for OpenAI in the Azure Marketplace and clicking on the Azure OpenAI product card.](https://swimburger.net/media/p03jwxxq/2-azure-portal-create-resource-search-openai.png align="left")

Azure OpenAI의 제품 페이지에서 **만들기(Create)** 단추를 클릭합니다.

![Azure OpenAI product page where user click on the Create button.](https://swimburger.net/media/4hflcwbx/3-azure-portal-openai-product-page.png align="left")

사용하려는 **리소스 그룹**을 선택하거나 새 리소스 그룹을 만들고, OpenAI 인스턴스에 **전역 고유 이름**을 지정하고, 가격 책정 계층을 선택합니다(이 글을 작성하는 시점에는 하나만 있음). 그런 다음 **다음(Next)**을 클릭합니다.

![Create Azure OpenAI dialog, asking for the subscription, resource group, region, name, and pricing tier. User filled out the form and is clicking on the Next button.](https://swimburger.net/media/450feb5l/4-create-openai-service-basics.png align="left")

**네트워크(Next)** 및 **태그(Tag)** 페이지의 기본값을 그대로 둡니다. **검토 + 제출(Review + submit)** 페이지가 표시될 때까지 다음을 클릭합니다. 여기에서 만들려는 항목에 대한 개요를 볼 수 있습니다. **만들기(Create)** 버튼을 클릭합니다.

![Create Azure OpenAI Review + submit dialog, showing an overview of the resource to be created. User clicks on Create button.](https://swimburger.net/media/jwafvnwg/5-create-openai-service-review-and-submit.png align="left")

이제 Azure에서 Azure OpenAI 인스턴스를 제공하는데 시간이 좀 걸립니다. 제 경우에는 10분 정도 걸렸으니 커피나 맛있는 차를 마시러 가세요. 😉

![Azure deployment screen showing the Azure OpenAI service has been deployed. User clicks on &quot;Go to resource&quot; button.](https://swimburger.net/media/p2phsbmx/6-openai-service-created.png align="left")

Azure에서 "배포가 완료되었습니다"라는 메시지가 표시되면 **리소스로 이동(Go to resource)** 버튼을 클릭합니다. 그런 다음 OpenAI 인스턴스에서 왼쪽 탐색의 **모델 배포(Model deployments)** 탭을 클릭한 다음 상단의 **만들기(Create)** 버튼을 클릭합니다.

![User navigates to the Model deployments tab using the left side navigation of the Azure OpenAI service. User then clicks Create button which opens the Create Model deployment modal. User fills out the form as described below, and clicks the Save button.](https://swimburger.net/media/1u4j3cxh/7-openai-service-create-model-deployment.png align="left")

모델에 배포 이름에 아무 **이름**을 지정하고, 모델에 **gpt-35-turbo(버전 0301)**를 선택하고, 버젼에 0301을 선택한 다음 **저장(Save)**을 클릭합니다.

> Gpt-35-turbo는 OpenAI가 ChatGPT를 위해 특별히 학습한 모델이지만, ChatGPT는 Azure OpenAI에서 아직 사용할 수 없는 최신 모델인 GPT4도 제공합니다.
> 
> Azure 포털 자체에서 모델을 사용자 지정할 수는 없지만 "Azure OpenAI Studio로 이동" 링크를 클릭하면 인지 서비스 포털로 이동하여 다양한 모델을 실험할 수 있는 플레이그라운드를 사용할 수 있으며, 추가 학습 데이터를 제공하여 모델을 사용자 지정할 수 있습니다. 이것은 이 새로운 서비스의 시작에 불과하며 앞으로 더 많은 기능이 추가될 예정이라는 점을 기억하세요.)

나중에 .NET 애플리케이션에서 **이 이름이 필요하므로 모델 배포 이름을 기억**하세요.

이제 왼쪽 탐색에서 **키 및 엔드포인트(Keys and Endpoint)** 탭을 클릭하고 **KEY 1**과 엔드포인트를 안전한 곳에 복사합니다. .NET 애플리케이션에도 이 두 개가 필요합니다.

![User navigates to Keys and Endpoint tab using the left navigation for Azure OpenAI service. Then user copies KEY 1 and Endpoint field.](https://swimburger.net/media/cl0jaxpo/8-openai-service-keys-and-endpoint.png align="left")

## .NET 기반 콘솔 챗봇 만들기

.NET CLI(또는 IDE)를 사용하여 새 콘솔 애플리케이션을 만듭니다:

```bash
dotnet new console -o ChatConsole
cd ChatConsole
```

[Secrets Manager](https://learn.microsoft.com/en-us/aspnet/core/security/app-secrets?view=aspnetcore-7.0&tabs=windows#secret-manager)에서 구성을 로드하는 데 사용할 [사용자 비밀번호 구성 빌더 NuGet 패키지](https://www.nuget.org/packages/Microsoft.Extensions.Configuration.UserSecrets)를 추가합니다:

```bash
dotnet add package Microsoft.Extensions.Configuration.UserSecrets
```

모든 구성을 사용자 비밀로 저장하여 작업을 간단하고 안전하게 유지합니다. 프로젝트에 대한 사용자 비밀을 초기화하고 다음 사용자 비밀을 구성합니다:

```bash
dotnet user-secrets init
dotnet user-secrets set Azure:OpenAI:Endpoint [YOUR_AZURE_OPENAI_ENDPOINT]
dotnet user-secrets set Azure:OpenAI:ApiKey [YOUR_AZURE_OPENAI_APIKEY]
dotnet user-secrets set Azure:OpenAI:ModelName [YOUR_MODEL_DEPLOYMENT]
```

`[YOUR_AZURE_OPENAI_ENDPOINT]`를 **엔드포인트** URL로, `[YOUR_AZURE_OPENAI_APIKEY]`를 **KEY 1**로, `[YOUR_MODEL_DEPloyment]`를 앞서 메모한 모델 배포의 이름으로 바꿉니다.

다음 코드를 사용하여 Program.cs 파일을 업데이트합니다:

```csharp
using Microsoft.Extensions.Configuration;

var configuration = new ConfigurationBuilder()
    .AddUserSecrets<Program>()
    .Build();
```

이 코드는 방금 구성한 사용자 비밀번호로 구성을 빌드합니다.

## Azure OpenAI SDK 설치 및 구성

[Azure OpenAI NuGet 패키지](https://www.nuget.org/packages/Azure.AI.OpenAI)를 추가합니다:

```bash
dotnet add package Azure.AI.OpenAI --prerelease
```

SDK는 아직 베타 버전이므로 현재 사전 릴리스로만 사용할 수 있습니다. 정식 릴리스가 출시되면 언제든지 사전 릴리스 인수를 삭제하세요.

Program.cs에 다음 네임스페이스를 추가합니다:

```csharp
using Azure;
using Azure.AI.OpenAI;
```

그리고 Program.cs에 다음 코드를 추가하여 새 `OpenAIClient`를 만듭니다:

```csharp
var openAiClient = new OpenAIClient(
    new Uri(configuration["Azure:OpenAI:Endpoint"]),
    new AzureKeyCredential(configuration["Azure:OpenAI:ApiKey"])
);
```

## ChatComplements API 호출

이제 OpenAI 클라이언트를 설정했으므로 API 호출을 시작할 수 있습니다. Program.cs에 다음 코드를 추가하여 챗봇을 완성합니다:

```csharp
var chatCompletionsOptions = new ChatCompletionsOptions
{
    Messages =
    {
        new ChatMessage(ChatRole.System, "You are Rick from the TV show Rick & Morty. Pretend to be Rick."),
        new ChatMessage(ChatRole.User, "Introduce yourself."),
    }
};

while (true)
{
    Console.WriteLine();
    Console.Write("Rick: ");
    
    var chatCompletionsResponse = await openAiClient.GetChatCompletionsAsync(
        configuration["Azure:OpenAI:ModelName"],
        chatCompletionsOptions
    );

    var chatMessage = chatCompletionsResponse.Value.Choices[0].Message;
    Console.Write(chatMessage.Content);
    
    chatCompletionsOptions.Messages.Add(chatMessage);
    
    Console.WriteLine();
    
    Console.Write("Enter a message: ");
    var userMessage = Console.ReadLine();
    chatCompletionsOptions.Messages.Add(new ChatMessage(ChatRole.User, userMessage));
}
```

`ChatCompletionsOptions` 객체는 사용자와 챗봇 간의 대화를 추적하지만, 대화가 실제로 시작되기도 전에 두 개의 채팅 메시지가 미리 채워져 있다는 점에 유의해야 합니다. 한 채팅 메시지는 `시스템`에서 보낸 것으로, 어떤 종류의 챗봇이 되어야 하는지에 대한 지침을 채팅 모델에 제공합니다. 이 경우, 저는 채팅 모델에게 [릭 앤 모티의 릭](https://www.adultswim.com/videos/rick-and-morty)처럼 행동하라고 지시했습니다. 그런 다음 챗봇에게 "자신을 소개하세요."라는 `사용자`의 메시지를 추가하여 자신을 소개하도록 했습니다.

무한 루프 내부에서 `chatCompletionsOptions`는 생성한 배포 모델 이름과 함께 `openAiClient.GetChatCompletionsAsync` 메서드에 전달됩니다.

그런 다음 채팅 모델의 응답이 콘솔에 기록되고 `chatCompletionsOptions`에 저장된 채팅 기록에 추가됩니다.

이제 사용자에게 채팅 기록에 추가될 내용을 입력하라는 메시지가 표시되고 루프의 다음 반복이 시작되어 채팅 기록이 Azure OpenAI의 채팅 모델에 다시 전송됩니다.

멋진 작업입니다! 프로젝트를 실행하여 실제로 작동하는지 확인하세요:

```bash
dotnet run
```

![Console output from the chat bot console application](https://swimburger.net/media/mscnca3s/console-output.png align="left")

## 채팅 응답 스트리밍

현재 채팅 모델의 응답은 콘솔에 한 번에 모두 기록됩니다. 릭이 실제로 답변을 입력하는 것처럼 보이게 하려면 스트리밍 API를 사용하여 들어오는 단어를 스트리밍하여 하나씩 콘솔에 기록하면 됩니다.

다음 코드를 사용하여 Program.cs 파일을 업데이트합니다:

```csharp
using System.Text;
using Azure;
using Azure.AI.OpenAI;
using Microsoft.Extensions.Configuration;

var configuration = new ConfigurationBuilder()
    .AddUserSecrets<Program>()
    .Build();
    
var openAiClient = new OpenAIClient(
    new Uri(configuration["Azure:OpenAI:Endpoint"]),
    new AzureKeyCredential(configuration["Azure:OpenAI:ApiKey"])
);

var chatCompletionsOptions = new ChatCompletionsOptions
{
    Messages =
    {
        new ChatMessage(ChatRole.System, "You are Rick from the TV show Rick & Morty. Pretend to be Rick."),
        new ChatMessage(ChatRole.User, "Introduce yourself."),
    }
};

while (true)
{
    Console.WriteLine();
    Console.Write("Rick: ");
    
    var chatCompletionsResponse = await openAiClient.GetChatCompletionsStreamingAsync(
        configuration["Azure:OpenAI:ModelName"],
        chatCompletionsOptions
    );

    var chatResponseBuilder = new StringBuilder();
    await foreach (var chatChoice in chatCompletionsResponse.Value.GetChoicesStreaming())
    {
        await foreach (var chatMessage in chatChoice.GetMessageStreaming())
        {
            chatResponseBuilder.AppendLine(chatMessage.Content);
            Console.Write(chatMessage.Content);
            await Task.Delay(TimeSpan.FromMilliseconds(200));
        }
    }
    
    chatCompletionsOptions.Messages.Add(new ChatMessage(ChatRole.Assistant, chatResponseBuilder.ToString()));
    
    Console.WriteLine();
    
    Console.Write("Enter a message: ");
    var userMessage = Console.ReadLine();
    chatCompletionsOptions.Messages.Add(new ChatMessage(ChatRole.User, userMessage));
}
```

이제 `openAiClient.GetChatComplementsAsync`를 사용하는 대신 `openAiClient.GetChatComplementsStreamingAsync`를 사용하고 여러 비동기 열거형을 반복하여 응답의 단어를 스트리밍하지만 각 단어 사이에 약간의 지연 시간을 추가하여 Rick이 실제로 입력하는 것처럼 보이게 한 다음 콘솔에 단어를 작성합니다.

프로젝트를 다시 실행하여 결과를 확인합니다:

%[https://youtu.be/sdAO_WAdPWk] 

## 결론

이 자습서에서는 Azure OpenAI 인스턴스를 만들고 모델을 배포하는 방법, Azure OpenAI SDK를 .NET 애플리케이션에 통합하는 방법, 채팅 완료 API를 사용하여 챗봇을 만드는 방법을 배웠습니다.

---

%[https://swimburger.net/blog/dotnet/create-a-chatbot-in-the-console-with-azure-openai-and-csharp]