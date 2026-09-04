# ios-experiences

Personal experiences of using Swift, Objective-C, etc.

## Layout

```
articles/<topic>/<article>.md      prose
assets/<topic>/<image>.png         images, mirroring the articles/ tree
drafts/                            local staging, untracked
```

`assets/` mirrors `articles/` one-to-one: an article in `articles/feed/layout/`
keeps its images in `assets/feed/layout/`.

Topics: `deeplink`, `feed/layout`, `feed/crashes`, `notification`,
`notification/crashes`, `onboarding`, `performance`, `tech-design`, `webview`.

## Referencing an image

Paths are relative to the Markdown file, so an article needs one `../` per
directory level it sits under:

| Article | Image reference |
| --- | --- |
| `articles/deeplink/udl-routing.md` | `../../assets/deeplink/flow.png` |
| `articles/feed/layout/cell-sizing.md` | `../../../assets/feed/layout/trace.png` |

```markdown
![Routing flow](../../assets/deeplink/flow.png)
```

Markdown has no width syntax; use HTML when an image needs constraining:

```markdown
<img src="../../assets/deeplink/flow.png" width="480" alt="Routing flow">
```

A leading slash (`/assets/...`) resolves against the site root, not the repo —
it will not render.

## File naming

Lowercase, hyphen-separated, no spaces — `cell-sizing.md`, `flow-diagram.png`.

GitHub serves paths case-sensitively while macOS checkouts are case-insensitive,
so a mismatched capital renders fine locally and breaks once pushed.

## drafts/

`drafts/` is git-ignored. Notes land there while being rewritten, so nothing
reaches this public repo before it is ready.
