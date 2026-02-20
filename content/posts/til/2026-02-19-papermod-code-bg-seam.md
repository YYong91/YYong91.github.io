---
title: "PaperMod 커스텀 테마에서 nav와 body 사이에 줄이 생기는 이유"
date: 2026-02-19
categories: ["til"]
tags: ["hugo", "papermod", "css", "dark-mode"]
project: "YYong91.github.io"
source_sessions: ["2026-02-19-YYong91-github-io-main"]
---

다크 모드로 전환하면 nav 하단에 이상한 경계선이 생겼습니다. 배경색이 두 겹인 것처럼 보이는 그 선입니다.

처음에는 `header`에 `border-bottom`이 붙어있나 싶어서 DevTools로 한참 뒤졌는데 아무것도 없었습니다. `background: transparent`도 이미 적용되어 있었습니다.

## 범인은 `--code-bg`

PaperMod 소스를 직접 확인해보니 `body.list`가 배경색으로 `--theme`이 아니라 `--code-bg`를 사용하고 있었습니다.

```css
/* themes/PaperMod/assets/css/core/theme-vars.css */
.list {
  background: var(--code-bg);
}
```

포스트 목록 페이지(`/posts/`, `/tags/` 등)가 전부 `body.list`입니다. `--code-bg`가 `--theme`과 다른 색으로 지정되면, nav는 `--theme` 배경, body는 `--code-bg` 배경이 되면서 경계가 눈에 보이게 됩니다.

## 놓친 부분

커스텀 CSS에서 `--theme`은 `#231e19`로 맞게 지정했는데, `--code-bg`는 PaperMod 기본값이 그대로 남아 있었습니다. 색이 달라서 경계가 생긴 것입니다.

수정은 단순했습니다. `--code-bg`를 `--theme`과 같은 값으로 맞추면 됩니다.

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

참고로 `--code-block-bg`는 코드 블록 자체의 배경이라 별도로 유지해도 됩니다. `--code-bg`와 다른 변수입니다. 이름이 비슷해서 처음에는 같은 것인 줄 알았습니다.

## 삽질 포인트

처음에 `--code-bg`를 라이트 모드만 고쳤는데, 다크 모드에서 여전히 경계가 보였습니다. `.dark` 블록에서도 따로 지정해야 한다는 것을 잊었기 때문입니다.

그리고 accent color인 `--craft-accent: #c96442`는 라이트/다크 모두 동일하게 유지했습니다. 다크 모드에서 색을 바꾸면 브랜드 일관성이 깨집니다. 다크 모드라고 억지로 다른 색을 사용할 필요는 없었습니다.

## 핵심

서드파티 테마의 변수명은 직관적이지 않을 수 있습니다. `--code-bg`가 코드 블록 배경인 줄 알았는데 실제로는 목록 페이지 `body` 배경이었습니다. 테마 소스를 직접 grep해보는 것이 DevTools를 뒤지는 것보다 빠를 때가 있습니다.
