---
title: 'This site is alive'
date: 2026-03-02
tags: [meta]
---

This is a sample post — delete it once you have something of your own to
say. But before you do, here is how the machine works.

Every post is a Markdown file in `content/posts/`. To start a new one:

```sh
hugo new posts/my-first-real-post.md
```

That creates a file with `draft: true` in its front matter. Drafts show up
when you preview locally with `hugo server -D`, but they are never
published. When a post is ready, remove the draft line (or set it to
`false`), commit, and push. That's the entire workflow.

The design here makes a few deliberate promises:

- **No JavaScript.** Nothing runs in your reader's browser.
- **No webfonts, no CDNs.** The type is set in Charter, Georgia, or
  whatever good serif the reader's system already has.
- **No analytics.** If you want to know whether people read your writing,
  ask them.

Dark mode follows the reader's system preference. The RSS feed lives at
[`/index.xml`](/index.xml) and updates itself. Everything else is just
writing.
