---
title: "PaperMod 커스텀 테마 쓸 때 nav와 body 사이에 줄이 생기는 이유"
date: 2026-02-19
categories: ["til"]
tags: ["hugo", "papermod", "css", "dark-mode"]
---

다크 모드로 전환하면 nav 하단에 이상한 경계선이 생겼어요. 뭔가 배경색이 두 겹인 것처럼 보이는 그 선.

처음엔 `header`에 `border-bottom`이라도 붙어있나 싶어서 DevTools로 한참 뒤졌는데 아무것도 없었어요. `background: transparent`도 이미 걸려 있었고요.

## 범인은 `--code-bg`

PaperMod 소스를 직접 까봤더니 `body.list`가 배경색으로 `--theme`이 아니라 `--code-bg`를 쓰고 있었어요.

```css
/* PaperMod 내부 (themes/PaperMod/assets/css/core/theme-vars.css) */
body.list {
  background: var(--code-bg);
}
```

포스트 목록 페이지(`/posts/`, `/tags/` 등)가 전부 `body.list`예요. 여기서 `--code-bg`가 `--theme`이랑 다른 색으로 지정되어 있으면, nav는 `--theme` 배경, body는 `--code-bg` 배경이 되면서 경계가 눈에 보이는 거예요.

## 내가 뭘 놓쳤냐면

커스텀 CSS에서 `--theme`은 `#231e19`로 맞게 지정했는데, `--code-bg`는 깜빡하고 그냥 PaperMod 기본값인 `#161b22` 계열이 살아있었어요. 색이 달라서 경계가 생긴 것.

수정은 단순했어요. `--code-bg`를 `--theme`과 같은 값으로 맞춰주면 끝.

```css
/* assets/css/extended/custom.css */
:root {
  --theme: #faf6f1;
  --code-bg: #faf6f1; /* body.list 배경 → --theme과 동일하게 */
}

.dark,
:root[data-theme="dark"] {
  --theme: #231e19;
  --code-bg: #231e19; /* 여기도 맞춰줘야 함 */
}
```

`--code-block-bg`는 코드 블록 자체 배경이라 별도로 유지해도 돼요. `--code-bg`랑 다른 변수예요. 이름이 비슷해서 처음엔 같은 거인 줄 알았어요.

## 삽질 포인트

처음에 `--code-bg`를 라이트 모드만 고쳤는데 다크 모드에서 여전히 경계가 보였어요. `.dark` 블록에서도 따로 지정해줘야 한다는 걸 잊었던 거예요.

그리고 accent color인 `--craft-accent: #c96442`는 라이트/다크 모두 동일하게 유지했어요. 다크에서 색을 바꾸면 브랜드 일관성이 깨지더라고요. 다크 모드라고 억지로 다른 색 쓸 필요 없었어요.

## 핵심

서드파티 테마 변수명은 직관적이지 않아요. `--code-bg`가 코드 블록 배경인 줄 알았는데 실제로는 목록 페이지 `body` 배경이었어요. 테마 소스를 직접 grep해보는 게 DevTools 뒤지는 것보다 빠를 때가 있어요.
