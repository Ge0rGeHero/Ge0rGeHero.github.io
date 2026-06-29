# AGENTS.md — Mizuki

## Project identity

- Astro 6 static blog template, forked from Fuwari
- `packageManager`: **pnpm 11.1.3** (enforced via `preinstall` script, `.npmrc`)
- Node >= 22 required; TypeScript 6.x in strict mode
- Output: `dist/` (static, deployed to GitHub Pages)

## Commands

```bash
pnpm install           # must use pnpm (bun/npm/yarn are blocked)
pnpm dev               # dev server at localhost:3000 (not 4321 — README is wrong)
pnpm build             # update-anime → astro build → pagefind → compress-fonts
pnpm check             # astro check (Astro's own type/lint check)
pnpm type-check        # tsc --noEmit (full TS type check, wider than astro check)
pnpm format            # biome format --write ./src (src only, not scripts/tests)
pnpm lint              # biome check --write ./src (lints AND auto-fixes)
pnpm new-post <name>   # scaffold a new post
pnpm sync-content      # pull content from separate repo (needs .env)
```

**Order when changing code**: `pnpm format && pnpm lint && pnpm check && pnpm type-check`
- `check` (astro check) is faster than `type-check`; run both before PRs

## Architecture

### Directory map

| Dir | Purpose |
|---|---|
| `src/config/` | All site configuration (18 files, re-exported via `index.ts`) |
| `src/config/index.ts` | Single import point — `import { siteConfig, profileConfig, … } from "@/config"` |
| `src/content/` | Astro content collections (`posts/`, `spec/`) — content lives here |
| `src/pages/` | Route pages (includes `[...permalink].astro` for custom URLs) |
| `src/layouts/Layout.astro` | Root layout: banner, theme variables, Swup container, global CSS |
| `src/components/` | Atomic Design: `atoms/`, `misc/`, `organisms/`, `widgets/`, `features/`, `control/`, `common/`, `layout/` |
| `src/components/index.ts` | Barrel export for all components |
| `src/plugins/` | Custom remark/rehype plugins (admonitions, mermaid, github-cards, etc.) |
| `src/styles/` | CSS files: Tailwind v4 main.css + 24 feature stylesheets; includes `.styl` files (Stylus) |
| `src/scripts/` | Client-side TS (Swup manager, theme optimizer, panel handlers) |
| `src/i18n/` | Multi-language translation system (ja, zh-CN, zh-TW, en) |
| `src/utils/` | Shared utilities (crypto, permalinks, URL helpers, TOC manager) |
| `src/stores/` | Svelte writable stores (music player state) |
| `src/types/` | TypeScript interfaces (`config.ts`, `album.ts`, `framework-components.d.ts`) |
| `scripts/` | Build support scripts (content sync, anime/bangumi/bilibili updaters, font compression) |
| `tests/` | Standalone test scripts (no framework — run with `node tests/<name>.mjs`) |
| `docs/rule/` | 7 specification docs — **authoritative** for component architecture, CSS, icons, widgets |

### Technology layers

- **Framework**: Astro 6 (static output, `.astro` pages) with Svelte 5 for interactive islands
- **CSS**: Tailwind CSS v4 (`@import "tailwindcss"` + `@plugin "@tailwindcss/typography"`) with PostCSS (`postcss-import`, `postcss-nesting`); also uses Stylus for markdown styles
- **Page transitions**: Swup (container-based, selected containers: `main`)
- **Code blocks**: Expressive Code with custom plugins (language badge, copy button, collapsible sections, line numbers)
- **Search**: Pagefind (generated post-build; `pagefind.yml` excludes katex spans)
- **Markdown**: Astro MDX + custom remark/rehype pipeline (math, admonitions, mermaid, github cards, external links)
- **Icons**: 3 distinct icon systems — see `docs/rule/07-icon-usage-specification.md`
  - `.astro` files → `import { Icon } from "astro-icon/components"` (prop: `name`)
  - `.svelte` files → `import Icon from "@iconify/svelte"` (prop: `icon`)
  - Never use raw `<iconify-icon>` elements

### Import path aliases (tsconfig.json)

```ts
@components/*  → ./src/components/*
@assets/*      → ./src/assets/*
@constants/*   → ./src/constants/*
@utils/*       → ./src/utils/*
@i18n/*        → ./src/i18n/*
@layouts/*     → ./src/layouts/*
@/*            → ./src/*
src/*          → ./src/*
```

### Content model (`src/content.config.ts`)

Two collections: `posts` (glob: `**/*.{md,mdx}`) and `spec` (special pages). Frontmatter schema includes: `title`, `published`, `draft`, `pinned`, `tags`, `category`, `comment`, `lang`, `encrypted`/`password`/`passwordHint`, `permalink`, `alias`.

## Style & conventions

- **Formatter**: Biome (tab indentation, double quotes, semicolons, trailing commas, arrow parens)
- **VSCode settings recommend Prettier but the project actually uses Biome** — do not change the formatter; follow `biome.json`
- Biomes `useConst` and `useImportType` rules are **off** for `.svelte`, `.astro`, `.vue` files
- `biome format --write ./src` only covers `src/` — scripts and tests are excluded from formatting
- CSS: avoid `!important` except in `src/styles/twikoo.css` (third-party overrides)
- New sidebar widgets require 3 steps: add to `WidgetComponentType` in types, configure in `sidebarLayoutConfig`, implement the component (see `docs/rule/06-sidebar-widget-dev.md`)
- Component architecture follows Atomic Design: atoms → molecules → organisms → pages (see `docs/rule/01-component-architecture.md`)

## Environment & secrets

- `.env` file (not committed) loaded by `scripts/load-env.js` (no dotenv package)
- Key env vars: `ENABLE_CONTENT_SYNC`, `CONTENT_REPO_URL`, `INDEXNOW_KEY`, `INDEXNOW_HOST`, `BILI_SESSDATA`
- Content separation: if `ENABLE_CONTENT_SYNC=true`, `src/content/` is symlinked from a separate git repo
- The `content/` dir at root and `src/data/bangumi-data.json` / `src/data/bilibili-data.json` are gitignored (generated)

## Build quirks

- `pnpm build` runs `scripts/update-anime.mjs` first — anime data is fetched before the Astro build
- `predev` and `prebuild` hooks run `sync-content.js || true` — **failures are silently ignored**
- Production build drops `console` and `debugger` via esbuild options
- The `buildCommand` in `vercel.json` is `pnpm build` (implicitly includes update-anime)
- Output is `dist/`, deployed as static site

## Testing

- No test framework installed (no vitest, jest, mocha)
- `tests/crypto.test.mjs` — run with `node tests/crypto.test.mjs`
- `tests/post-card-content.test.ts` — for tsx/ts-node runner
- Test files were removed from config (README notes "Test File Cleanup")

## CI (`.github/workflows/`)

| Workflow | Triggers | What it runs |
|---|---|---|
| `deploy.yml` | push to `main`, `workflow_dispatch` | build via `withastro/action@v3` → deploy via `actions/deploy-pages@v4` |

## GitHub Pages 部署指南

### 仓库要求

- 仓库名必须是 **`<username>.github.io`**（用户站点），否则需在 `astro.config.mjs` 中设置 `base` 路径

### 配置文件修改

部署到 GitHub Pages 前需要修改以下文件：

1. **`src/config/siteConfig.ts`** — 修改 `siteURL`：
   ```ts
   siteURL: "https://<username>.github.io/",
   ```
2. **`astro.config.mjs`** — `base` 保持 `"/"`（用户站点），项目站点需改为 `"/<repo-name>/"`

3. **源代码中所有原作者的仓库名、域名引用** — 全局搜索替换：
   - 原仓库名 `LyraVoid/Mizuki` → 你的仓库名
   - 原域名 `mizuki.mysqil.com` → 你的域名
   - 涉及的配置文件：`profileConfig.ts`, `pioConfig.ts`, `navBarConfig.ts`, `projects.ts`, `friends.ts`, `Footer.astro`, `about.md` 等

### 部署文件

| 文件 | 作用 |
|------|------|
| `.github/workflows/deploy.yml` | Astro 官方 GitHub Pages 部署工作流 |
| `.nojekyll` | 空文件，阻止 GitHub 用 Jekyll 处理 `.astro` 文件 |

### `deploy.yml` 关键配置

```yaml
name: Deploy to GitHub Pages
on:
  push:
    branches: [ main ]
  workflow_dispatch:
permissions:
  contents: read
  pages: write
  id-token: write
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: withastro/action@v3
        with:
          node-version: 22          # pnpm 11 要求 Node >= 22
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/deploy-pages@v4
```

### 常见坑

| 问题 | 原因 | 解决 |
|------|------|------|
| pnpm 报 `node:sqlite` 缺失 | `withastro/action@v3` 默认 Node 20，pnpm 11 需要 Node >= 22 | 在 action 中显式设置 `node-version: 22` |
| Jekyll 报 YAML 解析错误 | GitHub 自动用 Jekyll 构建 `username.github.io` 仓库，但 `.astro` 文件不是 Jekyll 格式 | 根目录加 `.nojekyll` 空文件 |
| `package-manager` 版本冲突 | `withastro/action` 的 `package-manager` 参数与 `package.json` 的 `packageManager` 字段冲突 | 不设 `package-manager`，让 action 自动从 `package.json` 读取 |
| `env:` 块全是注释导致 YAML 无效 | GitHub Actions 不允许空的 `env:` 映射 | 删除空的 `env:` 块，或填入真实环境变量 |

### 部署步骤

1. 在 GitHub 新建仓库 `https://github.com/<username>/<username>.github.io`（空仓库）
2. 本地推送 `main` 分支到该仓库
3. GitHub 仓库 Settings → Pages → Source 选择 **"GitHub Actions"**
4. 每次推送 `main` 分支，Actions 自动构建并部署

## Key docs (authoritative)

- `docs/rule/01-component-architecture.md` — Atomic Design layers
- `docs/rule/02-component-split-guide.md` — when/how to split components
- `docs/rule/03-file-organization-architecture.md` — directory structure rules
- `docs/rule/04-css-style-guide.md` — CSS conventions, `!important` policy
- `docs/rule/05-atom-component-usage.md` — atom component patterns
- `docs/rule/06-sidebar-widget-dev.md` — sidebar widget integration checklist
- `docs/rule/07-icon-usage-specification.md` — 3 icon systems, which to use where
