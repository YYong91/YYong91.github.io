---
title: "미니언즈 레거시(4) - 도메인 서비스 적용기"
date: 2025-10-03
categories: ["ddd"]
tags: ["DDD", "도메인서비스"]
series: ["미니언즈 레거시"]
migrated_from: "velog"
---

## 개요

이 글은 다건 메시지 수신 시 여러 Aggregate를 갱신하면서 이벤트를 한 Aggregate에만 부착하던 패턴을 개선한 과정을 설명합니다. ServiceCommand와 DomainService를 활용하여 코드 의도를 명확히 하고 책임을 적절히 분배했습니다.

---

## Before: 문제 있는 접근 방식

### 흐름
다건 정보 수신 → Aggregate 목록 조회·업데이트 → 강제 이벤트 주입 → 첫 번째 Aggregate에 귀속

### 문제점
- Application Layer Handler에서 조회, 업데이트, 이벤트 주입, UoW 커밋까지 모두 처리
- 여러 Aggregate 수정 후 이벤트는 첫 번째 Aggregate에만 부착
- 로직, 변환, 이벤트 처리 책임이 Application Layer에 과도하게 집중
- 가독성과 테스트 응집성 저하

### Before 코드 예시

```python
# Handler (Before)
updated_aggregates = []
for key, payload in incoming.items():
    agg = await repository.get_by_key_async(key)
    if not agg:
        continue

    agg.update_text(UpdateTextCommand(text="updated text"))
    updated_aggregates.append(agg)

if updated_aggregates:
    updated_aggregates[0].add_domain_event(
        AggregatesUpdatedEvent(aggregates=updated_aggregates)
    )

await unit_of_work.save_changes_async()
```

---

## After: 개선된 접근 방식

### 흐름
다건 정보 수신 → ServiceCommand 변환 → Aggregate.DomainService 처리 → 이벤트 축적 → UoW 커밋

### 핵심 개선 사항
1. **의도를 커맨드로 명확히 표현**
2. **묶음 처리 책임을 DomainService로 이동**

### 단계별 처리

1. **Application Layer Handler**: 외부 요청을 받아 Aggregate 목록을 확인하고 ServiceCommand로 구성해 DomainService에 전달
2. **Aggregate.DomainService**: ServiceCommand를 입력받아 여러 Aggregate를 일괄 갱신하고 DomainService 이벤트 축적
3. **DomainService Store**: 요청 범위에서 사용된 DomainService를 등록해 UoW가 나중에 이벤트를 수거·발행 가능하도록 함
4. **Unit of Work**: 이벤트 수거·발행 → DB 커밋 → 메시지 송신 순서로 처리

### After 코드 예시

```python
class Manager(AggregateRoot):
    domain_service = ManagerDomainService()

class UpdateTextsCommand(ServiceCommand):
    class Item:
        def __init__(self, aggregate: "Manager", text: str):
            self.aggregate = aggregate
            self.text = text
    def __init__(self, items: list["UpdateTextsCommand.Item"]):
        self.items = items

class ManagerDomainService(DomainServiceBase):
    def update_texts(self, cmd: UpdateTextsCommand) -> None:
        for item in cmd.items:
            item.aggregate.update_text(UpdateTextCommand(text=item.text))
        self.add_domain_service_event(
            AggregatesUpdatedEvent(aggregates=[i.aggregate for i in cmd.items])
        )

# Handler (After)
items: list[UpdateTextsCommand.Item] = []
for key, payload in incoming.items():
    agg = await repository.get_by_key_async(key)
    if not agg:
        continue
    items.append(UpdateTextsCommand.Item(
        aggregate=agg,
        text=build_text_from(payload, agg)
    ))

Manager.domain_service.update_texts(UpdateTextsCommand(items=items))
domain_service_store.prepare_domain_service(Manager)
await unit_of_work.save_changes_async()
```

---

## 패턴 핵심 요소

1. **ServiceCommand**: 작업의 의도와 범위를 명확히 담음
2. **Aggregate-Scoped DomainService**: Command를 입력받아 여러 Aggregate를 일관성 있게 갱신
3. **Unit of Work & DomainService Store**: DomainService를 등록하면 UoW가 이벤트를 수거·발행

---

## Unit of Work 동작 순서

`save_changes_async()` 실행 시:

1. 이벤트 수거·발행 (엔티티와 도메인서비스)
2. 메시지 점검 (inspect)
3. DB 커밋
4. 보조 저장소 커밋
5. 메시지 송신 (커밋 이후)

---

## 개선 효과

| 측면 | 개선 내용 |
|------|---------|
| **간결성** | Application Layer는 매핑과 위임만 남아 가독성 향상 |
| **의도 노출** | "무엇을 하려는가"는 커맨드에서, "어디서/범위"는 DomainService에서 드러남 |
| **테스트 응집성** | DomainService 단위 테스트만으로 규칙 검증 가능 |
| **UoW 일원화** | 이벤트 수거·발행 책임을 UoW로 집중시켜 타이밍 통제 |

---

## 운영 원칙

1. Aggregate는 클래스 변수로 DomainService 인스턴스를 가짐
2. DomainService는 전역 상태를 갖지 않음
3. 이벤트는 DomainService 레벨에서 한 번만 축적
4. Application Layer Handler는 `save_changes_async()` 호출만 수행

---

## 결론

ServiceCommand → DomainService → UoW 흐름으로 변경하여 코드가 의도 중심으로 정리됨. 이벤트 부착 위치를 DomainService로 통일함으로써 작업의 범위가 명확해졌으며, 이는 협업과 유지보수성 향상에 기여합니다.
