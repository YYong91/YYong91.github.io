---
title: "Infisical을 GitHub Actions CI에 연동하기 — curl이 Python urllib보다 나은 이유"
date: 2026-02-20
categories: ["infra"]
tags: ["infisical", "github-actions", "ci", "secrets-management", "cloudflare", "stripe"]
project: "core_backend"
source_sessions: ["2026-02-20-core_backend-TD-425"]
---

팀에서 만드는 제품이 클라우드뿐 아니라 온프레미스 환경에도 납품됩니다. 고객사 인프라에 직접 설치해 운용해야 하는 구조라서, 시크릿 관리 도구 선택지가 처음부터 좁았습니다.

AWS Secrets Manager는 AWS에 묶이고, Doppler는 클라우드 전용입니다. HashiCorp Vault는 셀프호스트가 가능하지만 운영 부담이 만만치 않습니다. Infisical은 오픈소스이고 셀프호스트를 공식 지원합니다. 회사에 남는 PC가 생기면서 거기에 Infisical을 올리는 것으로 결론이 났습니다.

## 🤔 `.env` 파일과 뭐가 다른가

| | `.env` 파일 | Infisical |
|--|------------|-----------|
| 저장 위치 | 로컬 파일 (평문) | 서버 (암호화) |
| 팀 공유 | Slack/이메일 수동 전달 | 프로젝트 멤버 자동 공유 |
| 환경 분리 | 파일을 여러 개 유지 | dev / staging / prod 내장 |
| 변경 이력 | 없음 | 감사 로그 기록 |
| CI 연동 | 값마다 GitHub Secrets 수동 등록 | Machine Identity로 자동화 |

팀원 온보딩이 `infisical login` 한 줄로 끝납니다. `.env.local` 파일을 Slack으로 주고받던 일이 없어졌습니다.

## 🛠️ 로컬 개발 셋업

CLI를 설치하고 로그인합니다.

```bash
brew install infisical/get-cli/infisical
infisical login --domain="https://infisical.example.com/api"
```

브라우저가 열리고 회사 계정으로 로그인하면 끝입니다. 프로젝트 루트에서 `infisical init`을 실행하면 `.infisical.json`이 생성됩니다. 이 파일에는 프로젝트 ID만 들어 있고 시크릿은 없어서 커밋해도 됩니다.

시크릿은 폴더 단위로 구조화했습니다.

```
/ (루트)       ← AWS, Cognito, DB 공통 변수
├── /account   ← Account 서비스 전용
├── /billing   ← Billing 서비스 전용 (Stripe 등)
├── /community ← Community 서비스 전용
└── /worker    ← Temporal Worker 전용
```

서비스별 폴더는 루트를 Secret Import로 상속받습니다. 폴더 하나만 지정하면 공통 + 전용 변수를 모두 주입받습니다.

사용법은 기존 명령어 앞에 `infisical run`을 붙이면 됩니다.

```bash
infisical run --env=dev --path=/billing -- \
  uv run uvicorn app.main:app --reload --port 8004
```

## 🔧 GitHub Actions CI 연동

로컬 셋업은 금방 됐는데, CI가 문제였습니다.

GitHub Actions Runner에서는 브라우저 로그인을 할 수 없습니다. Infisical은 이를 위해 **Machine Identity**를 제공합니다. 사람 계정 대신 CI 전용 자격증명을 별도로 발급받는 방식입니다.

Infisical 웹 UI에서 Organization → Access Control → Machine Identities로 이동해 Machine Identity를 생성합니다.

{{< figure src="machine-identity-list.png" caption="Organization Access Control → Machine Identities" >}}

생성 다이얼로그에서 이름을 지정하면 Client ID와 Client Secret이 발급됩니다.

{{< figure src="machine-identity-create.png" caption="Machine Identity 생성 다이얼로그" >}}

발급받은 값을 GitHub Secrets에 등록합니다.

- `INFISICAL_MACHINE_CLIENT_ID`
- `INFISICAL_MACHINE_CLIENT_SECRET`

여기까지는 순조로웠습니다.

### Cloudflare가 막아버렸습니다

처음에는 Python으로 Infisical API를 직접 호출했습니다. Runner에서 실행하면 응답이 오긴 오는데, JSON이 아니라 HTML이 돌아왔습니다. 셀프호스트 서버가 Cloudflare 뒤에 있었고, **Bot Fight Mode**가 Python `urllib`의 User-Agent를 봇으로 판단해 차단한 것입니다.

`infisical` CLI를 Runner에 직접 설치하는 것도 시도했지만, self-hosted Runner 환경에서 Homebrew를 보장하기 어려웠습니다. 에러 로그를 보면서 고치기를 반복하다 — 커밋 5개 만에 — `curl`로 해결했습니다.

`curl`은 User-Agent가 달라 Cloudflare를 통과합니다.

```bash
LOGIN_RESP=$(curl -s -w "\n%{http_code}" \
  -X POST "$BASE/api/v1/auth/universal-auth/login" \
  -H "Content-Type: application/json" \
  -d "{\"clientId\": \"$INFISICAL_CLIENT_ID\", \"clientSecret\": \"$INFISICAL_CLIENT_SECRET\"}")

LOGIN_HTTP=$(echo "$LOGIN_RESP" | tail -1)
LOGIN_BODY=$(echo "$LOGIN_RESP" | head -1)

# HTTP 200이 아니면 즉시 실패 처리
if [ "$LOGIN_HTTP" != "200" ]; then
  echo "Login failed: $LOGIN_BODY"
  exit 1
fi

TOKEN=$(echo "$LOGIN_BODY" | python3 -c \
  'import sys,json;print(json.load(sys.stdin)["accessToken"])')
```

발급받은 Access Token으로 시크릿을 조회해서 `GITHUB_ENV` 파일에 기록하면, 이후 Step에서 환경변수로 자동으로 사용할 수 있습니다.

```bash
curl -s "$BASE/api/v3/secrets/raw?workspaceId=$PROJECT_ID&environment=dev&secretPath=/billing" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c '
import sys, json, os
secrets = json.load(sys.stdin)["secrets"]
with open(os.environ["GITHUB_ENV"], "a") as f:
    for s in secrets:
        f.write(s["secretKey"] + "=" + s["secretValue"] + "\n")
print(f"Loaded {len(secrets)} secrets")
'
```

### Blocking / Non-blocking Job 분리

Billing 서비스 CI는 두 Job으로 나눴습니다.

| Job | 테스트 종류 | PR 차단 여부 |
|-----|-----------|------------|
| `tests` | 단위 + 통합 (DB 직접 연결) | ✅ 차단 |
| `stripe-integration` | 실제 Stripe API 호출 | ❌ 경고만 |

Stripe 통합 테스트는 외부 API에 의존하기 때문에 네트워크 상황이나 Stripe 서비스 상태에 따라 간헐적으로 실패할 수 있습니다. 이런 Job을 PR 차단으로 걸면 코드 문제가 없는데도 머지가 막힙니다.

```yaml
stripe-integration:
  name: Stripe Integration Tests (non-blocking)
  continue-on-error: true  # 실패해도 PR merge 차단 안 함
```

`continue-on-error: true`를 붙이면 실패 시 GitHub PR에 노란색 경고로 표시됩니다. 이상 신호는 주되, 머지는 막지 않는 구조입니다.

{{< figure src="github-actions-result.png" caption="단위 테스트 ✅ / Stripe 통합 테스트 ❌ (non-blocking)" >}}

## 📦 마무리

최종적으로 `billing-tests.yml`은 두 Job 구조로 완성됐습니다. Infisical 덕분에 Stripe API Key 같은 민감한 값을 GitHub Secrets에 직접 등록할 필요가 없어졌습니다. Infisical에서 키를 교체하면 다음 CI 실행 때 자동으로 반영됩니다.

셀프호스트한 Infisical 서버가 Cloudflare 뒤에 있다면, CI에서 Python `urllib` 대신 `curl`을 쓰세요. User-Agent 하나 차이입니다.
