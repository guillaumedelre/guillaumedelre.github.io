# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repository is a personal tech article publication site hosted on GitHub Pages. Content is authored locally in Markdown and published by pushing to `main`, which triggers a GitHub Actions workflow that builds and deploys the site automatically.

## Documentation

- [Hugo](https://gohugo.io/documentation/)
- [Risotto theme](https://github.com/joeroe/risotto)

## Stack

Hugo blog hosted on GitHub Pages. Theme: Risotto (git submodule at `themes/risotto`), a minimal terminal-style theme, with the `tokyo-night-dark` palette. Site-specific styles live in `static/css/custom.css` (e.g. the `.gh-card` GitHub cards).

Hugo features active: RSS at `/index.xml`, sitemap at `/sitemap.xml`. There is no search index.

Risotto itself only provides the page shell (header, sidebar, footer with the language switcher). Everything else comes from the overrides in `layouts/` (see Layout overrides): date and reading time, series navigation, tags and series in the sidebar, the last-modified date when it differs from the publication date, and a sidebar table of contents for posts with `toc: true` in their frontmatter.

Taxonomies: `tags`, `series` (native Hugo taxonomy — no registration step needed).

Deployment: `.github/workflows/hugo-deploy.yml` builds with `hugo --minify` and deploys to GitHub Pages via the Pages environment. Push to `main` triggers the workflow.

## Local development

Requires Docker. No local Hugo install needed.

```bash
docker run --rm -v "$(pwd):/src" -u "$(id -u):$(id -g)" \
  -p 1313:1313 hugomods/hugo:exts \
  hugo server --bind 0.0.0.0 --buildDrafts
```

The site is served at `http://localhost:1313` with live reload.

## Writing a post

Create a file in `content/posts/` following the naming convention:

```
content/posts/YYYY-MM-DD-slug.md
```

Required frontmatter:

```yaml
---
title: "Post title"
date: YYYY-MM-DD
tags: [tag1, tag2]
format: experiment   # experiment | showcase | guide — see Writing style
description: "One-sentence summary (50-160 characters)."
---
```

Optional: `images: ["/images/<slug>/<file>.png"]` sets the Open Graph / Twitter card image used when the post is shared (it is not displayed on the page). Use it whenever the post has a representative screenshot or photo; otherwise the site default applies. There are no categories: tags cover the topics.

Permalink format is `/:year/:month/:day/:slug/` as configured in `hugo.yaml`.

When making a significant update to an existing post (content change, not a typo fix), add or update the `lastmod` field in the frontmatter:

```yaml
lastmod: YYYY-MM-DD
```

Without this field, Hugo falls back to the file's modification date, which changes on every `git checkout` and produces an unreliable freshness signal in the sitemap.

## Before writing a new post

Before drafting any article, always ask the user for:

1. **Date** — publication date in `YYYY-MM-DD` format (used in both the filename and the frontmatter).
2. **Tags** — propose a list based on existing tags already used across posts. Check `content/posts/` frontmatter to build the suggestion. Prefer reusing existing tags over creating new ones; only propose new tags when no existing tag covers the concept. Keep EN and FR tags identical. When a new tag is created, add `content/en/tags/<slug>/_index.md` and `content/fr/tags/<slug>/_index.md` with its display name as `title` (e.g. `Claude Code`, `BME280`), otherwise Hugo shows the capitalized slug.
3. **Description** — write a one-sentence summary (50-160 characters) for the `description` frontmatter field. Used for the meta description (`layouts/partials/head.html`), Open Graph and Twitter cards (Hugo built-in templates), post lists and the post sidebar.
4. **Language** — ask whether the article will have a French translation. If yes, create both `slug.md` and `slug.fr.md` with a shared `translationKey`.
5. **Series** — ask whether the post belongs to an existing series or starts a new one (see the Series section below).
6. **Format** — propose `experiment`, `showcase` or `guide` based on the topic (see Writing style) and set it in the `format` frontmatter field. The format drives structure, tone and length.
7. **Share image** — propose a screenshot or photo for `images` if one exists.

After publishing a new post, check `static/robots.txt`: if you added a path that should be blocked from indexing (e.g. a draft or admin page), add a `Disallow` rule. Standard posts need no change.

## Series

A series is a group of thematically linked articles published under a shared title. Hugo handles series as a native taxonomy — no registration file or custom layout needed.

### Adding an article to a series

Add two fields to the post frontmatter:

```yaml
series: ["kubernetes-migration"]
part: 1
```

`series` is an array containing the series slug. `part` is the article's position in the series (integer). `layouts/partials/series_nav.html` uses `part` to show "part X of Y" with previous/next links at the bottom of each post, and `layouts/_default/taxonomy.html` uses it to order the series listing.

### Creating a new series

No registration step required. Adding `series: ["my-new-series"]` to a post frontmatter is enough — Hugo creates the taxonomy page at `/series/my-new-series/` automatically.

To add a title and description to the series index page, create `content/series/my-new-series/_index.md`:

```yaml
---
title: "Human-readable series title"
description: "One sentence describing the series."
---

One paragraph introducing the series.
```

### Before writing a new post in a series

- Add `series` and `part` to the frontmatter
- Do not add a manual series footer — `series_nav.html` generates it automatically
- If the series is new, optionally create the `_index.md` page above for a richer series listing

## Layout overrides

Custom templates live in `layouts/` and take precedence over the Risotto theme (`themes/risotto/layouts/`). When creating or modifying an override:

- Read the theme template it replaces in full before writing the override, including all partials it calls, and keep its `main` / `aside` blocks and CSS classes so the palette still applies
- For each partial in the reference template, explicitly decide: include it, replace it with contextually appropriate data, or omit it with a reason
- Reuse the existing partials (`post_meta.html`, `post_list.html`, `terms.html`, `series_nav.html`) instead of re-implementing them
- Metadata that makes no sense in the new context should be replaced, not dropped silently (e.g., reading time on a series listing becomes article count)
- Styles go in `static/css/custom.css` using the palette's CSS variables (e.g. `var(--link)`), never inline

Current overrides:
- `_default/single.html` — post page: date and reading time, content, series navigation; sidebar with description, last-modified date, tags, series and the optional table of contents
- `_default/taxonomy.html` / `_default/terms.html` — a term's post list (series sorted by `part`) and the term index with post counts
- `_default/archives.html` — posts grouped by year
- `_default/rss.xml` — RSS feed with series-aware descriptions
- `_default/_markup/render-link.html` — external links open in a new tab (see External links)
- `index.html` / `posts/list.html` — home page with the five latest posts, paginated post list
- `partials/head.html` — meta description, canonical, hreflang, Open Graph / Twitter / schema.org, Google Analytics in production only

## External links

Markdown links to external domains automatically open in a new tab — a render hook at `layouts/_default/_markup/render-link.html` adds `target="_blank" rel="noopener noreferrer"` to any link whose destination starts with `http`.

Use standard Markdown syntax for all links:

```markdown
[link text](https://example.com)
```

Internal links (relative paths) are rendered normally with no `target` attribute.

## Language

Posts are primarily written in English. A French translation is optional for any article.

### Keeping translations in sync

When an article is modified — content, frontmatter, or both — always propagate the changes to all its translated versions:

- Update the translated prose to reflect the modified content.
- Mirror any frontmatter change: `title`, `description`, `date`, `tags`, `format`, `images`, `series`, `part`. Translate `title` and `description`; copy the other fields verbatim.
- If a section is added or removed in EN, add or remove the equivalent in FR.
- If a code block or technical example changes, update it identically in all translations (code is not translated).

Check for a matching file in `content/fr/posts/` with the same filename before saving any edit to a file in `content/en/posts/`.

To add a French version of an existing post:

1. Create `content/posts/YYYY-MM-DD-slug.fr.md` with the translated content.
2. Add `translationKey: "slug"` to the frontmatter of both the EN and FR files.

Hugo automatically generates `/fr/YYYY/MM/DD/slug-fr/`, adds `hreflang` tags for SEO, and Risotto shows a language switcher in the footer.

Articles without a French version remain available in English only — no placeholder or stub needed.

## Icons

When an icon is needed, use one of these two sources:

- [Simple Icons](https://simpleicons.org) for brand and tech icons (GitHub, PHP, Symfony, Docker, Redis, etc.)
  CDN: `https://cdn.simpleicons.org/{icon-name}`
- [Flaticon](https://flaticon.com) for general icons not covered by Simple Icons (LinkedIn, etc.)
  CDN: `https://cdn-icons-png.flaticon.com/{size}/{pack}/{id}.png`

Embed with `width="16"` and `style="vertical-align: middle"` for inline icons.

For social links, use standard Markdown or HTML with the appropriate CDN icon.

## Anonymization

When writing an article based on real code or experience from a professional project, anonymize all references to the company and codebase:

- Never mention the employer, client or internal project names. The exact list lives in `CLAUDE.local.md` (not versioned, so the names never reach this public repository): @CLAUDE.local.md
- More generally, never mention any identifiable project or company name.
- Replace specific names with generic terms: "the platform", "the project", "a microservices codebase", etc.
- This applies to code snippets (namespaces, class prefixes), narrative context, and any incidental detail that would identify the source.

## API Platform releases series

Articles in the `api-platform-releases` series cover one minor or major release per post. Each article must be verified against the actual source code before publishing — the release notes and blog posts are not sufficient.

### Sources to check (in order)

1. **Release date** — `https://github.com/api-platform/core/releases` (tag `vX.Y.0`, field `published_at`)
2. **CHANGELOG** — `https://github.com/api-platform/core/blob/X.Y/CHANGELOG.md` for what actually shipped and when
3. **Official docs** — `https://github.com/api-platform/docs` for usage examples and configuration guides maintained by the team
4. **Interface signatures** — fetch the PHP file directly from the versioned branch of `api-platform/core`, not `main`
5. **Config keys** — verify in `src/Symfony/Bundle/DependencyInjection/Configuration.php` on the target branch
6. **Class namespaces** — fetch the file from the branch tree; never infer from memory

### What to verify for each claim

| Claim type | What to check |
|---|---|
| Release date | `published_at` on the GitHub API, not the blog post |
| "X is the minimum version" | `composer.json` at the release tag (`require` section) |
| Interface return type | The PHP interface file on the versioned branch |
| Config YAML key | `Configuration.php` — the exact node name, not a guess |
| Class or attribute namespace | The actual file path in `src/` on the branch |
| Constructor parameter name | The constructor signature in the source file |
| Query parameter format | Source file + PHPDoc example (e.g. ExistsFilter docblock) |
| "Feature X is new in Y.Z" | CHANGELOG entry — if absent, the feature likely predates this release |
| "Feature X was removed" | CHANGELOG + verify the file is actually gone from the branch |

### Recurring mistakes to avoid

- **ProviderInterface return type** is `object|array|null`, not `object|iterable|null`
- **Config keys use snake_case** matching the PHP node name exactly — do not guess camelCase or plurals (`x-apiplatform-tag` not `x-apiplatform-tags`, `enable_json_streamer` not `use_json_streamer`)
- **ObjectMapper** comes from `Symfony\Component\ObjectMapper\Attribute\Map`, not an `ApiPlatform\ObjectMapper` namespace; the constructor parameter is `source:`, not `from:`
- **DBAL/Symfony/ORM minimums** — check `composer.json` constraints; they often support two major versions simultaneously (`^6.4 || ^7.0`)
- **Laravel bridge** uses a dedicated `policy:` parameter on operations, not a Symfony security expression string
- **ExistsFilter** query format is `?exists[property]=true`, not `?property[exists]=true`
- **Event listener removal** — in 3.2 they were made optional (`event_listeners_backward_compatibility_layer`), not removed

### Version references

Always pin code examples to the version being discussed. Do not use `main` branch as a reference — APIs change between releases.

## PHP releases series

Articles in the `php-releases` series cover one minor or major release per post. Each article must be verified against the official PHP sources before publishing — the RFC text is authoritative; the PHP manual and `UPGRADING` file are the ground truth for syntax and behavior.

### Sources to check (in order)

1. **Release date** — `https://www.php.net/releases/` (exact day and month, not just year)
2. **UPGRADING file** — `https://github.com/php/php-src/blob/PHP-X.Y/UPGRADING` — the canonical list of new features, deprecations, and removals for that version; always read this first
3. **RFC** — `https://wiki.php.net/rfc` — one RFC per feature; the RFC is authoritative on semantics, edge cases, and the exact syntax adopted
4. **php.watch** — `https://php.watch/versions` — curated per-version summaries with RFC links, code examples, and migration notes; useful for cross-checking claims and spotting missed features
5. **PHP manual** — `https://www.php.net/manual/en/` — for function signatures, class names, constants, and return types
6. **ChangeLog** — `https://www.php.net/ChangeLog-8.php` (or `-7.php`) — for confirming which patch first shipped a fix or addition
7. **php-src** — `https://github.com/php/php-src` — for exact implementation details when the manual is incomplete

### What to verify for each claim

| Claim type | What to check |
|---|---|
| Release date | `php.net/releases/` — day and month, not just "November" |
| New function | Manual page: parameters, types, return type, since version |
| New class or namespace | Manual + RFC: exact namespace (`BcMath\Number`, `Dom\HTMLDocument`, `Pdo\Mysql`) |
| Syntax change | RFC "Proposal" section + `UPGRADING` for the exact accepted form |
| Default value change | `UPGRADING` "Changed" section + manual (e.g. bcrypt cost) |
| Deprecation | `UPGRADING` "Deprecated" section — what exactly, not just the category |
| Removed feature | `UPGRADING` "Removed" section — confirm it's actually gone, not just deprecated |
| "Feature X was added in Y.Z" | Check UPGRADING for Y.Z and earlier versions — attribution errors are common |
| Library version (PCRE, libxml…) | `UPGRADING` or php-src `configure.ac` / release notes |

### Recurring mistakes to avoid

- **Version attribution** — do not rely on memory for which minor version introduced a feature; always confirm in `UPGRADING`. Many features appear in the wrong article.
- **Function parameter names** — check the manual signature exactly; named arguments make parameter names part of the public API (`round($num, $precision, $mode)` not `$value`).
- **Class namespaces** — PHP 8.4 introduced `Dom\`, `Pdo\`, and `BcMath\` namespaces; do not invent variants.
- **Asymmetric visibility** — the syntax is `public private(set)`, not `public readonly` and not `public(get) private(set)`.
- **Property hooks** — interfaces *can* declare hooked properties; verify the exact constraints (no backing store on abstract hooks, etc.) against the RFC.
- **`request_parse_body()`** — returns `[array, array]` (post data, files), not a single array.
- **Deprecation vs removal** — something deprecated in 8.X is typically removed in 9.0, not in the next minor. Do not say "removed" unless `UPGRADING` confirms it.
- **bcrypt cost default** — changed from 10 to 12 in 8.4; verify similar defaults against the actual source before stating them.

### Frontmatter conventions

```yaml
series: ["php-releases"]
part: <integer matching release order, 7.0=1>
tags: [php, ...]
```

The `part` field follows the chronological release order across the full series (7.0 = part 1, 7.1 = part 2, etc.).

## Symfony releases series

Articles in the `symfony-releases` series cover one minor or major release per post. Symfony releases follow a strict schedule (minor every 6 months, LTS every 2 years) and the X.0 release always ships the same day as the X-1.Y LTS. Verify these relationships before writing.

### Sources to check (in order)

1. **Release date** — `https://symfony.com/releases` or `https://github.com/symfony/symfony/releases` (tag `vX.Y.0`, exact day)
2. **Official blog post** — `https://symfony.com/blog/` — the release announcement lists every feature with links to the relevant PRs and docs
3. **UPGRADE file** — `https://github.com/symfony/symfony/blob/X.Y/UPGRADE-X.Y.md` — authoritative list of BC breaks, removals, and renamed classes
4. **Per-component CHANGELOG** — `https://github.com/symfony/symfony/blob/X.Y/src/Symfony/<Component>/CHANGELOG.md` — details per feature with the minor version it landed in
5. **Official docs** — `https://symfony.com/doc/X.Y/` — for configuration keys, attribute names, and method signatures pinned to the release
6. **symfony.com/components** — for checking if a component is experimental, stable, or deprecated at a given version

### What to verify for each claim

| Claim type | What to check |
|---|---|
| Release date | `symfony.com/releases` — Symfony releases on a fixed schedule; day and month matter |
| "X.0 released the same day as Y.Z LTS" | Confirm both tags on GitHub — this is always true for major releases |
| PHP minimum version | `composer.json` at the release tag (`require.php`) |
| New attribute | Source file + docs: exact class name, namespace, constructor parameters |
| "Feature X introduced in Y.Z" | Per-component CHANGELOG — "introduced" vs. "graduated to stable" is a key distinction |
| "Feature X is now stable" | `@experimental` removed in that version's CHANGELOG entry |
| Class or namespace move | UPGRADE file "Renamed classes" or "Moved classes" section |
| Method signature change | Source file on the versioned branch |
| Config option name | Docs for the version + Configuration class in the component |
| Removal | UPGRADE file "Removed" section — verify the class/method is actually absent from the branch |
| Command option | `bin/console help <command>` output or source `InputOption` definitions |

### Recurring mistakes to avoid

- **Introduced vs. stable** — Symfony often introduces features as `@experimental` in X.N and graduates them in X.N+1 or X+1.0. Do not say "introduced in 7.0" if it was introduced in 6.3 and just stabilized in 7.0.
- **Same-day releases** — Symfony X.0 and X-1.Y LTS always ship the same day. Do not give two different dates.
- **Attribute namespaces** — Symfony DI attributes (`#[AutowireLocator]`, `#[Target]`, `#[AsAlias]`) live in `Symfony\Component\DependencyInjection\Attribute\`; scheduler attributes in `Symfony\Component\Scheduler\Attribute\`. Verify the full namespace.
- **`UriSigner` namespace** — moved from `HttpKernel` to `HttpFoundation` in 7.0; do not use the old namespace in examples.
- **Removed vs. deprecated** — Doctrine annotations were deprecated in 5.2 and removed in 7.0. State the right version for each action.
- **LTS support window** — Symfony LTS versions receive bug fixes for 3 years and security fixes for 4 years. Do not confuse with the standard 8-month support window.
- **Component graduation** — check whether a component (e.g. AssetMapper, Scheduler) was marked experimental at the time of the release being described.

### Frontmatter conventions

```yaml
series: ["symfony-releases"]
part: <integer matching release order, 3.3=1>
tags: [symfony, ...]
```

The `part` field follows the chronological release order across the full series starting from the first article (3.3 = part 1).

## Writing style

Every post belongs to one of three types: **experiment**, **showcase** or **guide**. Decide the type before drafting (propose one, confirm with the user if unclear). The type sets the structure, the length and how much of the story goes into the text. The **Voice** rules apply to all three.

The common thread: these are posts by someone who built and ran the thing, not documentation and not marketing. The reader should feel that a specific person wrote it.

### Voice (all types)

- First person, direct, casual. Write like explaining to a curious colleague over coffee, not like writing a README.
- Address the reader consistently. English: "you". French: always "vous" — never "tu", "ton", "ta", "toi", "t'". "On" is fine for general statements, not as a way to dodge addressing the reader.
- Give opinions in the body, not only at the end. If something is bad, awkward or overrated, say so and say why.
- Specifics beat adjectives: the time lost, the exact error message, the number, the hardware model. "It took a while" is weaker than "three evenings".
- Vary sentence length. A long explanatory sentence followed by a short blunt one creates pacing. Avoid stacks of three uniform paragraphs in a row — they read as documentation regardless of pronoun choice.
- Humor only when it comes from the situation itself. No forced jokes, no decorative emoji in headers.
- Avoid: "In this post we will...", "As you can see...", bullet-point summaries of what was just said, superlatives ("revolutionary", "powerful", "game changer").

### Experiment

A personal project or hack where things didn't go as planned. Default type. Examples: `dream-cheeky-thunder`, `bme280`.

**Goal:** make the reader want to read to the end.

- Open with the problem or the odd situation that triggered the project. The first paragraph makes a specific promise: a question that will be answered, a mystery, a number that doesn't make sense, a setup that shouldn't work. "I rescued a missile launcher from a drawer" is a setup; "I rescued a missile launcher and spent three hours fighting Docker over a permission error" is a promise.
- Build tension: the obstacle, the wrong assumption, the unexpected behavior. Every solution comes with what failed first or what was assumed wrong. A solution without its dead-end reads like documentation; a solution with one reads like experience.
- Section headers mark narrative beats, not topic buckets. Prefer "When the container couldn't see the USB" over "Docker setup".
- Resolve with the technical solution, focused on *why* it matters, not *what* the code does.
- End with honest takeaways — what surprised, what failed, what's worth keeping. Prose by default; a list only if the takeaways are genuinely independent of each other.
- Length: 800–1500 words.

### Showcase

Presenting a tool I built (or adopted) so that readers try it. Example: `claude-agents-board`.

**Goal:** within 30 seconds the reader knows whether it's for them; by the end they know how to try it.

- Open with the pain, as a concrete scene ("the third session has been waiting twenty minutes for me to approve a `psql` command"), then say in one sentence what the tool is.
- If installation is one command, show it in the first screen of text.
- Say why I built it instead of using an existing alternative — one opinionated paragraph, naming the alternatives.
- Organize sections by the reader's questions or use cases ("When the context fills up"), not by screen or feature. In each section: the problem first, the feature second.
- At most one screenshot per section, with a caption that says what to look at.
- Honesty is the selling point: a "What it doesn't do" section is mandatory, including security limits and known gaps.
- No marketing vocabulary, no superlatives, no "simply" or "just" to hide complexity.
- End with who should try it, how, and what kind of feedback or contribution is welcome. Then the GitHub card.
- Length: 600–1200 words. Anything already in the README gets a link, not a copy.

### Guide

A step-by-step reference the reader follows with a terminal open, and comes back to later. Example: `homelab-docker-compose-traefik`.

**Goal:** the reader reaches a working result without guessing.

- Say it's a guide in the `description` and in the first paragraphs. State the prerequisites and the end result (what the reader will have running at the end).
- Keep the story for the intro (why I set it up this way, one or two paragraphs) and for the pitfalls, which should be told as things that actually happened.
- Topic headers and numbered steps are allowed here — and only here. Above ~1500 words, set `toc: true` in the frontmatter to get the sidebar table of contents; never write one by hand in the post.
- Each step: what to do, how to check that it worked, the most common failure. Commands are complete, copy-pastable and tested; versions are pinned or stated.
- "Why X and not Y" justifications: one opinionated paragraph each, not a feature comparison.
- Pitfalls follow the shape *symptom → cause → fix*, with the exact error message when there is one.
- End with what the guide doesn't cover and where to go next. No closing aphorism needed.
- Length: as long as needed, but never longer than the steps require.

### Mixed posts

When a post mixes types (e.g. `traefik-local-https`: news analysis then a setup guide), pick the dominant type for the opening and the ending, and keep the other part short and clearly separated by a header.

### Technical content (all types)

- Include code and commands only when they illustrate a non-obvious point (guides excepted: there, every step needs its command).
- Tables earn their place only if the rows differ in ways prose can't show compactly (benchmarks, comparisons, dimensions). A table that lists features in the same shape as the README should be replaced by one sentence describing the *shape* of the thing, with a link to the source.
- Diagrams support explanations; they don't replace them.
- Do not reproduce what is already in the project README or docs — link to them instead.

## Review

Run this review whenever the user asks for one, and on every draft before presenting it as finished. Review both language versions when they exist.

Report findings as a list, each with: the rule, the quoted passage (with line number), and a suggested fix. Fix mechanical issues directly (typos, agreement, typography, reader address). Propose — don't apply — changes to structure, voice or content, and let the user decide.

### 1. Type fit

- [ ] The type set in `format` (experiment / showcase / guide) is identifiable from the first two paragraphs, and the structure follows that type's rules.
- [ ] Experiment: at least one dead-end or wrong assumption is shown before a solution.
- [ ] Showcase: a "What it doesn't do" section exists; no superlative; the alternatives are named.
- [ ] Guide: prerequisites and end result are stated up front; every step can be verified; pitfalls have symptom, cause and fix.

### 2. Opening and ending

- [ ] The first paragraph could not open any other post. If it could, it's too generic.
- [ ] The last paragraph adds something (a lesson, a limit, a next step) instead of restating the post.

### 3. Formula tics

Each pattern is fine once. A post that stacks them reads as machine-polished, and so does a blog where every post uses the same ones. Compare against the two most recently published posts, not just the draft.

- [ ] "Ce n'est pas X. C'est Y." / "Not X. It's Y." / "Pas X : Y." — at most once per post.
- [ ] One-line aphorism as the final sentence ("The perfect homelab doesn't exist. The one that runs does.") — never in two consecutive posts.
- [ ] Takeaway lists with a bolded lead sentence ("**X is not optional.** explanation") — at most one per post, never by default.
- [ ] Groups of exactly three items or adjectives used for rhythm rather than because there are three things.
- [ ] Paragraphs ending on a short punchline sentence — no more than about one paragraph in four.
- [ ] Filler intensifiers: "vraiment", "simplement", "essentiellement", "really", "just", "actually", "basically" — delete unless they change the meaning.

### 4. Specificity and voice

- [ ] Every claim of difficulty or effort comes with a concrete detail (duration, error message, number, hardware).
- [ ] Sentences that could appear in any tech blog are cut or made specific.
- [ ] At least one personal opinion appears in the body of the post.
- [ ] The register stays consistent; a guide may become neutral inside its steps, but not elsewhere.

### 5. Reader address

- [ ] English: "you" throughout.
- [ ] French: "vous" throughout. `grep -nwE "tu|ton|ta|tes|toi" <file>` and check for `t'` before a verb.

### 6. French posts

The French text must read as written in French, not translated from English. Read it aloud: a sentence you wouldn't say to a French colleague gets rewritten.

- [ ] No calques from English. Known offenders and replacements:

  | Calque | Source | Prefer |
  |---|---|---|
  | « porte tout le poids » | does the heavy lifting | « fait tout le travail », « c'est elle qui compte » |
  | « X bat Y » | X beats Y | « mieux vaut X que Y » |
  | « semble juste » | feels right | « est naturel », « tombe sous le sens » |
  | « ça compte si… » | it matters if… | « c'est important si… » |
  | « faire sens » | make sense | « avoir du sens » |
  | « à la fin de la journée » | at the end of the day | « au final » |
  | « adresser un problème » | address a problem | « traiter », « régler » |
  | « supporter » (une version) | support | « prendre en charge » |

- [ ] Anglicisms: use the French word when it is common (« configuration » or « installation » rather than « setup », « certificat wildcard » rather than « wildcard cert »). Keep terms that have no French usage (hook, commit, pull request). Pick « conteneur » or « container » and stick to it within a post.
- [ ] Consistent gender for borrowed words: une stack, un hook, un workflow, une pull request, une issue, un container.
- [ ] Agreement, especially on borrowed nouns (« pull requests bienvenues »).
- [ ] Typography: a space before `:` `;` `!` `?`; French quotes « » with inner spaces instead of `"…"` in prose.

### 7. Facts and links

- [ ] Versions, dates, numbers and quoted behaviours are verified against a primary source (release page, changelog, source code), not memory.
- [ ] Commands and config snippets were run, or are explicitly marked as untested.
- [ ] External links point to stable or versioned URLs; internal links use `relref`.
- [ ] Every image has alt text describing what the reader should see.

### 8. Frontmatter

- [ ] `format` is set; `images` is set when the post has a representative image.
- [ ] `description` is 50–160 characters and matches the type (a guide says it's a guide).
- [ ] `lastmod` is updated if the content changed significantly.
- [ ] Tags reuse existing ones and are identical in EN and FR.
