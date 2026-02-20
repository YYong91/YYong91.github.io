---
title: "PaperMod 커스텀 테마에서 nav와 body 사이에 줄이 생기는 이유"
date: 2026-02-19
categories: ["til"]
tags: ["hugo", "papermod", "css", "dark-mode"]
project: "YYong91.github.io"
source_sessions: ["2026-02-19-YYong91-github-io-main"]
---

다크 모드로 전환하면 nav 하단에 이상한 경계선이 생겼습니다. 배경색이 두 겹인 것처럼 보이는 그 선입니다.

처음에는 `header`에 `border-bottom`이 있나 싶어서 DevTools로 한참 뒤졌습니다. 아무것도 없었습니다. `background: transparent`도 이미 적용되어 있었습니다.

## 🔍 범인은 `--code-bg`

DevTools를 아무리 봐도 답이 안 나와서 PaperMod 소스를 직접 grep했습니다.

```css
/* themes/PaperMod/assets/css/core/theme-vars.css */
.list {
  background: var(--code-bg);
}
```

포스트 목록 페이지(`/posts/`, `/tags/` 등)가 전부 `body.list`입니다. `--code-bg`가 `--theme`과 다른 색으로 지정되면, nav는 `--theme` 배경이고 body는 `--code-bg` 배경이 되면서 경계가 눈에 보이게 됩니다.

변수명만 보면 코드 블록 배경색 같은데, 실제로는 목록 페이지 `body` 배경이었습니다.

## ✔ 수정

커스텀 CSS에서 `--theme`은 `#231e19`로 맞게 지정했는데, `--code-bg`는 PaperMod 기본값이 그대로 남아 있었습니다.

| 변수 | 변경 전 | 변경 후 |
|------|---------|---------|
| `--theme` (dark) | `#231e19` | `#231e19` (유지) |
| `--code-bg` (dark) | PaperMod 기본값 | `#231e19` (--theme과 동일) |

```css
/* assets/css/extended/custom.css */
:root {
  --theme: #faf6f1;
  --code-bg: #faf6f1; /* body.list 배경 → --theme과 동일하게 */
}

.dark,
:root[data-theme="dark"] {
  --theme: #231e19;
  --code-bg: #231e19; /* 여기도 맞춰야 합니다 */
}
```

참고로 `--code-block-bg`는 코드 블록 자체의 배경이라 별도로 유지해도 됩니다. 이름이 비슷해서 처음에는 같은 변수인 줄 알았습니다.

## 🏁 삽질 포인트

처음에 `--code-bg`를 라이트 모드만 고쳤습니다. 다크 모드에서 여전히 경계가 보여서 한참 헤맸는데, `.dark` 블록에서도 따로 지정해야 한다는 것을 잊은 것이었습니다.

서드파티 테마의 변수명은 직관적이지 않을 수 있습니다. `--code-bg`가 코드 배경인 줄 알았는데 실제로는 목록 페이지 body 배경이었습니다. DevTools를 뒤지는 것보다 테마 소스를 직접 grep하는 것이 빠를 때가 있습니다.
