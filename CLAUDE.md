# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

This project holds my private blog posts, published with Hugo. The individual posts live in `content/posts/`.

## Writing rules

- **File names must not contain spaces.** Use kebab-case (e.g. `my-new-post.md`); the file name becomes the URL.
- **Every post is a draft until I say otherwise.** Set `draft: true` in the front matter of every new post, and never remove it or set it to `false` unless I explicitly tell you to publish that post.
- **Cite references as linked footnotes.** Put a marker in the text (`...as shown in the paper.[^1]`) and a definition at the end of the post that links to the source:

  ```markdown
  [^1]: [Title of the source](https://example.com/source), Author or publisher.
  ```

## Site

A personal Hugo blog deployed to `https://fatih.macht.sachen.online/` via Codeberg Pages. Design constraints are deliberate: **no JavaScript, no webfonts, no trackers**. Serif for prose, monospace for "machine facts" (dates, tags, code), dark mode via `prefers-color-scheme` only. Keep changes within these constraints.

## Commands

Requires Hugo **extended, ≥ 0.146** (templates use the new layout system: `layouts/_partials/`, `home.html`, `page.html`, `section.html` etc. — not `layouts/_default/`). CI pins `0.154.5` in `.forgejo/workflows/deploy.yaml`; keep local and CI versions in sync.

```sh
hugo server -D          # local preview at http://localhost:1313, includes drafts
hugo --minify           # production build into public/ (gitignored)
hugo new posts/slug.md  # new post from archetypes/posts.md (starts as draft: true)
hugo --buildFuture      # include future-dated posts (otherwise silently skipped)
hugo list drafts        # which posts won't ship (also: hugo list future)
```

There are no tests or linters; `hugo --minify` succeeding without warnings is the check.

## Content conventions

- Posts live in `content/posts/` and need YAML front matter (`title`, `date`, `tags`, optional `draft`). Files without front matter, or with `draft: true` / a future `date`, won't appear properly on the live site. Most posts are currently `draft: true` (see Writing rules above).
- Optional `description` / `summary` front matter feed the `<meta name="description">` and `og:description` in `head.html` (falling back to `params.description`). `categories` appears in some posts but is **not** a configured taxonomy (only `tags` is), so it has no effect.
- The home page (`layouts/home.html`) lists all pages in the `posts` section, grouped by year via `layouts/_partials/post-list.html`; the same partial is reused for section/term listings.
- Non-post pages (`about.md`, `open-books.md`, `my-apps/index.md`) also render through `page.html`; the date, reading time and newer/older nav only show when `.Section` is `posts`. Each has a menu entry in `hugo.toml` whose `pageRef` must match the content path.
- Post-specific images: use a page bundle (`content/posts/slug/index.md` + image, as `content/my-apps/` does with its screenshots). Shared images go in `static/`. Exception: images that templates process (e.g. the about page's `avatar` front matter param, which `page.html` loads via `resources.Get` and crops with `.Fill`) must live in `assets/`, not `static/`.

## Architecture

- `hugo.toml` — site identity, `params` (author, description, intro line, linkedin), menu, and `markup.highlight.noClasses = false` so Chroma emits CSS classes.
- `layouts/baseof.html` wraps every page with `_partials/head.html`, `site-header.html`, `site-footer.html`.
- CSS: `head.html` concatenates `assets/css/main.css` + `assets/css/syntax.css` into one minified, fingerprinted bundle. All colors/fonts/spacing are tokens in the `:root` block at the top of `main.css`; `syntax.css` holds the light/dark code-highlight classes. Add new CSS to these files rather than inline or new `<link>`s.

## Deployment

Every push to `main` triggers `.forgejo/workflows/deploy.yaml` (runner `codeberg-tiny`): checkout → install Hugo → `hugo --minify` → `git-pages/action` uploads `public/` to `codeberg.page` using the auto-injected `forge.token`. No `pages` branch exists. `baseURL` in `hugo.toml` and `site:` in the workflow must both match the real URL (trailing slash included) or CSS and links break. The custom domain works because the workflow sets `server: 'codeberg.page'` and DNS has a CNAME plus a `_git-pages-forge-allowlist` TXT record (see README); switching to a plain `*.codeberg.page` URL means changing `baseURL`, `site:`, and removing `server:`.
