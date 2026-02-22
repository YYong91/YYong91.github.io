---
title: "JWT `iss` claim 없으면 서비스 간 토큰이 구분되지 않는다"
date: 2026-02-21
categories: ["til"]
tags: ["auth", "jwt", "security", "fastapi", "sso"]
project: "podonest"
source_sessions: ["2026-02-21-podonest-multiuser-sso"]
---

podo-auth, podo-bookshelf, podo-budget 세 서비스에 SSO를 붙이는 작업을 하다가 문제를 발견했습니다. 세 서비스가 동일한 `JWT_SECRET`을 공유하고 있었습니다.

## 🏗 문제 상황

공유 시크릿 자체는 의도된 설계였습니다. SSO니까 podo-auth가 발급한 토큰을 나머지 서비스들이 검증할 수 있어야 합니다. 그런데 `iss` (issuer) claim이 없으면 어떻게 될까요.

podo-bookshelf가 직접 발급한 토큰이 있다고 가정하면, 그 토큰을 podo-budget에 들고 가도 서명 검증을 통과합니다. 시크릿이 같으니까요. 서비스 간 토큰 경계가 사라지는 셈입니다.

## 🔐 원인: issuer claim 부재

JWT 스펙에는 `iss` claim이 있습니다. 토큰을 누가 발급했는지를 명시하는 필드입니다. 서명 검증과는 별개로, 이 값을 애플리케이션 레벨에서 확인해야 "이 토큰이 내가 신뢰하는 발급자로부터 왔는가"를 판단할 수 있습니다.

podo-bookshelf는 이미 이 패턴을 적용하고 있었습니다. podo-budget은 빠져 있었고, 이번 SSO 통합 과정에서 발견했습니다.

| 항목 | 서명 검증 (`JWT_SECRET`) | 발급자 검증 (`iss`) |
|---|---|---|
| 역할 | 토큰이 위변조되지 않았는가 | 토큰이 신뢰할 수 있는 서비스에서 왔는가 |
| 검증 시점 | `jwt.decode()` 내부 | decode 이후 애플리케이션 레벨 |
| 공유 시크릿 환경에서 필요성 | 필수 | 필수 |

## ✔ 적용한 해결책

발급 측인 podo-auth의 페이로드에 `"iss": "podo-auth"`를 추가하고, 검증 측 각 서비스에서 decode 후 issuer를 명시적으로 확인합니다.

```python
# podo-auth: 토큰 발급
payload = {
    "sub": str(user.id),
    "iss": "podo-auth",          # 발급자 명시
    "exp": datetime.now(UTC) + timedelta(hours=24),
}

# podo-budget, podo-bookshelf: 토큰 검증
payload = jwt.decode(token, settings.JWT_SECRET, algorithms=["HS256"])
if payload.get("iss") != "podo-auth":
    raise HTTPException(status_code=401, detail="Invalid token issuer")
```

`python-jose` 같은 라이브러리는 `jwt.decode()` 옵션으로 issuer 검증을 내장 지원하기도 합니다. 여기서는 명시적인 조건문으로 처리했습니다. 어느 쪽이든 핵심은 서명 검증과 issuer 검증을 둘 다 수행하는 것입니다.

시크릿을 공유하는 멀티 서비스 구조에서는 서명이 유효해도 발급자가 다를 수 있습니다. `iss` 검증은 선택이 아니라 필수입니다.
