# AGENTS

This repository contains the Hugo sources for [chenchik.me](https://chenchik.me), a personal site built with the PaperMod theme. The site is published from `main` through GitHub Pages (`.github/workflows/hugo.yaml`). All agents should keep the repository production-ready and avoid breaking the automated deployment.

## Repo map and tooling
- `hugo.yaml` – central config (menus, params, theme import, social links).
- `content/` – Markdown sources for posts; use existing files as templates.
- `archetypes/` – optional front matter templates (unused today but add here when repeated post types are needed).
- `assets/` and `static/` – theme overrides, images, favicons.
- `public/` – Hugo build artifacts kept in-repo for local previews; CI regenerates it so do not hand-edit.
- Go modules (`go.mod`, `go.sum`) pin the PaperMod theme; run `hugo mod get -u` if the theme must be updated.

Recommended tool versions:

```bash
hugo version         # use Hugo Extended v0.152.2+ to match CI
npm install          # only if you add JS/CSS tooling that requires Node
```

## Local workflows
- `hugo server -D` – run the development server with drafts enabled when editing or drafting posts.
- `hugo --gc --minify` – replicate the CI build locally; cleans generated resources and writes the minified site into `public/`.
- `hugo mod tidy` – clean module references if PaperMod or other Hugo modules change.

## Continuous deployment
`Deploy Hugo site to Pages` (`.github/workflows/hugo.yaml`) runs on every push to `main`. It installs Hugo Extended 0.152.2, builds the site with `--gc --minify`, uploads `public/` as an artifact, and deploys to GitHub Pages. Changes that break the build block deployment, so agents should run the local build command when touching configuration/theme code.

## Agent guide

### Content Author Agent
- Scope: create or update Markdown posts in `content/posts/`.
- Expectations:
  - Use front matter similar to `content/posts/why-cognitive-bias.md`: `title`, ISO8601 `date`, `draft`, `description`, `summary`, `tags`, and PaperMod toggles such as `ShowToc`.
  - Keep tone conversational and first-person unless the topic demands otherwise.
  - Include outbound links using Markdown; prefer HTTPS and cite the source (books, posts, tools).
  - For posts with hero images, use the nested `cover.image` structure as seen in `content/posts/simply-receipts.md`.
- Checklist: spell-check, ensure headings progress (`##` then `###`), add tags, and run `hugo server` to confirm the post renders correctly.

### Editor Agent
- Scope: improve clarity, structure, and metadata of existing posts.
- Expectations:
  - Preserve author’s voice but fix grammar, pacing, and flow.
  - Ensure summaries stay under ~30 words and descriptions are compelling snippets for social cards.
  - Verify internal links (e.g., `/about`, `/tags/`) resolve to real pages.
- Checklist: re-run `hugo server -D` to verify ToCs, code fences, and typography after edits.

### Site Maintainer Agent
- Scope: configuration, theme overrides, assets, and performance tweaks.
- Expectations:
  - Modify `hugo.yaml` for menus, params, or social links; avoid removing existing sections without a replacement plan.
  - PaperMod overrides live in `assets/`; prefer SCSS additions over editing theme sources.
  - Update favicons or static files in `static/`; keep filenames stable when possible to avoid cache issues.
  - When upgrading Hugo or PaperMod, update `hugo.yaml`, `go.mod`, and CI’s `HUGO_VERSION` in `.github/workflows/hugo.yaml` in one change.
- Checklist: run `hugo --gc --minify` and open `/public/index.html` locally to confirm styling before pushing.

### Release/QA Agent
- Scope: validation before merging to `main`.
- Expectations:
  - Ensure `public/` reflects the latest build (run `hugo --gc --minify`).
  - Spot-check canonical pages (home, `/archives`, latest post) for layout issues.
  - Confirm GitHub Actions status badges stay green; rebuild locally if the pipeline fails.
- Checklist: remove stray TODOs, confirm no draft posts are accidentally published, and verify `resources/` or `public/` diffs contain generated content only.

## Writing standards
- Prefer US English and concise paragraphs.
- Highlight actionable takeaways with bullet lists or numbered steps to match the style in `content/posts/why-cognitive-bias.md`.
- Always fill `tags` with 1–4 lowercase keywords; introduce new tags only when an existing one is insufficient.
- Include `description` for SEO and `summary` for PaperMod cards; they can differ slightly so cards remain punchy.
- Quote `description`/`summary` front matter strings that contain punctuation like colons so Hugo/YAML parsing never breaks.
- Show/hide the table of contents using `ShowToc`/`TocOpen` depending on article length (enable for pieces longer than ~400 words).

## Adding media
- Store large media in `static/images/...` and reference with absolute paths (`/images/...`) to leverage Hugo’s static handling.
- For optimized processing (resizing, fingerprinting), place source assets under `assets/` and use Hugo Pipes in templates or shortcodes.
- When adding a new cover image, include width/height metadata if known to reduce layout shift.

## Troubleshooting tips
- Build fails with module errors: run `hugo mod tidy` and commit updated `go.sum`.
- Missing styles locally: ensure you use the Hugo **Extended** build; standard builds omit SCSS processing.
- GitHub Pages shows stale content: verify the Actions log uploaded the latest `public/` artifact and that caching is disabled in your browser when checking.
