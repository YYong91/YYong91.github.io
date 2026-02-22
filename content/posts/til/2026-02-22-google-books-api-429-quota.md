---
title: "Google Books API 키 없이 쓰면 공유 쿼터를 소진한다"
date: 2026-02-22
categories: ["til"]
tags: ["google-books-api", "fly.io", "api", "debugging", "podonest"]
project: "podonest"
source_sessions: ["2026-02-22-podo-design-unification"]
---

책 검색 API가 갑자기 500을 반환하기 시작했습니다. 코드는 건드리지 않았는데도요.

## 🔍 원인: API 키 없이 쓰면 익명 공유 쿼터를 씁니다

Google Books API는 키 없이도 호출이 됩니다. 문서에는 "무인증 요청도 허용"이라고 나와 있고, 개발 중엔 실제로 잘 동작합니다. 문제는 이 무인증 요청이 IP 기반 익명 쿼터를 사용한다는 점입니다.

Fly.io처럼 공유 아웃바운드 IP를 사용하는 플랫폼에서는 같은 IP 대역을 쓰는 다른 앱들이 익명 쿼터를 나눠 씁니다. 내 앱이 아무것도 안 해도 쿼터가 소진되고, 그 결과가 `429 Too Many Requests`입니다. 백엔드에서 이를 잡아 내리면 `500`으로 변환되니 로그만 봐서는 원인이 더 흐릿합니다.

| 인증 방식 | 쿼터 기준 | 공유 여부 |
|-----------|-----------|-----------|
| API 키 없음 | IP 주소 | 공유 아웃바운드 IP면 다른 앱과 공유 |
| API 키 있음 | API 키 (프로젝트) | 내 프로젝트만 |

## ✔ 해결: API 키 발급 후 Fly.io secrets에 등록

Google Cloud Console에서 Books API를 활성화하고 키를 발급받았습니다. 그런 다음 Fly.io secrets에 등록했습니다.

```bash
fly secrets set GOOGLE_BOOKS_API_KEY=AIza...
```

백엔드에서는 환경변수를 읽어 요청 URL에 붙이는 것이 전부입니다.

```python
# 기존: API 키 없이 호출
url = f"https://www.googleapis.com/books/v1/volumes?q={query}"

# 수정: API 키 추가
api_key = os.getenv("GOOGLE_BOOKS_API_KEY")
url = f"https://www.googleapis.com/books/v1/volumes?q={query}&key={api_key}"
```

배포 후 curl로 바로 검증했습니다.

```bash
curl "https://podonest.fly.dev/api/books/search?q=python"
# → 결과 15개 반환 확인
```

## 🏁 삽질 포인트

에러가 간헐적이었습니다. 처음 몇 번은 잘 되다가 갑자기 터지는 패턴이라 코드 문제라고 생각하기 쉽습니다. 실제로 쿼터 소진 때문이라는 걸 파악하는 데 시간이 걸렸습니다.

외부 API를 붙일 때는 "키 없이도 동작한다"는 사실이 함정입니다. 개발 환경에서 잘 됐다고 프로덕션에서도 된다는 보장이 없습니다. 공유 인프라(Fly.io, Railway, Render 등)를 쓴다면 API 키 발급은 처음부터 하는 것이 맞습니다.
