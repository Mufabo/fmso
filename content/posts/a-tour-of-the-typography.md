---
title: 'A tour of the typography'
date: 2026-07-01
tags: [meta, design]
---

This post exercises every kind of content the stylesheet knows about. Keep
it around while you tweak the design, then delete it.

## Two voices

Prose is set in a serif — Charter where available, Georgia otherwise.
Machine facts wear a monospace instead: dates, tags, labels, and of course
`inline code`. The rule of thumb: if a human wrote it, serif; if the
computer knows it, mono.

## Structure

A second-level heading opens a section. Third-level headings subdivide
when a piece genuinely needs them.

### Lists

Ordered lists for sequences:

1. Write the draft.
2. Let it cool overnight.
3. Cut a third of it.

Unordered lists for everything else:

- Leaders connect a title to its date, like a book's table of contents.
- Year labels hang in the left margin on wide screens.
- Nothing on this site animates, apart from your own scrolling.

## Quotation

> The main thing is to write a lot, and to mean it. Style shows up on its
> own, uninvited, once the sentences are honest.

## Code

Fenced code blocks are highlighted at build time — no highlight.js, no
client-side work:

```go
func main() {
	posts := load("content/posts")
	for _, p := range posts {
		fmt.Println(p.Title, "—", p.Date.Format("2006-01-02"))
	}
}
```

## Tables

| Element    | Voice | Face             |
| ---------- | ----- | ---------------- |
| Body text  | Human | Charter, Georgia |
| Dates      | Machine | System mono    |
| Code       | Machine | System mono    |

## Small print

Footnotes work as you'd expect.[^1] A dotted rule separates major
movements within a post:

---

That's the whole vocabulary. It's deliberately small — a blog needs a
good paragraph far more than it needs components.

[^1]: They collect at the bottom, in quieter type.
