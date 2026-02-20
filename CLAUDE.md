# YYong91.github.io

Personal dev log & portfolio. Hugo + PaperMod + GitHub Actions auto-deploy.

## Quick Start

```bash
hugo server -D          # http://localhost:1313
hugo --gc --minify      # production build
```

## Site Structure

```
content/
├── posts/til/          # TIL posts
├── posts/claude-code/  # Claude Code experience
├── projects/           # Project portfolio
├── resume/             # Resume (data/resume.yaml)
└── search.md           # Search page
```

## Deploy

Push to `main` → GitHub Actions (`.github/workflows/hugo.yml`) → GitHub Pages.

PaperMod is a git submodule. After clone: `git submodule update --init --recursive`

## Theme: Warm Craft

Custom CSS at `assets/css/extended/custom.css`.
- Serif fonts: Noto Serif KR + Source Serif 4
- Palette: cream `#faf6f1` + terracotta `#c96442`
- `--code-bg` MUST equal `--theme` (PaperMod body.list uses --code-bg as background)
- Dark mode: same accent color, only darken backgrounds

## Writing Style (YYong's Voice)

**Reference**: 작성자의 velog 스타일 (ActiveMQ → RabbitMQ 전환기)

- **톤**: 입니다체 (formal polite). 전문적이되 접근 가능한 중간 톤
- **인칭**: 개인 작업 "~했습니다", 팀 작업 "저희 팀은 ~했습니다"
- **구조**: Problem → Cause → Solution. Before-After 대비. 도입 1-2문장으로 바로 시작
- **포매팅**: 비교는 표(table), 섹션 이모지(🚀, ✔, 🏁), 구체적 숫자/에러/파일명 필수
- **금지**: "이번 포스트에서는", "알아보겠습니다", "도움이 되셨길", "결론적으로", "또한/더불어/아울러", marketing tone

## dev-portfolio Plugin

Location: `~/.claude/plugins/local/dev-portfolio/`
GitHub: YYong91/dev-portfolio (private)
Config: `~/.claude/portfolio/config.json`
Raw logs: `~/.claude/portfolio/raw/*.json`

| Command | Purpose |
|---------|---------|
| `/portfolio setup` | One-time config |
| `/portfolio log` | Capture session as raw log |
| `/portfolio wrap` | /wrap + /portfolio log |
| `/portfolio generate` | Generate content → PR |
| `/portfolio status` | Raw log inventory |

## Key Config Notes

- `outputs.home` must include `JSON` (search depends on it)
- Hugo version: 0.156.0 (local and CI must match)
- Language: `ko`
