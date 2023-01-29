# Discord 봇 만들기

[Discord](https://discord.com/)는 게이머가 애용하는 실시간 채팅 커뮤니티 서비스로 이제는 게이밍 부터 교육과 비즈니스 영역으로 그 서비스를 확장하고 있습니다.

Discord 봇을 만드는 것은 [Discord.NET](https://discordnet.dev/)을 이용하면 어렵지 않습니다.

## 봇 생성

[DEVELOPER PORTAL](https://discord.com/developers/applications)에서 새로운 애플리케이션을 만듭니다.

![image](https://discourse-dotnetdev-upload.ewr1.vultrobjects.com/optimized/2X/3/3928914d4d5949491dfd745ec9e897f4371ed8cf_2_690x400.png align="left")

![image](https://discourse-dotnetdev-upload.ewr1.vultrobjects.com/original/2X/9/9bf8b6cdc200aa72a83762f4a034fc695a464b96.png align="left")

다음 `Bot`에서 봇을 추가합니다.

![image](https://discourse-dotnetdev-upload.ewr1.vultrobjects.com/optimized/2X/7/7d75b4d40a5b4531b2cae55c96ac235a8310b2d6_2_690x449.png align="left")

이 때 앱 이름이 많이 사용되는 이름일 경우 봇을 생성할 수 없으므로 앱 이름을 적절하게 변경해야 합니다.

`Reset Token`으로 봇 API에서 사용할 토큰을 생성합니다.

![image](https://discourse-dotnetdev-upload.ewr1.vultrobjects.com/optimized/2X/7/7d75b4d40a5b4531b2cae55c96ac235a8310b2d6_2_690x449.png align="left")

적절한 인텐트가 활성화 되지 않으면 봇 시작 시 정상 동작하지 않습니다.

![image](https://discourse-dotnetdev-upload.ewr1.vultrobjects.com/optimized/2X/2/25c1fcc13cf1c063a9277878dbe0a9033ce9c405_2_690x388.png align="left")

다음 서버에 봇을 추가할 URL을 생성합니다.

![image](https://discourse-dotnetdev-upload.ewr1.vultrobjects.com/optimized/2X/9/915f618e04c8c395b443031a0af74336323d0f4e_2_690x388.png align="left")

(BOT PERMISSIONS는 편의상 Administrator를 체크하였습니다. 실제로는 필요에 부합하는 최소한의 퍼미션만 선택해야 합니다.)

Discord가 로그인 된 상태에서 클립보드 복사된 주소를 실행하면 자신의 서버에 봇을 추가할 수 있습니다.

![image](https://discourse-dotnetdev-upload.ewr1.vultrobjects.com/optimized/2X/4/46277d149c24935415201ef2e1bbbe5b60694ede_2_605x500.png align="left")

![image](https://discourse-dotnetdev-upload.ewr1.vultrobjects.com/optimized/2X/1/18aaf802808192b25fcdf48fbae3246da2e67553_2_605x500.png align="left")

## Discord.NET을 이용해서 애코 기능 구현

봇의 시작은 에코 테스트부터 하는 것이 좋습니다. 콘솔 프로젝트로 먼저 [Discord.NET](http://Discord.NET) 패키지를 프로젝트에 설치합니다.

![image](https://discourse-dotnetdev-upload.ewr1.vultrobjects.com/optimized/2X/2/2b5d049bb8bceb5d6626570c011856c977f272a7_2_690x424.png align="left")

다음은 디스코드 봇의 기본 에코 코드 입니다.

```csharp
using Discord;
using Discord.Commands;
using Discord.WebSocket;

using System.Reflection;

var client = new DiscordSocketClient(new()
{
    LogLevel = LogSeverity.Verbose,
    // 실제 사용할 때는 꼭 필요한 intent만 지정하는 것이 좋습니다.
    GatewayIntents = GatewayIntents.All ^ (GatewayIntents.GuildPresences | GatewayIntents.GuildScheduledEvents | GatewayIntents.GuildInvites)
});
var commandService = new CommandService(new()
{
    LogLevel = LogSeverity.Verbose,
});

client.Log += OnClientLogReceived;
commandService.Log += OnClientLogReceived;

await client.LoginAsync(TokenType.Bot, "봇 토큰");
await client.StartAsync();
await commandService.AddModulesAsync(assembly: Assembly.GetExecutingAssembly(), services: null);

client.MessageReceived += OnClientMessage;
client.Ready += OnReady;

Console.ReadLine();

async Task OnClientMessage(SocketMessage arg)
{
    // 클라이언트 메시지 처리

    // 사용자 메시지가 아니면
    if (arg is not SocketUserMessage message)
        return;

    if (client is null || commandService is null)
        return;

    var pos = 0;
    
    //if (message.HasCharPrefix('!', ref pos) is false || message.HasMentionPrefix(_client.CurrentUser, ref pos) is true || message.Author.IsBot is true)
    
    
    if (message.HasCharPrefix('!', ref pos) is false || message.Author.IsBot is true)
        return;

    var context = new SocketCommandContext(client, message);
    await commandService.ExecuteAsync(context, pos, null);
}

Task OnReady()
{
    // 봇이 시작 준비되었을 때 관련 처리

    return Task.CompletedTask;
}

Task OnClientLogReceived(LogMessage arg)
{
    // Discord 로그 기록
    Console.WriteLine(arg.ToString());

    return Task.CompletedTask;
}

/// <summary>
/// 명령어 모듈. commandService.AddModulesAsync()에 의해 추가됨
/// </summary>
public class RemoteModule : ModuleBase<SocketCommandContext>
{
    [Command("echo")]
    [Alias("!")]
    public Task Echo(string message) => Send(message);

    private Task Send(string message) => Context.Channel.SendMessageAsync(message);
}
```

코드를 실행하면 `OnClientLogReceived()`에 의해 다음처럼 로그를 확인할 수 있고

![image](https://discourse-dotnetdev-upload.ewr1.vultrobjects.com/original/2X/d/d1fe97bcc993a8fa762e61f4e6bfe14f923787d4.png align="left")

Discord 채널에서 봇에게 명령을 줄 수 있게 됩니다.

![image](https://discourse-dotnetdev-upload.ewr1.vultrobjects.com/optimized/2X/d/d2c58ad5a3bbdb995d979aa9bc0a8a27b741dd08_2_690x419.png align="left")

좀 더 상세한 사용 방법은 [Discord.NET](http://Discord.NET) 제공 문서를 참조하세요!

[https://discordnet.dev/guides/introduction/intro.html](https://discordnet.dev/guides/introduction/intro.html)