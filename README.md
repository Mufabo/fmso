# fatihs-ai-dumping-ground.duckdns.org — a personal Hugo blog

My personal ~~AI-slop dumping ground~~ blog.

Requires [Hugo](https://gohugo.io/) **extended edition, 0.146 or newer**
(the templates use Hugo's current layout system). Built and tested with
Hugo 0.154.5.

## 1. Run the blog locally

Install Hugo (extended) from https://gohugo.io/installation/ — it is a
single binary. Verify with `hugo version`.

Then, from the project root:

```sh
# Start a local preview server, including drafts (-D)
hugo server -D
```

Open http://localhost:1313. The page reloads automatically whenever you
save a file.

To produce the deployable site without serving it:

```sh
hugo --minify
```

The finished site lands in `public/` as plain HTML and CSS. That folder
is generated output — it is `.gitignore`d and safe to delete at any time.

## 2. Add a new blog post

Create a post with:

```sh
hugo new posts/my-first-post.md
```

This generates `content/posts/my-first-post.md` from the archetype in
`archetypes/posts.md`, pre-filled with front matter:

```yaml
---
title: 'My First Post'
date: 2026-07-15T10:00:00+00:00
draft: true
tags: []
---
```

Write normal Markdown below the front matter. Everything the stylesheet
supports is demonstrated in the sample post "A tour of the typography".

A few things to know:

- **Drafts.** While `draft: true` is set, the post appears in
  `hugo server -D` previews but is never included in a real build.
  Remove the line (or set it to `false`) to publish.
- **Tags.** Fill the list like `tags: [writing, hugo]`. Tag pages and
  the tag index are generated automatically.
- **Dates.** Posts are sorted and grouped by year on the home page using
  the `date` field. A post dated in the future won't be published until
  that date passes (or build with `hugo --buildFuture`).
- **Images.** Put shared images in `static/` (e.g.
  `static/img/photo.jpg`, referenced as `/img/photo.jpg`). For images
  that belong to one post, use a page bundle: create
  `content/posts/my-post/index.md` and drop the image next to it, then
  reference it by filename.
- **Delete the samples.** The three posts in `content/posts/` and the
  placeholder text in `content/about.md` are examples.

## 3. Host on GitHub Pages

The site is built and deployed by GitHub Actions from the public repo
`github.com/Mufabo/fmso` and served at
`https://fatihs-ai-dumping-ground.duckdns.org/`.

### How it works

`.github/workflows/deploy.yaml` runs on every push to `main`:

1. `actions/checkout` — checks out the source.
2. `peaceiris/actions-hugo` — installs Hugo extended, pinned to the same
   version used locally.
3. `hugo --minify` — builds the site into `public/`.
4. `actions/upload-pages-artifact` + `actions/deploy-pages` — publish
   `public/` to GitHub Pages. No `gh-pages` branch is involved.

### One-time setup

1. In the repo, **Settings → Pages → Build and deployment → Source:
   GitHub Actions**.
2. **Custom domain:** `static/CNAME` contains
   `fatihs-ai-dumping-ground.duckdns.org`, and the same name is set under
   **Settings → Pages → Custom domain**. Keep both in sync with
   `baseURL` in `hugo.toml`.
3. **DNS (DuckDNS):** DuckDNS can't do CNAMEs and only holds one IPv4,
   so point the domain at one GitHub Pages IP:

   ```
   https://www.duckdns.org/update?domains=fatihs-ai-dumping-ground&token=YOUR_TOKEN&ip=185.199.108.153
   ```

   (or set it in the DuckDNS web UI). Any of `185.199.108.153`,
   `.109.153`, `.110.153`, `.111.153` works. Disable any dynamic-IP
   updater that would overwrite it with your home IP.
4. Once DNS resolves and GitHub has issued the certificate, tick
   **Enforce HTTPS** in the Pages settings.

Reference: https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site

## Customizing

- `hugo.toml` — title, author, description, intro line, menu.
- `assets/css/main.css` — every color, font, and spacing decision lives
  in the token block at the top of the file.
- `layouts/` — the HTML templates, if you want structural changes.

## Troubleshooting

- **Unstyled page or broken links after deploy** — `baseURL` doesn't
  match the real URL (check the trailing slash and repo-name path).
- **Deploy job fails / Pages 404** — Pages source isn't set to
  "GitHub Actions" (step 1 above).
- **Custom domain shows a GitHub 404 or certificate error** — DNS still
  points elsewhere; check `dig fatihs-ai-dumping-ground.duckdns.org`.
- **Builds locally but fails in CI** — version drift; align
  `hugo-version` in the workflow with `hugo version` on your machine.
- **Post missing from the live site** — it is still `draft: true`, or
  its date is in the future.

## License

The repository is public. Consider adding a license: one for the code (MIT works well for templates) and consider a Creative
Commons license for your writing.
