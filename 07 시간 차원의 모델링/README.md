# 07 시간 차원의 모델링

- 도메인 모델 패턴은 복잡한 비즈니스 로직을 갖는 핵심 하위 도메인에 적용됩니다.
- 이벤트 소싱은 도메인 모델 패턴은 도메인 모델 패턴과 동일한 전술적 패턴(밸류 오브젝트, 애그리게이트, 도메인 이벤트)를 사용합니다.
- 대신, 이벤트 소싱 패턴을 사용해 애그리게이트 상태를 저장합니다. 애그리게이트의 상태를 유지하는 대신, 변경사항을 설명하는 도메인 이벤트를 생성하고 원천 데이터로 여깁니다.

## 이벤트 소싱

> 순서도는 필요 없게 된다. 테이블의 정보만으로도 명확하기 때문이다.

- 테이블 스키마만 봐도 많은 정보를 얻을 수 있습니다: 유비쿼터스 언어, 속성의 의미나 관계 등
- 하지만, 일반적인 테이블 스키마에는 시간축이 없습니다. 오로지 데이터의 현재 상태뿐입니다.
- 데이터가 현재 상태에 도달한 이력에 대한 이야기가 없고, 데이터의 수명주기 동안 어떤 일이 발생했는지 분석할 수도 없습니다.
- 이는 비즈니스 문제를 해결하기 위한, 즉 질문에 답할 수 있는 정보가 없다는 의미입니다.
- 그렇지만 데이터 분석 및 경험에 기반한 비즈니스 최적화는 매우 중요합니다.

이벤트 소싱 패턴

- 시간 차원을 도입합니다.
- 애그리게이트의 상태를 저장하는 대신, 애그리게이트의 상태 변화를 설명하는 도메인 이벤트를 저장합니다.
- 고객의 상태는 이 도메인 이벤트를 프로젝션해서 얻을 수 있습니다.
  - 쓰기 모델에서 이벤트 소싱 시스템에 이력 형태로 저장한 데이터를,
  - **다양한** 읽기 모델을 원하는 시점의 데이터를 원하는 형태로 추출할 수 있습니다.

```json
[
    {
        "lead-id": 12,
        "event-id": 0,
        "event-type": "lead-initialized",
        "first-name": "Casey",
        "last-name": "David",
        "phone-number": "555-2951",
        "timestamp": "2020-05-20T09:52:55.95Z"
    },
    {
        "lead-id": 12,
        "event-id": 1,
        "event-type": "contacted",
        "timestamp": "2020-05-20T12:32:08.24Z"
    },
    {
        "lead-id": 12,
        "event-id": 2,
        "event-type": "followup-set",
        "followup-on": "2020-05-27T12:00:00.00Z",
        "timestamp": "2020-05-20T12:32:08.24Z"
    },
    {
        "lead-id": 12,
        "event-id": 3,
        "event-type": "contact-details-updated",
        "first-name": "Casey",
        "last-name": "Davis",
        "phone-number": "555-8101",
        "timestamp": "2020-05-20T12:32:08.24Z"
    },
    {
        "lead-id": 12,
        "event-id": 4,
        "event-type": "contacted",
        "timestamp": "2020-05-27T12:02:12.51Z"
    },
    {
        "lead-id": 12,
        "event-id": 5,
        "event-type": "order-submitted",
        "payment-deadline": "2020-05-30T12:02:12.51Z",
        "timestamp": "2020-05-27T12:02:12.51Z"
    },
    {
        "lead-id": 12,
        "event-id": 6,
        "event-type": "payment-confirmed",
        "status": "converted",
        "timestamp": "2020-05-27T12:38:44.12Z"
    }
]
```

모델 프로젝션

```csharp
public class LeadSearchModelProjection
{
    public long LeadId { get; private set; }
    public HashSet<string> FirstNames { get; private set; }
    public HashSet<string> LastNames { get; private set; }
    public HashSet<PhoneNumber> PhoneNumbers { get; private set; }
    public int Version { get; private set; }

    public void Apply(LeadInitialized @event)
    {
        LeadId = @event.LeadId;
        FirstNames = new HashSet < string > ();
        LastNames = new HashSet < string > ();
        PhoneNumbers = new HashSet < PhoneNumber > ();
        FirstNames.Add(@event.FirstName);
        LastNames.Add(@event.LastName);
        PhoneNumbers.Add(@event.PhoneNumber);
        Version = 0;
    }

    public void Apply(ContactDetailsChanged @event)
    {
        FirstNames.Add(@event.FirstName);
        LastNames.Add(@event.LastName);
        PhoneNumbers.Add(@event.PhoneNumber);
        Version += 1;
    }

    public void Apply(Contacted @event)
    {
        Version += 1;
    }

    public void Apply(FollowupSet @event)
    {
        Version += 1;
    }

    public void Apply(OrderSubmitted @event)
    {
        Version += 1;
    }

    public void Apply(PaymentConfirmed @event)
    {
        Version += 1;
    }
}
```

- 각 이벤트를 순회하며, 종류에 따라 적절한 Apply를 호출해 변화를 축적하면 모델의 현재 상태가 만들어집니다.
- 이벤트를 일부만 반영해 특정 시점의 상태를 얻을 수도 있습니다.

버전 필드

- 위에서 말한 '시간 여행'에 필요합니다.
- 버전 5(Version = 4)의 엔티티 상태가 필요한 경우, 5개의 이벤트를 순회하며 변화를 축적하면 됩니다.

모델링 커스터마이즈

- 프로젝션을 어떻게 작성하느냐에 따라 다양한 형태의 모델을 얻을 수 있습니다. 이를 통해 다양한 비즈니스 질의응답이 가능해집니다.

### 검색 모델

- 예컨대 과거 이력들을 프로젝션에 포함시킬 수 있습니다. 위에 있는 코드는 인적사항의 이력들을 누적합니다.

```text
LeadId: 12
FirstNames: ["Casey"]
LastNames: ["David", "Davis"]
PhoneNumbers: ["555-2951", "555-8101"]
Version: 6
```

### 분석 모델

- 이런 비즈니스 상황을 가정해봅시다.
  - 비즈니스 인텔리전스 부서에서
  - 리드 데이터별
  - 후속 전화 예약의 개수와
  - 나중에 필터링에 사용할 수 있는 상태를 요구했습니다.
- 이를 위해 프로젝션을 다음과 같이 작성할 수 있습니다.

```csharp
public class AnalysisModelProjection
{
    public long LeadId { get; private set; }
    public int Followups { get; private set; }
    public LeadStatus Status { get; private set; }
    public int Version { get; private set; }

    public void Apply(LeadInitialized @event)
    {
        LeadId = @event.LeadId;
        Followups = 0;
        Status = LeadStatus.NEW_LEAD;
        Version = 0;
    }

    public void Apply(Contacted @event)
    {
        Version += 1;
    }

    public void Apply(FollowupSet @event)
    {
        Status = LeadStatus.FOLLOWUP_SET;
        Followups += 1;
        Version += 1;
    }

    public void Apply(ContactDetailsChanged @event)
    {
        Version += 1;
    }

    public void Apply(OrderSubmitted @event)
    {
        Status = LeadStatus.PENDING_PAYMENT;
        Version += 1;
    }

    public void Apply(PaymentConfirmed @event)
    {
        Status = LeadStatus.CONVERTED;
        Version += 1;
    }
}
```

- 그러면 다음과 같은 모델을 얻을 수 있습니다.

```text
LeadId: 12
Followups: 1
Status: LeadStatus.CONVERTED
Version: 6
```
