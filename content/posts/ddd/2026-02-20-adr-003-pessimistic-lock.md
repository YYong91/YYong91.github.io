---
title: "[ADR-003] 크레딧 차감에 Pessimistic Lock을 선택한 이유"
date: 2026-02-20
categories: ["ddd"]
tags: ["DDD", "동시성", "Lock", "ADR", "PostgreSQL", "아키텍처"]
series: ["Billing 마이크로서비스 ADR"]
series_order: 3
---

> **이 시리즈**: Billing 마이크로서비스 설계 결정 기록. [ADR-001](/posts/ddd/2026-02-20-adr-001-bounded-context/) · [ADR-002](/posts/ddd/2026-02-20-adr-002-customer-aggregate/)에서 이어집니다.

---

## 상황 (Context)

크레딧 차감 로직을 구현할 때 동시성 문제를 고려해야 했습니다.

시나리오: 사용자가 두 개의 탭에서 동시에 크레딧을 사용하는 요청을 보냅니다. 잔액이 100이고 각각 80을 차감하려 한다면, 둘 다 잔액을 100으로 읽고 차감을 시도할 수 있습니다.

```
T1: 잔액 읽기(100) → 차감(-80) → 저장(20)
T2: 잔액 읽기(100) → 차감(-80) → 저장(20)  ← 잔액 초과 차감 발생
```

이 문제를 어떻게 해결할지가 설계 결정 포인트였습니다.

---

## 문제 (Problem)

선택지는 크게 세 가지였습니다:

| 방법 | 원리 | 특징 |
|------|------|------|
| **Optimistic Lock** | 버전 번호로 충돌 감지, 실패 시 재시도 | 충돌이 드물 때 유리 |
| **Pessimistic Lock** | 읽을 때 바로 잠금, 트랜잭션 완료까지 대기 | 충돌이 빈번할 때 유리 |
| **Application-level Lock** | Redis 등으로 분산 락 구현 | 인프라 추가 필요 |

어떤 방법을 선택할지는 **크레딧 사용 패턴**에 달려 있었습니다.

---

## 결정 (Decision)

**`SELECT ... FOR UPDATE`를 사용한 Pessimistic Lock을 적용합니다.**

```python
# CreditAccountRepository
async def get_for_update(self, account_id: CreditAccountId) -> CreditAccount:
    result = await self.session.execute(
        select(CreditAccountModel)
        .where(CreditAccountModel.id == account_id.value)
        .with_for_update()  # Pessimistic Lock
    )
    return self._to_domain(result.scalar_one())
```

차감 요청이 들어오면:
1. `FOR UPDATE`로 해당 계좌 행을 잠금
2. 잔액 확인 후 차감
3. 트랜잭션 커밋 → 잠금 해제
4. 대기 중인 다음 요청이 잠금 획득 후 처리

---

## 이유 (Rationale)

**왜 Optimistic Lock이 아닌가:**

Optimistic Lock은 충돌이 적을 때 효율적입니다. 충돌이 나면 버전 불일치 예외가 발생하고 재시도합니다. 크레딧 차감은 다릅니다.

- 같은 계좌에 동시 요청이 몰릴 수 있음 (여러 탭, 자동화 스크립트)
- 재시도 로직을 Application Layer에 직접 구현해야 함
- 재시도가 반복되면 오히려 DB 부하가 커짐
- 충돌 시 사용자에게 오류가 노출될 가능성

크레딧 차감처럼 **정합성이 최우선이고 충돌 가능성이 있는 경우**, Pessimistic Lock이 더 단순하고 안전합니다.

**왜 Application-level Lock이 아닌가:**

Redis 분산 락은 인프라 복잡도를 높입니다. Redis 장애 시 크레딧 차감 전체가 영향을 받습니다. DB 트랜잭션으로 해결할 수 있는 문제에 외부 시스템을 추가하는 건 과도한 설계입니다.

**Pessimistic Lock의 트레이드오프:**

단점도 있습니다. 동시 요청이 많으면 대기 시간이 길어집니다. 하지만 크레딧 차감은 빠른 트랜잭션(잔액 확인 + 업데이트)이라 잠금 유지 시간이 짧습니다. 실제 운영에서 문제가 된 사례는 없었습니다.

---

## 결과 (Outcome)

Pessimistic Lock 적용 후 동시 차감 시 잔액 초과 문제가 발생하지 않았습니다.

**구현 시 주의한 점:**

- 잠금 범위를 최소화: `FOR UPDATE`는 해당 행 하나만
- 트랜잭션을 짧게 유지: 잠금 획득 → 차감 → 커밋, 이 사이에 외부 API 호출 없음
- 타임아웃 설정: 무한 대기를 방지하기 위해 DB 레벨 lock timeout 설정

```
"잠금을 쥐고 있는 시간이 길수록 병목이 된다"
— 트랜잭션 안에서 외부 IO가 없도록 설계한 이유
```
