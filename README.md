# fatih.machtsachen.online — a personal Hugo blog

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

## 3. Host on Codeberg Pages

Codeberg Pages serves static sites for free. Your address depends on the
repository name:

| Repository name | Site URL                                  |
| --------------- | ----------------------------------------- |
| `pages`         | `https://USERNAME.codeberg.page/`          |
| anything else   | `https://USERNAME.codeberg.page/REPO-NAME/` |

### One-time setup

1. **Check `baseURL`** in `hugo.toml`. It is set to the custom domain
   `https://fatih.machtsachen.online/`; if you first want to publish on
   a codeberg.page URL from the table above, use that instead (trailing
   slash included). A mismatch breaks CSS and internal links — it is
   the most common mistake.
2. **Create a public repository** on Codeberg. With the custom-domain
   setup the repo name is free to choose; on codeberg.page it
   determines the URL as in the table above.
3. **Enable CI:** in the repository, go to **Settings → Units →
   Overview** and tick **Enable integrated CI/CD pipelines with Forgejo
   Actions**.
4. **Check the workflow target:** `.forgejo/workflows/deploy.yaml`
   already deploys to `site: 'https://fatih.machtsachen.online/'` with
   `server: 'codeberg.page'`. For a codeberg.page URL instead, set
   `site:` accordingly and remove the `server:` line.
5. **Add the DNS records** described under "Custom domain" below.
5. **Push the source** (not the `public/` folder) to `main`:

   ```sh
   git init -b main
   git add .
   git commit -m "New blog"
   git remote add origin https://codeberg.org/USERNAME/REPO-NAME.git
   git push -u origin main
   ```

That's it. From now on, every push to `main` rebuilds and publishes the
site. Watch progress and logs in the repository's **Actions** tab; the
first deploy usually appears within a minute or two.

### How the Forgejo Actions workflow works

The pipeline lives in `.forgejo/workflows/deploy.yaml` and has four
steps:

1. `actions/checkout` — checks out your repository on `codeberg-tiny`,
   one of Codeberg's free hosted runners.
2. `peaceiris/actions-hugo` — installs Hugo extended, pinned to the same
   version this project was built with (keep it in sync with your local
   version to avoid surprises).
3. `hugo --minify` — builds the site into `public/`.
4. `git-pages/action` — uploads `public/` straight to Codeberg's pages
   server. Authentication uses `${{ forge.token }}`, which Forgejo
   injects automatically — there are no secrets to configure.

Because deployment happens over HTTP, no `pages` branch is created; your
repository stays clean source code.

### Alternative: deploy without CI

If you prefer building on your own machine:

1. Run `hugo --minify` locally.
2. Commit the *contents* of `public/` to a branch named `pages`
   (`git switch --orphan pages` the first time).
3. In **Settings → Webhooks**, add a **Forgejo**-type webhook with your
   site URL (e.g. `https://USERNAME.codeberg.page/`) as the target URL
   and the branch filter set to `pages`.

Every push to the `pages` branch then triggers publication. Full docs:
https://docs.codeberg.org/codeberg-pages/

### Custom domain: fatih.machtsachen.online

This project deploys to `https://fatih.machtsachen.online/`. Codeberg's
current (git-pages) backend authorizes custom domains purely through
DNS — no `.domains` file is needed. Add two records in the DNS settings
of `machtsachen.online`:

| Host                              | Type  | Value                                    |
| --------------------------------- | ----- | ---------------------------------------- |
| `fatih`                           | CNAME | `codeberg.page.`                         |
| `_git-pages-forge-allowlist.fatih`| TXT   | `https://codeberg.org/USERNAME/REPO.git` |

- The **CNAME** routes the subdomain to Codeberg's pages server; the
  TLS certificate is issued automatically via Let's Encrypt.
- The **TXT** record authorizes Forgejo Actions deploys from exactly
  one repository — replace `USERNAME/REPO` with your repo's real HTTPS
  clone URL.
- Most registrars auto-append the domain, so enter only the host part
  shown above (not the full `...machtsachen.online`).
- If `machtsachen.online` uses **DNSSEC**, use `A 217.197.84.141` and
  `AAAA 2a0a:4580:103f:c0de::2` instead of the CNAME (IPs may change;
  verify against the docs).
- DNS changes can take a while to propagate; the first successful
  deploy after that gets the site live.

Reference: https://docs.codeberg.org/codeberg-pages/using-custom-domain/

### Hosting on your own Forgejo instance

This setup also works outside Codeberg, since both the CI and the pages
server are ordinary Forgejo-compatible software:

- Register a [Forgejo runner](https://forgejo.org/docs/latest/admin/actions/)
  for your instance and change `runs-on:` in the workflow to your
  runner's label — `codeberg-tiny` exists only on Codeberg.
- Run your own [git-pages](https://codeberg.org/git-pages/git-pages)
  server and point the `site:` parameter at it — or replace the deploy
  step entirely (e.g. `rsync` the `public/` folder to any web server).

## Customizing

- `hugo.toml` — title, author, description, intro line, menu.
- `assets/css/main.css` — every color, font, and spacing decision lives
  in the token block at the top of the file.
- `layouts/` — the HTML templates, if you want structural changes.

## Troubleshooting

- **Unstyled page or broken links after deploy** — `baseURL` doesn't
  match the real URL (check the trailing slash and repo-name path).
- **Nothing in the Actions tab** — Forgejo Actions isn't enabled for the
  repository (step 3 above).
- **Deploy step fails with an authorization error** — the `site:` URL in
  the workflow doesn't correspond to this repository's owner/name.
- **Builds locally but fails in CI** — version drift; align
  `hugo-version` in the workflow with `hugo version` on your machine.
- **Post missing from the live site** — it is still `draft: true`, or
  its date is in the future.

## License

Codeberg's terms expect public repositories to carry a free license. Add
one for the code (MIT works well for templates) and consider a Creative
Commons license for your writing.
