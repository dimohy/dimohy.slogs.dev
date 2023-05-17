---
title: "구현을 쉽게 교체할 수 있는 추상화? 그렇지 않습니다. | Derek Comartin"
datePublished: Wed May 17 2023 14:49:32 GMT+0000 (Coordinated Universal Time)
cuid: clhrtjylu000209lc118c321t
slug: abstractions-to-easily-swap-implementations-not-so-fast
canonical: https://codeopinion.com/abstractions-to-easily-swap-implementations-not-so-fast/
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1684334910617/3c843d8f-f307-4944-922e-53a91c0b9659.webp
tags: net, dotnet

---

> **Derek Comartin**님의 [**Abstractions to easily swap implementations? Not so fast.**](https://codeopinion.com/abstractions-to-easily-swap-implementations-not-so-fast/)글을 번역하였습니다.

---

> **스폰서**: 복잡한 소프트웨어 시스템을 구축하시나요? NServiceBus를 통해 메시지 큐를 사용하여 느슨한 연결을 달성하는 소프트웨어 시스템을 더 쉽게 설계, 빌드 및 관리하는 방법을 알아보세요. [무료로 시작](https://go.particular.net/codeopinion)하세요.

추상화를 만드는 이유는 무엇인가요? 한 가지 이유는 기본 개념과 API를 단순화하기 위해서입니다. 또 다른 일반적인 이유는 내부 구현이 변경될 수 있기 때문일 수 있습니다. 사실일 수도 있지만, 생각만큼 간단하지는 않습니다. API를 설계할 때 고려해야 할 몇 가지 예를 들어보겠습니다.

## 유튜브

이 포스팅의 모든 내용을 보여주는 이 동영상을 비롯하여 제가 포스팅에 첨부하는 모든 종류의 콘텐츠를 게시하는 [유튜브 채널](https://www.youtube.com/channel/UC3RKA4vunFAfrfxiJhPEplw)을 확인하세요.

%[https://www.youtube.com/watch?v=qeJeS-7luo8] 

## 예상 동작

리포지토리는 매우 일반적이고 (일반적으로) 이해되는 개념이므로 이 글의 대부분에서 리포지토리를 예로 사용하겠습니다.

```csharp
public interface IRepository<T>
{
    Task<T?> GetById<TId>(TId id, CancellationToken cancellationToken = default) where TId : notnull;
    Task<T?> GetBySpec(ISpecification<T> specification, CancellationToken cancellationToken = default);
    Task<List<T>> List(CancellationToken cancellationToken = default);
    Task<List<T>> List(ISpecification<T> specification, CancellationToken cancellationToken = default);
    
    Task<T> Add(T entity, CancellationToken cancellationToken = default);
    Task<IEnumerable<T>> AddRange(IEnumerable<T> entities, CancellationToken cancellationToken = default);
    Task Update(T entity, CancellationToken cancellationToken = default);
    Task DeleteAsync(T entity, CancellationToken cancellationToken = default);
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}
```

따라서 이 리포지토리 인터페이스를 통해 데이터를 가져오고 데이터 저장소에서 데이터를 추가/업데이트/삭제하는 다양한 방법을 정의할 수 있습니다. 주목해야 할 한 가지 중요한 측면은 이러한 추상화는 우리가 어떤 면에서 컬렉션 기반 데이터 집합에 기반하고 있다는 것을 정의한다는 것입니다.

구현의 경우, 일반적으로 문서 저장소 등 사용 중인 데이터베이스 유형에 따라 엔티티 프레임워크와 같은 ORM을 사용하는 것과 데이터베이스 SDK를 사용하는 것이 있습니다.

다른 구현에 사용되는 일반적인 예는 캐시입니다. 캐시는 메모리 또는 분산 캐시에 있을 수 있지만, 이와 같은 추상화가 유용한 이유를 설명하는 예로 자주 사용됩니다.

```csharp
public class CacheRepository<T> : IRepository<T>
{
    private readonly IMemoryCache _cache;
    private readonly EfRepository<T> _repository;

    public CacheRepository(IMemoryCache cache, EfRepository<T> repository)
    {
        _cache = cache;
        _repository = repository;
    }

    public async Task<List<T>> List(ISpecification<T> specification, CancellationToken cancellationToken = default)
    {
        return await _memoryCache.GetOrCreateAsync(specification.CacheKey, async entry =>
        {
            entry.SlidingExpiration = TimeSpan.FromMinutes(15);
            return await _repository.List(specification, cancellationToken);
        });
    }
    
    // The rest of the implementation....
}
```

위의 일반적인 IRepository의 예에 따르면, 데이터베이스에 도달하는 다른 구현과 동일한 예상 동작을 지원하지 않으므로 끔찍한 아이디어가 될 수 있습니다. 왜 그럴까요? 캐시가 본질적으로 부실하기 때문입니다.

호출 코드(소비자)가 이전에 DB 직접 구현에서 제공한 IRepository에 의존하는 경우 이를 캐시된 리포지토리로 변경하면 예상되는 동작이 소비자에 맞게 변경됩니다. 예를 들어, 캐시가 일관적이지 않기 때문에 직접 읽으려는 경우 읽을 수 없게 됩니다.

기대치가 중요합니다. 리포지토리가 완전히 일관성이 있다고 생각했는데 구현을 바꿨는데 그렇지 않다면 몇 가지 해결해야 할 문제가 있을 수 있습니다. 이 상황에서는 다른 인터페이스(ICachedReadRepository)를 통해 데이터를 캐시하거나 호출된 메서드의 매개변수/옵션을 통해 캐시된(오래된) 데이터를 반환해도 괜찮다는 것을 명시하는 것이 더 좋습니다.

## 지원 모델

리포지토리 예제를 계속 진행하기 위해 관계형 데이터베이스 또는 문서 저장소를 사용하여 엔티티의 현재 상태를 기반으로 엔티티를 반환하고 보존하고 있습니다. 하지만 상태를 유지하는 방법으로 이벤트 소싱을 사용하고자 한다면 이러한 추상화가 어떻게 유지될까요?

관계형 데이터베이스에서 제품 엔티티의 현재 상태를 유지하는 예입니다.

![](https://codeopinion.com/wp-content/uploads/2023/05/1-1024x185.png align="left")

이벤트 소싱을 사용하는 동일한 엔티티가 스트림에서 이벤트를 지속할 수 있습니다.

![](https://codeopinion.com/wp-content/uploads/2023/05/2-1024x183.png align="left")

이것은 상태를 지속하는 매우 다른 방식입니다. 생성하는 추상화는 염두에 두고 있는 구현을 기반으로 합니다. 구현을 만든 후에 추상화를 만드는 경우가 일반적입니다.

여러분이 만드는 추상화에는 특정 모델이 염두에 두고 있습니다. 모든 모델에 적합하지는 않을 수 있습니다. 이는 전적으로 괜찮습니다. 하지만 추상화는 구현에 대한 이해를 기반으로 한다는 점을 기억하세요.

이벤트 소스 리포지토리와 구현의 모습은 다음과 같습니다. 테이블이나 컬렉션이 아닌 이벤트 스트림을 다루기 때문에 이전 리포지토리와는 매우 다릅니다.

```csharp
public interface IEventStreamRepository<T>
{
    Task<T> Get(string id);
    Task Save(T entity);
}

public class WarehouseProductEventStoreStream : IEventStreamRepository<WarehouseProduct>
{
    private const int SnapshotInterval = 100;
    private readonly IEventStoreConnection _connection;

    public static async Task<WarehouseProductEventStoreStream> Factory()
    {
        var connectionSettings = ConnectionSettings.Create()
            .KeepReconnecting()
            .KeepRetrying()
            .SetHeartbeatTimeout(TimeSpan.FromMinutes(5))
            .SetHeartbeatInterval(TimeSpan.FromMinutes(1))
            .DisableTls()
            .DisableServerCertificateValidation()
            .SetDefaultUserCredentials(new UserCredentials("admin", "changeit"))
            .Build();

        var conn = EventStoreConnection.Create(connectionSettings, new IPEndPoint(IPAddress.Parse("127.0.0.1"), 1113));
        await conn.ConnectAsync();

        return new WarehouseProductEventStoreStream(conn);
    }

    private WarehouseProductEventStoreStream(IEventStoreConnection connection)
    {
        _connection = connection;
    }

    public async Task<WarehouseProduct> Get(string sku)
    {
        var streamName = GetStreamName(sku);
        var snapshot = await GetSnapshot(sku);

        var warehouseProduct = new WarehouseProduct(sku, snapshot.State);

        StreamEventsSlice currentSlice;
        var nextSliceStart = snapshot.Version + 1;
        do
        {
            currentSlice = await _connection.ReadStreamEventsForwardAsync(
                streamName,
                nextSliceStart,
                200,
                false
            );

            nextSliceStart = currentSlice.NextEventNumber;

            foreach (var evnt in currentSlice.Events)
            {
                var eventObj = DeserializeEvent(evnt);
                warehouseProduct.ApplyEvent(eventObj);
            }
        } while (!currentSlice.IsEndOfStream);

        return warehouseProduct;
    }

    public async Task Save(WarehouseProduct warehouseProduct)
    {
        var streamName = GetStreamName(warehouseProduct.Sku);

        var newEvents = warehouseProduct.GetUncommittedEvents();
        long version = 0;
        foreach (var evnt in newEvents)
        {
            var data = Encoding.UTF8.GetBytes(JsonConvert.SerializeObject(evnt));
            var metadata = Encoding.UTF8.GetBytes("{}");
            var evt = new EventData(Guid.NewGuid(), evnt.EventType, true, data, metadata);
            var result = await _connection.AppendToStreamAsync(streamName, ExpectedVersion.Any, evt);
            version = result.NextExpectedVersion;
        }

        if ((version + 1) >= SnapshotInterval && (version + 1) % SnapshotInterval == 0)
        {
            await AppendSnapshot(warehouseProduct, version);
        }
    }

    private string GetStreamName(string sku)
    {
        return $"WarehouseProduct-{sku}";
    }

    private string GetSnapshotStreamName(string sku)
    {
        return $"WarehouseProduct-Snapshot-{sku}";
    }

    private async Task<Snapshot> GetSnapshot(string sku)
    {
        var streamName = GetSnapshotStreamName(sku);
        var slice = await _connection.ReadStreamEventsBackwardAsync(streamName, (long)StreamPosition.End, 1, false);
        if (slice.Events.Any())
        {
            var evnt = slice.Events.First();
            var json = Encoding.UTF8.GetString(evnt.Event.Data);
            return JsonConvert.DeserializeObject<Snapshot>(json);
        }

        return new Snapshot();
    }

    private IEvent DeserializeEvent(ResolvedEvent evnt)
    {
        var json = Encoding.UTF8.GetString(evnt.Event.Data);
        return evnt.Event.EventType switch
        {
            "InventoryAdjusted" => JsonConvert.DeserializeObject<InventoryAdjusted>(json),
            "ProductShipped" => JsonConvert.DeserializeObject<ProductShipped>(json),
            "ProductReceived" => JsonConvert.DeserializeObject<ProductReceived>(json),
            _ => throw new InvalidOperationException($"Unknown Event: {evnt.Event.EventType}")
        };
    }

    private async Task AppendSnapshot(WarehouseProduct warehouseProduct, long version)
    {
        var streamName = GetSnapshotStreamName(warehouseProduct.Sku);
        var state = warehouseProduct.GetState();

        var snapshot = new Snapshot
        {
            State = state,
            Version = version
        };

        var metadata = Encoding.UTF8.GetBytes("{}");
        var data = Encoding.UTF8.GetBytes(JsonConvert.SerializeObject(snapshot));
        var evt = new EventData(Guid.NewGuid(), "snapshot", true, data, metadata);
        await _connection.AppendToStreamAsync(streamName, ExpectedVersion.Any, evt);
    }

    public void Dispose()
    {
        _connection?.Dispose();
    }
}
```

현재 상태를 유지하는 것과 비교하여 이벤트 스트림에 이벤트를 추가하는 근본적인 아이디어는 추상화를 변경합니다.

생성한 추상화는 특정 모델에 맞게 조정됩니다.

## 구현 누수

단일 구현을 염두에 두고 추상화를 만들면 해당 모델과 해당 구현을 중심으로 추상화를 구축하게 됩니다. 단일 구현을 기반으로 추상화를 사후에 만들면 결국 같은 지점에 도달하게 됩니다.

이것은 잘못된 것이 아닙니다. 하지만 추상화가 구체적인 구현을 매끄럽게 교체할 수 있는 마법 같은 도구라는 생각은 사실이 아닙니다. 그렇다면 모든 구현에 대해 추상화를 만들어야 하는 이유는 무엇일까요? 추상화의 기반이 되는 구현은 단 하나뿐이기 때문에 잘못 이해하고 있을 가능성이 높습니다.

많은 구현을 구축한 후에야 추상화를 통해 추상화하려는 기본 개념을 단순화할 수 있는 경우가 많습니다.

## 참여하세요!

제 [YouTube 채널](https://www.youtube.com/channel/UC3RKA4vunFAfrfxiJhPEplw/join) 또는 [Patreon](https://www.patreon.com/codeopinion)의 개발자 레벨 회원은 비공개 Discord 서버에 액세스하여 다른 개발자와 소프트웨어 아키텍처 및 디자인에 대해 채팅하고 블로그나 YouTube에 게시하는 모든 데모 애플리케이션의 소스 코드에 액세스할 수 있습니다. 자세한 내용은 내 [Patreon](https://www.patreon.com/codeopinion) 또는 [YouTube 멤버십](https://www.youtube.com/channel/UC3RKA4vunFAfrfxiJhPEplw/join)을 확인하세요.

## 관련 링크

* [간접 참조 및 추상화의 비용은 얼마인가요?](https://codeopinion.com/whats-the-cost-of-indirection-abstractions/)
    
* [추상화 계층을 작성하지 말아야 할 때](https://codeopinion.com/when-not-to-write-an-abstraction-layer/)
    

---

%[https://codeopinion.com/abstractions-to-easily-swap-implementations-not-so-fast/]