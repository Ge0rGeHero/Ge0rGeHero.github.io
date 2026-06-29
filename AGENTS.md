# AGENTS.md — Mizuki

## Project identity

- Astro 6 static blog template, forked from Fuwari
- `packageManager`: **pnpm 11.1.3** (enforced via `preinstall` script, `.npmrc`)
- Node >= 22 required; TypeScript 6.x in strict mode
- Output: `dist/` (static, deployed to Vercel / GH Pages)

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
| `lint.yml` | push/PR to `master` | biome check, astro check, build (with `ENABLE_CONTENT_SYNC=false`) |
| `deploy.yml` | push to `main` | build + deploy to `pages` branch via JamesIves action |

**Note**: lint workflow targets `master` branch, deploy workflow targets `main` — inconsistent branch names.

## Key docs (authoritative)

- `docs/rule/01-component-architecture.md` — Atomic Design layers
- `docs/rule/02-component-split-guide.md` — when/how to split components
- `docs/rule/03-file-organization-architecture.md` — directory structure rules
- `docs/rule/04-css-style-guide.md` — CSS conventions, `!important` policy
- `docs/rule/05-atom-component-usage.md` — atom component patterns
- `docs/rule/06-sidebar-widget-dev.md` — sidebar widget integration checklist
- `docs/rule/07-icon-usage-specification.md` — 3 icon systems, which to use where
