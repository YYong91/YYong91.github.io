---
title: "Cloudflare Bot Fight Mode는 Python urllib를 봇으로 본다"
date: 2026-02-20
categories: ["til"]
tags: ["github-actions", "cloudflare", "infisical", "curl", "ci"]
project: "core_backend"
source_sessions: ["2026-02-20-core_backend-TD-425"]
---

GitHub Actions에서 Infisical Machine Identity로 로그인하는 스텝을 Python으로 짰는데, 응답이 계속 JSON이 아닌 HTML이었습니다.

처음에는 엔드포인트 URL이 틀렸나 싶었습니다. 커밋을 5개 날린 뒤에야 원인을 찾았습니다.

## 🔍 원인: Bot Fight Mode

저희 Infisical은 self-hosted이고 Cloudflare 뒤에 있습니다. Cloudflare의 Bot Fight Mode가 활성화되어 있으면, Python `urllib`의 기본 User-Agent(`python-urllib/3.x`)를 봇으로 판단해서 요청을 차단합니다. 응답으로 JSON 대신 Cloudflare의 챌린지 HTML 페이지가 내려옵니다.

```python
# 이렇게 짰을 때 문제가 발생
import urllib.request
import json

req = urllib.request.Request(
    "https://infisical.example.com/api/v1/auth/universal-auth/login",
    data=json.dumps({"clientId": "...", "clientSecret": "..."}).encode(),
    headers={"Content-Type": "application/json"},
)
with urllib.request.urlopen(req) as resp:
    # resp.read()가 JSON이 아니라 Cloudflare HTML을 반환함
    data = json.loads(resp.read())
```

`User-Agent`를 변조하면 우회할 수도 있지만, 근본적으로 불안정한 방법입니다.

## ✔ curl로 교체

curl은 Cloudflare Bot Fight Mode를 통과합니다. GitHub Actions 러너에는 curl이 기본으로 설치되어 있어서 별도 설치 없이 바로 쓸 수 있습니다.

```yaml
- name: Infisical login
  run: |
    RESPONSE=$(curl -s -w "\n%{http_code}" -X POST \
      "https://infisical.example.com/api/v1/auth/universal-auth/login" \
      -H "Content-Type: application/json" \
      -d "{\"clientId\":\"$INFISICAL_CLIENT_ID\",\"clientSecret\":\"$INFISICAL_CLIENT_SECRET\"}")

    HTTP_STATUS=$(echo "$RESPONSE" | tail -n1)
    BODY=$(echo "$RESPONSE" | head -n -1)

    if [ "$HTTP_STATUS" != "200" ]; then
      echo "Login failed: HTTP $HTTP_STATUS"
      echo "$BODY"
      exit 1
    fi

    TOKEN=$(echo "$BODY" | python3 -c "import sys,json; print(json.load(sys.stdin)['accessToken'])")
    echo "INFISICAL_TOKEN=$TOKEN" >> $GITHUB_ENV
```

`-w "\n%{http_code}"`로 HTTP 상태 코드를 응답 바디 뒤에 붙여서 한 번에 받은 다음, 마지막 줄과 나머지를 분리하는 패턴입니다. 상태 코드를 먼저 체크하면 HTML이 돌아왔을 때 JSON 파싱 에러 전에 명확한 실패 메시지를 볼 수 있습니다.

## 📌 GITHUB_ENV 파일 주입 방식

같은 작업을 하면서 하나 더 알게 된 것이 있습니다. Step 사이에 값을 넘길 때 `$GITHUB_ENV` 파일을 씁니다.

```bash
echo "KEY=VALUE" >> $GITHUB_ENV
```

이 줄을 실행하면 **다음 Step부터** `KEY`가 환경변수로 주입됩니다. 같은 Step 안에서 `$KEY`를 바로 참조하면 빈 값이 나옵니다. `run` 블록 안에서 변수를 넘길 때는 그냥 shell 변수(`TOKEN=...`)를 쓰고, 다른 Step에 넘길 때만 `>> $GITHUB_ENV`를 사용합니다.

## 🏁 삽질 포인트

디버깅이 느렸던 이유는 CI 로그에서 HTML 응답을 그냥 `json.JSONDecodeError`로만 봤기 때문입니다. 에러 메시지만 봐서는 네트워크 문제인지, URL 문제인지, 응답 형식 문제인지 알 수 없었습니다.

HTTP 상태 코드를 먼저 체크하고, 실패 시 응답 바디 전체를 출력하는 구조로 짰더라면 한 번에 원인을 찾았을 것입니다. CI 스크립트에서 HTTP 응답은 항상 상태 코드 검증을 먼저 하는 것이 맞습니다.
