---
title: "제2부: CQRS, 의존성 주입, 그리고 에필로그"
description: "Cosmic Python Part 2 - 읽기/쓰기 분리, 의존성 주입으로 아키텍처를 완성하고, 현실 프로젝트에 적용하는 방법까지"
date: 2026-02-13 10:00:00 +0900
categories: [CS, Common]
tags: [Computer Science, Architecture, CQRS, DDD, Cosmic Python]
image: https://www.cosmicpython.com/images/seagull_nebula_jpl.jpeg
---

# Chapter 12: CQRS (Command-Query Responsibility Segregation)

지금까지 우리는 커맨드와 이벤트를 분리하고, 메시지 버스로 유스케이스를 통합했다.
그런데 한 가지 의문이 남는다.

> **도메인 모델은 비즈니스 로직을 위해 설계된 건데, 이걸 데이터 조회에도 그대로 쓰는 게 맞을까?**

답은 **"아니다"**이다. 읽기와 쓰기는 근본적으로 다르기 때문이다.

---

## 왜 읽기와 쓰기를 분리해야 할까?

| 구분 | 쓰기(Command) | 읽기(Query) |
|------|-------------|------------|
| **목적** | 상태 변경, 비즈니스 규칙 적용 | 데이터 조회, 화면 표시 |
| **복잡도** | 복잡한 도메인 로직 필요 | 단순한 데이터 반환 |
| **일관성** | 트랜잭션 일관성 필수 | 약간의 지연(stale) 허용 가능 |
| **캐싱** | 캐싱 불가 | 캐싱 가능 |

도메인 모델은 **"비즈니스가 어떻게 동작하는가"**를 표현한 것이지, **"데이터를 어떻게 보여줄 것인가"**를 표현한 게 아니다.

그런데 읽기 요청에 도메인 모델을 쓰면? Aggregate를 통째로 로딩하고, 관계를 탐색하고, 필터링하고… 단순히 "주문 목록 보여줘"에 불필요한 오버헤드가 붙는다.

> 사실 완벽하게 동기화된 시스템이라도, 페이지가 렌더링되는 순간 이미 데이터는 과거의 것이다.
> **"사용자가 화면을 보는 시점"과 "실제 데이터 상태"는 이미 다르다.**

이게 CQRS의 출발점이다. **어차피 읽기 데이터는 약간 stale할 수밖에 없으니, 쓰기 모델과 분리해서 각각 최적화하자.**

---

## 읽기 모델의 4가지 구현 방법

### 방법 A: Repository 패턴 그대로 사용

```python
def allocations(orderid: str, uow: AbstractUnitOfWork):
    with uow:
        products = uow.products.for_order(orderid=orderid)
        batches = [b for p in products for b in p.batches]
        return [
            {"sku": b.sku, "batchref": b.reference}
            for b in batches
            if orderid in b.orderids
        ]
```

기존 추상화를 재사용하니 편하지만, **Python 레벨에서 필터링**하는 것은 비효율적이다.
새로운 헬퍼 메서드(`for_order`)도 필요하고, 전체적으로 어색하다.

### 방법 B: ORM 쿼리

```python
def allocations(orderid: str, uow: AbstractUnitOfWork):
    with uow:
        batches = uow.session.query(model.Batch).join(
            model.OrderLine, model.Batch._allocations
        ).filter(
            model.OrderLine.orderid == orderid
        )
        return [
            {"sku": b.sku, "batchref": b.batchref}
            for b in batches
        ]
```

ORM 기능을 활용하니 좀 나아졌지만, **SELECT N+1 문제**가 발생할 수 있고, 복잡한 쿼리는 ORM 문법이 오히려 SQL보다 읽기 어렵다.

### 방법 C: 순수 SQL

```python
def allocations(orderid: str, uow: SqlAlchemyUnitOfWork):
    with uow:
        results = uow.session.execute(
            """
            SELECT ol.sku, b.reference
            FROM allocations AS a
            JOIN batches AS b ON a.batch_id = b.id
            JOIN order_lines AS ol ON a.orderline_id = ol.id
            WHERE ol.orderid = :orderid
            """,
            dict(orderid=orderid),
        )
    return [{"sku": sku, "batchref": batchref}
            for sku, batchref in results]
```

SQL을 직접 쓰면 **성능을 완전히 제어**할 수 있다.
읽기 전용이니까 도메인 모델을 거칠 필요가 없다. SQL이면 충분하다.

### 방법 D: 비정규화된 읽기 전용 테이블 + 이벤트 핸들러 (CQRS 본격 적용)

이것이 CQRS의 진짜 모습이다. **읽기 전용 테이블을 별도로 만들고, 이벤트 핸들러가 자동으로 업데이트**한다.

```python
# 읽기 전용 테이블 정의
allocations_view = Table(
    "allocations_view",
    metadata,
    Column("orderid", String(255)),
    Column("sku", String(255)),
    Column("batchref", String(255)),
)
```

```python
# 읽기는 이렇게 단순해진다
def allocations(orderid: str, uow: SqlAlchemyUnitOfWork):
    with uow:
        results = uow.session.execute(
            "SELECT sku, batchref FROM allocations_view WHERE orderid = :orderid",
            dict(orderid=orderid),
        )
    return [{"sku": sku, "batchref": batchref} for sku, batchref in results]
```

조인도 없고, 도메인 모델도 안 거치고, **단순한 SELECT 하나**로 끝난다.

---

## 이벤트 핸들러로 읽기 모델 갱신

읽기 전용 테이블은 누가 채우는가? **이벤트 핸들러**가 한다.

```python
def add_allocation_to_read_model(
    event: events.Allocated,
    uow: unit_of_work.SqlAlchemyUnitOfWork,
):
    with uow:
        uow.session.execute(
            """
            INSERT INTO allocations_view (orderid, sku, batchref)
            VALUES (:orderid, :sku, :batchref)
            """,
            dict(orderid=event.orderid, sku=event.sku,
                 batchref=event.batchref),
        )
        uow.commit()

def remove_allocation_from_read_model(
    event: events.Deallocated,
    uow: unit_of_work.SqlAlchemyUnitOfWork,
):
    with uow:
        uow.session.execute(
            """
            DELETE FROM allocations_view
            WHERE orderid = :orderid AND sku = :sku
            """,
            dict(orderid=event.orderid, sku=event.sku),
        )
        uow.commit()
```

메시지 버스에 핸들러를 등록한다.

```python
EVENT_HANDLERS = {
    events.Allocated: [
        handlers.publish_allocated_event,
        handlers.add_allocation_to_read_model,     # 읽기 모델 갱신
    ],
    events.Deallocated: [
        handlers.remove_allocation_from_read_model,  # 읽기 모델 삭제
    ],
}
```

**쓰기가 발생하면 → 이벤트가 발행되고 → 읽기 모델이 자동으로 갱신**된다.

---

## 읽기 모델의 유연성: 저장소 교체

이벤트 기반으로 읽기 모델을 구축하면, **저장소를 자유롭게 바꿀 수 있다.**
예를 들어 Redis로 교체하면 이렇다.

```python
# Redis에 읽기 모델 저장
def add_allocation_to_read_model(event: events.Allocated, _):
    redis_client.hset(event.orderid, event.sku, event.batchref)

# Redis에서 읽기
def allocations(orderid: str):
    batches = redis_client.hgetall(orderid)
    return [
        {"batchref": b.decode(), "sku": s.decode()}
        for s, b in batches.items()
    ]
```

비즈니스 로직은 전혀 건드리지 않고, **읽기 핸들러만 바꾸면 된다.**

---

## 읽기 모델이 깨지면?

"읽기 전용 테이블이 쓰기 모델과 안 맞으면 어떡하지?" 라는 걱정이 당연히 든다.

하지만 읽기 모델은 **재구축이 쉽다.** 쓰기 모델의 현재 상태를 기반으로 이벤트를 다시 재생(replay)하면 읽기 모델을 처음부터 다시 만들 수 있다.

이것이 CQRS의 또 다른 장점이다. **읽기 모델은 언제든 버리고 다시 만들 수 있는 파생 데이터**다.

---

## Post/Redirect/Get 패턴

CQRS와 함께 자주 사용되는 웹 패턴이 있다.

1. **POST**: 커맨드를 보내서 쓰기 작업을 수행 → 202 Accepted 반환
2. **Redirect**: 쓰기가 완료되면 읽기 URL로 리다이렉트
3. **GET**: 읽기 전용 엔드포인트에서 결과를 조회

```python
@app.route("/allocations/<orderid>", methods=["GET"])
def allocations_view_endpoint(orderid):
    uow = unit_of_work.SqlAlchemyUnitOfWork()
    result = views.allocations(orderid, uow)
    if not result:
        return "not found", 404
    return jsonify(result), 200
```

쓰기와 읽기가 **완전히 다른 경로**를 탄다.

---

## Chapter 12 트레이드오프 정리

| 접근 방식 | 장점 | 단점 |
|-----------|------|------|
| Repository 그대로 사용 | 단순, 기존 코드 재사용 | 복잡한 쿼리에 비효율적 |
| ORM 쿼리 | ORM 설정 재활용 | SELECT N+1, 복잡한 문법 |
| 순수 SQL | 성능 완전 제어 | 스키마 변경 시 수정 필요 |
| 비정규화 테이블 (CQRS) | 읽기가 극도로 단순하고 빠름 | 쓰기 시 약간의 추가 비용 |
| 별도 저장소 (Redis 등) | 수평 확장, 독립 최적화 | 일관성 지연, 복잡도 증가 |

---

# Chapter 13: 의존성 주입과 부트스트래핑(Dependency Injection and Bootstrapping)

지금까지 우리 아키텍처에는 한 가지 지저분한 점이 있었다.
**의존성이 여기저기 흩어져 있다**는 것이다.

Flask 엔드포인트에서 `SqlAlchemyUnitOfWork()`를 직접 생성하고, Redis 컨슈머에서도 같은 짓을 하고, 테스트에서는 `FakeUnitOfWork()`를 만들고…

> **의존성을 한 곳에서 조립하고, 핸들러에는 필요한 것만 주입하자.**

이것이 Chapter 13의 핵심이다.

---

## 문제: 암묵적 의존성

기존 핸들러를 보자.

```python
from allocation.adapters import email

def send_out_of_stock_notification(event: events.OutOfStock):
    email.send("stock@made.com", f"Out of stock for {event.sku}")
```

`email` 모듈을 **직접 import**하고 있다. 테스트하려면?

```python
with mock.patch("allocation.adapters.email.send") as mock_send:
    ...
```

`mock.patch`를 써야 한다. 이건 여러 문제를 만든다.

- **모든 테스트**에서 `email.send`를 mock해야 한다
- **import 경로가 바뀌면** 모든 mock이 깨진다
- mock이 쌓이면 **유지보수 비용**이 급격히 올라간다

> Python의 Zen: **"Explicit is better than implicit."**

---

## 해결: 명시적 의존성 주입

핸들러가 필요한 의존성을 **파라미터로 받게** 바꾼다.

```python
# Before: 암묵적 (직접 import)
def send_out_of_stock_notification(event: events.OutOfStock):
    email.send("stock@made.com", f"Out of stock for {event.sku}")

# After: 명시적 (파라미터로 주입)
def send_out_of_stock_notification(event: events.OutOfStock, send_mail):
    send_mail("stock@made.com", f"Out of stock for {event.sku}")
```

테스트할 때 mock.patch가 필요 없다. 그냥 **가짜 함수를 넘기면 된다.**

---

## 부트스트랩 스크립트: 조합의 뿌리(Composition Root)

의존성을 한 곳에서 조립하는 것이 **부트스트랩 스크립트**다.

```python
def bootstrap(
    start_orm: bool = True,
    uow: unit_of_work.AbstractUnitOfWork = unit_of_work.SqlAlchemyUnitOfWork(),
    send_mail = email.send,
    publish = redis_eventpublisher.publish,
) -> MessageBus:

    if start_orm:
        orm.start_mappers()

    dependencies = {"uow": uow, "send_mail": send_mail, "publish": publish}

    injected_event_handlers = {
        event_type: [
            inject_dependencies(handler, dependencies)
            for handler in event_handlers
        ]
        for event_type, event_handlers in handlers.EVENT_HANDLERS.items()
    }

    injected_command_handlers = {
        command_type: inject_dependencies(handler, dependencies)
        for command_type, handler in handlers.COMMAND_HANDLERS.items()
    }

    return MessageBus(
        uow=uow,
        event_handlers=injected_event_handlers,
        command_handlers=injected_command_handlers,
    )
```

핵심 아이디어를 정리하면 이렇다.

1. **기본값은 프로덕션 의존성**이다 (`SqlAlchemyUnitOfWork`, `email.send` 등)
2. **테스트에서는 가짜를 주입**한다 (`FakeUnitOfWork`, `lambda *args: None` 등)
3. **핸들러에 의존성을 자동으로 주입**한다 (`inject_dependencies`)
4. **조립된 메시지 버스를 반환**한다

---

## 의존성 자동 주입: 시그니처 검사

핸들러 함수의 파라미터 이름을 검사해서, **필요한 의존성만 자동으로 주입**한다.

```python
import inspect

def inject_dependencies(handler, dependencies):
    params = inspect.signature(handler).parameters
    deps = {
        name: dependency
        for name, dependency in dependencies.items()
        if name in params
    }
    return lambda message: handler(message, **deps)
```

핸들러가 `uow`만 필요하면 `uow`만, `send_mail`도 필요하면 `send_mail`도 주입된다.
**핸들러마다 수동으로 의존성을 연결할 필요가 없다.**

---

## 메시지 버스가 클래스로 변환

부트스트랩에서 조립된 핸들러를 들고 있으려면, 메시지 버스도 **인스턴스**가 되어야 한다.

```python
class MessageBus:
    def __init__(self, uow, event_handlers, command_handlers):
        self.uow = uow
        self.event_handlers = event_handlers
        self.command_handlers = command_handlers

    def handle(self, message):
        self.queue = [message]
        while self.queue:
            message = self.queue.pop(0)
            if isinstance(message, events.Event):
                self.handle_event(message)
            elif isinstance(message, commands.Command):
                self.handle_command(message)

    def handle_event(self, event):
        for handler in self.event_handlers[type(event)]:
            handler(event)  # 의존성은 이미 주입되어 있다
            self.queue.extend(self.uow.collect_new_events())
```

핸들러 호출 시 **인자가 `event` 하나**뿐이다. 나머지 의존성은 부트스트랩에서 이미 바인딩되어 있기 때문이다.

---

## 진입점(Entrypoint)이 깔끔해진다

### Flask

```python
# Before
from allocation import views
app = Flask(__name__)
orm.start_mappers()

@app.route("/add_batch", methods=["POST"])
def add_batch():
    cmd = commands.CreateBatch(...)
    uow = unit_of_work.SqlAlchemyUnitOfWork()  # 직접 생성
    messagebus.handle(cmd, uow)

# After
from allocation import bootstrap, views
app = Flask(__name__)
bus = bootstrap.bootstrap()  # 한 줄로 모든 의존성 조립

@app.route("/add_batch", methods=["POST"])
def add_batch():
    cmd = commands.CreateBatch(...)
    bus.handle(cmd)  # UoW를 넘길 필요 없다
```

---

## 테스트에서의 활용

### 통합 테스트: SQLite + 가짜 외부 서비스

```python
@pytest.fixture
def sqlite_bus(sqlite_session_factory):
    bus = bootstrap.bootstrap(
        start_orm=True,
        uow=unit_of_work.SqlAlchemyUnitOfWork(sqlite_session_factory),
        send_mail=lambda *args: None,    # 이메일 안 보냄
        publish=lambda *args: None,       # Redis 발행 안 함
    )
    yield bus
```

### 단위 테스트: 전부 가짜

```python
def bootstrap_test_app():
    return bootstrap.bootstrap(
        start_orm=False,                   # ORM 매핑 안 함
        uow=FakeUnitOfWork(),
        send_mail=lambda *args: None,
        publish=lambda *args: None,
    )
```

**mock.patch가 한 줄도 없다.** 명시적으로 가짜를 넣어줄 뿐이다.

---

## 어댑터 만들기: ABC → 구현 → 가짜 → 테스트

의존성 주입의 실전 워크플로우는 이렇다.

### 1단계: 추상 클래스(Port) 정의

```python
class AbstractNotifications(abc.ABC):
    @abc.abstractmethod
    def send(self, destination, message):
        raise NotImplementedError
```

### 2단계: 프로덕션 구현(Adapter)

```python
class EmailNotifications(AbstractNotifications):
    def __init__(self, smtp_host=DEFAULT_HOST, port=DEFAULT_PORT):
        self.server = smtplib.SMTP(smtp_host, port=port)

    def send(self, destination, message):
        msg = f"Subject: notification\n{message}"
        self.server.sendmail(
            from_addr="allocations@example.com",
            to_addrs=[destination],
            msg=msg,
        )
```

### 3단계: 가짜 구현(테스트용)

```python
class FakeNotifications(AbstractNotifications):
    def __init__(self):
        self.sent = defaultdict(list)

    def send(self, destination, message):
        self.sent[destination].append(message)
```

### 4단계: 부트스트랩에서 조립

```python
# 프로덕션
bus = bootstrap.bootstrap(notifications=EmailNotifications())

# 테스트
fake_notifications = FakeNotifications()
bus = bootstrap.bootstrap(notifications=fake_notifications)

# 테스트 후 검증
assert "Out of stock" in fake_notifications.sent["stock@made.com"][0]
```

---

## Chapter 13 트레이드오프 정리

| 장점 | 단점 |
|------|------|
| mock.patch 없이 테스트할 수 있다 | 초기에 코드가 다소 장황해 보인다 |
| 의존성이 명시적으로 드러나 가독성이 높다 | 의존성 체인이 복잡해지면 수동 DI로는 한계가 있다 |
| 프로덕션/테스트 환경 전환이 한 줄로 된다 | DI 프레임워크(Inject, Punq 등) 도입 검토가 필요할 수 있다 |
| 어댑터 교체가 자유롭다 (이메일 → Slack 등) | Flask 글로벌 bus 인스턴스는 스레드 안전성 고려 필요 |

---

# 에필로그: 현실 프로젝트에 어떻게 적용할까?

지금까지 배운 패턴들은 깔끔한 예제 프로젝트에서 진행됐다.
하지만 현실은 다르다. 이미 **수년간 쌓인 레거시 코드** 위에서 일해야 한다.

> **"바다를 끓이려 하지 마세요. 실수를 두려워하지도 마세요. 배움의 과정이 될 겁니다."**
> — David Seddon (기술 리뷰어)

---

## 핵심 원칙: 점진적으로 적용하라

전체를 다시 쓰는 건 거의 항상 실패한다. 대신 **아키텍처 세금(architecture tax)**이라는 개념을 활용하자.

> 6개월짜리 프로젝트가 있다면, "3주 정도 정리 작업이 필요합니다"라고 말하는 것이 훨씬 설득력 있다.

기능 개발과 **함께** 아키텍처를 개선하는 것이다.

---

## 단계별 적용 전략

### Phase 1: 서비스 레이어 추출

가장 먼저 할 일은 **유스케이스를 함수로 분리**하는 것이다.

- 각 유스케이스에 명령형 이름을 붙인다: "청구 요금 적용", "방치 계정 정리"
- 각 함수가 자체 트랜잭션을 시작하고, 데이터를 가져오고, 도메인을 업데이트하고, 변경을 저장한다
- **코드가 중복되어도 괜찮다.** 완벽한 코드가 아니라, 의미 있는 계층을 분리하는 게 목적이다

### Phase 2: 도메인 모델에서 I/O 분리

데이터 접근과 I/O 로직을 도메인 모델에서 **위로 끌어올린다** (핸들러/서비스 레이어로).

### Phase 3: Aggregate와 일관성 경계 식별

큰 객체 그래프를 탐색하는 코드를 발견하면, **Aggregate 경계가 잘못 설정된 것**이다.

> **양방향 링크(bidirectional links)는 Aggregate가 잘못 설계되었다는 신호다.**

객체 간 직접 참조 대신 **식별자(ID)**를 사용하도록 바꾼다.

### Phase 4: 이벤트와 메시지 버스 도입

Aggregate 간 통신을 이벤트로 전환한다. 하나의 트랜잭션에서 하나의 Aggregate만 수정하고, 나머지는 이벤트로 처리한다.

---

## Strangler Fig 패턴: 레거시 시스템 교체

기존 시스템을 한 번에 교체하는 대신, **서서히 감싸면서 교체**하는 패턴이다.

```
1. 기존 시스템의 변경을 "이벤트"로 노출한다
    │
    ▼
2. 새로운 시스템이 그 이벤트를 소비하며 자체 도메인 모델을 구축한다
    │
    ▼
3. 새 시스템이 충분히 성숙하면, 기존 시스템을 제거한다
```

**"Walking Skeleton"**부터 시작하자. 단 하나의 질문에 답할 수 있는 최소한의 시스템을 먼저 만든다.
이렇게 하면 인프라 문제(배포, 메시지 큐 연동 등)를 초기에 해결할 수 있다.

> 실제로 MADE.com 사례에서는 **수개월이 걸렸다.** 하나의 간단한 질문에 답하는 토이 도메인 모델을 만드는 것부터 시작했다.

---

## 언제 어떤 패턴을 쓸 것인가?

| 패턴 | 적용 시점 | 비고 |
|------|----------|------|
| **서비스 레이어** | 가장 먼저 | 레거시에서 유스케이스를 추출하는 출발점 |
| **Aggregate** | 큰 객체 그래프가 있을 때 | 일관성 경계를 명확히 한다 |
| **Repository** | 데이터 접근을 추상화하고 싶을 때 | 서비스 레이어와 함께 적용 |
| **Unit of Work** | 트랜잭션 관리가 필요할 때 | Repository와 함께 |
| **도메인 이벤트** | Aggregate 간 변경이 필요할 때 | 최종 일관성(eventual consistency) 허용 |
| **메시지 버스** | 이벤트를 통합 처리하고 싶을 때 | 서비스 함수와 이벤트 핸들러 통합 |
| **CQRS** | 읽기 성능이 중요할 때 | 필수는 아님, Repository로 충분하면 안 써도 된다 |
| **이벤트 소싱** | 감사/이력/리플레이가 필요할 때 | 기본 이벤트 기반과는 별개 |

```
패턴 도입 순서 (권장)

서비스 레이어 (기초)
    ↓
유스케이스 / 핸들러
    ├→ Aggregate 식별
    ├→ Repository 패턴
    └→ Unit of Work 패턴
         ↓
    도메인 이벤트
         ↓
    메시지 버스
         ├→ CQRS (선택)
         └→ 마이크로서비스 (선택)
```

---

## 주의사항

### 인프라 관련
- Redis Pub/Sub는 **메시지 유실 가능성이 있다.** 프로덕션에서는 RabbitMQ, Kafka, EventBridge 등을 고려하라
- **Outbox 패턴**으로 트랜잭션 안정성을 확보하라
- 핸들러를 **멱등(idempotent)**하게 만들어 안전한 재시도를 가능하게 하라

### 흔한 오해
- 이 패턴들을 쓰려고 **마이크로서비스가 필수는 아니다**
- **CQRS가 항상 필요하지는 않다.** Repository로 충분하면 굳이 안 써도 된다
- Django에서도 이 패턴들을 **적용할 수 있다** (Appendix D 참고)
- 이건 설계 원칙이지, **복사-붙여넣기할 코드가 아니다**

### 이벤트 스키마 관리
- 이벤트 스키마를 **문서화**하고 소비자와 공유하라
- 시간이 지나면 스키마는 **진화**한다. 호환성 정책을 미리 정해두자

---

## 도메인 모델링이 먼저다

패턴을 적용하기 전에, **도메인을 이해하는 것**이 가장 중요하다.

다음과 같은 방법들을 활용하자.
- **이벤트 스토밍(Event Storming)**: 도메인 전문가와 함께 이벤트를 도출하는 워크숍
- **CRC 모델링**: 클래스-책임-협력을 카드로 정리
- **TDD 카타**: 도메인 문제를 TDD 연습 문제처럼 풀어보기

> 엔지니어와 프로덕트 오너가 **같은 언어**로 대화하게 만드는 것. 이것이 모든 패턴의 출발점이다.

---

## 전체 시리즈 핵심 정리

Part 1부터 에필로그까지, 이 책이 말하고 싶은 것을 한 문장으로 압축하면 이렇다.

> **"도메인 모델을 중심에 두고, 인프라를 바깥으로 밀어내고, 메시지로 소통하라."**

```
                    ┌─── Flask ───┐
                    │             │
                    ▼             ▼
외부 세계 ──→ 진입점(Adapter) ──→ 메시지 버스 ──→ 핸들러
                                    │              │
                                    │         도메인 모델
                                    │              │
                                    │     Repository / UoW
                                    │              │
                                    └──── DB ◄──────┘
```

모든 패턴은 결국 **하나의 목적**을 위해 존재한다.
**비즈니스 로직이 인프라에 오염되지 않도록 보호하는 것.**

---

## 더 읽어볼 것

- *Clean Architectures in Python* — Leonardo Giordani
- *Enterprise Integration Patterns* — Hohpe & Woolf
- *Monolith to Microservices* — Sam Newman
- [bravenewgeek.com](https://bravenewgeek.com) — Tyler Treat의 분산 메시징 에세이

---

참고 자료
- [Cosmic Python - Chapter 12: CQRS](https://www.cosmicpython.com/book/chapter_12_cqrs.html)
- [Cosmic Python - Chapter 13: Dependency Injection](https://www.cosmicpython.com/book/chapter_13_dependency_injection.html)
- [Cosmic Python - Epilogue](https://www.cosmicpython.com/book/epilogue_1_how_to_get_there_from_here.html)
- [Cosmic Python 전체 목차](https://www.cosmicpython.com/book/preface.html)
