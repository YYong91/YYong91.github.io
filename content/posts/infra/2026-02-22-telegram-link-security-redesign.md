---
title: "Telegram 연동에서 패스워드를 없애고 일회용 코드로 교체한 이유"
date: 2026-02-22
categories: ["infra"]
tags: ["security", "telegram-bot", "fastapi", "tdd", "authentication", "one-time-code"]
project: "podo-budget"
source_sessions: ["2026-02-22-podo-budget-feat-telegram-link-code"]
---

podo-budget의 Telegram 봇 연동 기능에 보안 문제가 있었습니다. `/link username password` 방식이었는데, 사용자가 Telegram 채팅창에 평문 패스워드를 직접 입력하는 구조였습니다.

## ⚠️ 기존 방식의 문제

Telegram은 기본적으로 E2E 암호화가 적용되지 않습니다. 서버 사이드 암호화만 되기 때문에 Telegram 서버에는 메시지 내용이 평문에 가까운 형태로 존재합니다. 즉, `/link myaccount mypassword123`을 입력하면:

- 패스워드가 채팅 기록에 영구적으로 남습니다
- Telegram 서버가 이 내용을 볼 수 있습니다
- 만료 시간도 없고, 한 번 쓰면 무효화되는 보장도 없습니다

이 문제를 해결하기 위해 일회용 코드(one-time code) 방식으로 전환했습니다.

## 🏗 새로운 연동 플로우

변경 후 흐름은 다음과 같습니다.

1. 사용자가 웹 앱 설정 페이지에서 **"코드 발급"** 버튼을 클릭합니다
2. 서버가 6자리 대문자 영숫자 코드(예: `A7K2NP`)를 생성하고 15분 만료 시간과 함께 DB에 저장합니다
3. 사용자가 Telegram 봇에서 `/link A7K2NP`를 입력합니다
4. 봇이 코드 유효성, 만료 여부, 중복 연동 여부를 검증합니다
5. 검증 통과 시 `telegram_link_code`를 즉시 NULL로 만들고 `telegram_chat_id`를 저장합니다

패스워드는 더 이상 어떤 채팅창에도 등장하지 않습니다.

## 🔐 설계 결정과 근거

각 선택에는 명확한 이유가 있었습니다.

| 결정 | 선택한 값/방식 | 근거 |
|------|--------------|------|
| 코드 길이 | 6자리 | 36^6 ≈ 21억 조합. 브루트포스에 충분한 엔트로피 |
| 문자셋 | 대문자 + 숫자 | 모바일 키보드에서 입력하기 쉬움. 소문자 혼용 시 오타 위험 |
| 만료 시간 | 15분 | 앱 전환에 충분한 시간, 노출 위험 최소화 |
| 사용 후 처리 | 즉시 NULL | 리플레이 어택 원천 차단 |
| 중복 체크 | 서비스 레이어 | 하나의 Telegram 계정이 여러 웹 계정에 연동되는 것을 방지 |
| 기존 방식 | 완전 제거 | 패스워드 방식 폴백을 남겨두면 보안 개선의 의미가 없음 |

만료 처리도 세심하게 다뤘습니다. 만료된 코드로 요청이 오면 그 시점에 즉시 DB에서 코드를 지웁니다. 오래된 만료 코드가 DB에 잔류하지 않습니다.

## 🧪 TDD로 먼저 엣지케이스를 발견한 과정

구현 전에 테스트를 먼저 작성했습니다. API 테스트 5개, 봇 서비스 테스트 3개로 총 8개를 먼저 만들고, 전부 실패하는 것을 확인한 뒤 구현을 시작했습니다.

테스트를 먼저 쓰다 보니 코딩 전에 케이스들이 눈에 보였습니다.

- 만료된 코드로 `/link`를 시도하면?
- 존재하지 않는 코드를 입력하면?
- 이미 다른 계정에 연동된 `telegram_chat_id`로 시도하면?

이 세 가지 실패 케이스를 구현 전에 미리 정의했기 때문에, 실제 코드를 작성할 때 자연스럽게 각 케이스에 맞는 분기와 에러 메시지를 넣게 됐습니다.

## ✔ 구현 핵심 코드

코드 생성은 Python 표준 라이브러리 `secrets`를 사용했습니다. `random` 대신 `secrets`를 쓴 이유는 암호학적으로 안전한 난수를 보장하기 때문입니다.

```python
# API 엔드포인트 - 코드 발급
@router.post("/telegram-link-code", response_model=TelegramLinkCodeResponse)
async def generate_telegram_link_code(current_user, db):
    # secrets.choice는 암호학적으로 안전한 난수 생성
    code = "".join(secrets.choice(string.ascii_uppercase + string.digits) for _ in range(6))
    expires_at = datetime.now(UTC) + timedelta(minutes=15)
    current_user.telegram_link_code = code
    current_user.telegram_link_code_expires_at = expires_at
    await db.commit()
    return TelegramLinkCodeResponse(code=code, expires_at=expires_at)
```

봇 핸들러 쪽에서는 검증 순서가 중요합니다. 코드 존재 여부를 먼저 확인하고, 그다음 만료를 확인합니다. 만료된 코드는 그 자리에서 삭제합니다.

```python
# 봇 서비스 - 코드 검증 및 연동
async def link_telegram_account_by_code(db, code, telegram_chat_id):
    now = datetime.now(UTC)
    result = await db.execute(select(User).where(User.telegram_link_code == code))
    user = result.scalar_one_or_none()

    if user is None:
        return False, "❌ 유효하지 않은 코드입니다"

    if user.telegram_link_code_expires_at < now:
        user.telegram_link_code = None  # 만료된 코드 즉시 정리
        await db.commit()
        return False, "⏰ 코드가 만료되었습니다. 설정 페이지에서 새 코드를 발급하세요."

    # 중복 연동 체크 후
    user.telegram_chat_id = telegram_chat_id
    user.telegram_link_code = None  # 사용 후 즉시 삭제 → 리플레이 방지
    await db.commit()
    return True, "✅ 연동 완료!"
```

## 🚀 변경 전후 비교

| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| 연동 명령어 | `/link username password` | `/link A7K2NP` |
| 패스워드 노출 | 채팅창에 평문으로 남음 | 없음 |
| 만료 | 없음 | 15분 |
| 재사용 | 가능 | 불가 (즉시 삭제) |
| 중복 연동 방지 | 없음 | 서비스 레이어에서 체크 |
| 테스트 수 | 370개 | 378개 (+8) |

PR #8로 머지됐고, 프론트엔드 설정 페이지에는 코드 표시, 복사 버튼, 만료 카운트다운, 연동 해제 버튼이 함께 들어갔습니다.

패스워드 방식의 폴백은 남기지 않았습니다. 보안 개선을 하면서 구 방식을 옵션으로 남겨두는 건 의미가 없습니다. 기존 핸들러는 완전히 제거했습니다.
