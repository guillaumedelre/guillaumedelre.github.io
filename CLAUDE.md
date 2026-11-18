# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repository is a personal tech article publication site hosted on GitHub Pages. Content is authored locally in Markdown and published by pushing to `main`, which triggers a GitHub Actions workflow that builds and deploys the site automatically.

## Documentation

- [Hugo](https://gohugo.io/documentation/)
- [Blowfish theme](https://blowfish.page/docs/) — in particular [configuration](https://blowfish.page/docs/configuration/), [front matter](https://blowfish.page/docs/front-matter/) and [shortcodes](https://blowfish.page/docs/shortcodes/). The same docs ship with the theme in `themes/blowfish/exampleSite/content/docs/`.

## Stack

Hugo blog hosted on GitHub Pages, written in French only and served at the site root (no `/fr/` prefix).

Theme: Blowfish (git submodule at `themes/blowfish`, v3.10.0). It requires Hugo 0.166–0.169 extended; newer versions build with a compatibility warning. Configuration lives in `config/_default/`: `hugo.toml` (site, taxonomies, permalinks, outputs), `languages.fr.toml` (title, description, author), `menus.fr.toml`, `params.toml` (theme features) and `markup.toml` (required by the theme).

There are no template overrides: everything comes from Blowfish and its configuration. Before changing how something looks or behaves, look for a parameter in `params.toml` or a front matter field. If nothing fits, discuss it with the user before adding anything under `layouts/` or `assets/css/custom.css`.

Features enabled in `params.toml`: profile home page with recent posts as cards, search (`/index.json`), sticky table of contents on every post, reading progress bar, reading time and word count, last-updated date, breadcrumbs, related posts, previous/next links, sharing links, "edit on GitHub" link, author box, code copy button, zen mode, light/dark switcher.

Also generated: RSS at `/index.xml`, sitemap at `/sitemap.xml`. Google Analytics is configured in `hugo.toml`.

Taxonomies: `tags`, `series`. There are no categories.

Deployment: `.github/workflows/hugo-deploy.yml` builds with `hugo --minify` and deploys to GitHub Pages via the Pages environment. Push to `main` triggers the workflow.

## Local development

Requires Docker. No local Hugo install needed. The official image is used because Blowfish needs a recent Hugo.

```bash
docker run --rm --name hugo-blog -v "$(pwd):/project" -u "$(id -u):$(id -g)" -e HOME=/tmp \
  -p 1313:1313 ghcr.io/gohugoio/hugo:latest \
  server --bind 0.0.0.0 --buildDrafts
```

The site is served at `http://localhost:1313` with live reload. To open it from another machine on the LAN, add `--baseURL http://<LAN IP>/ --port 1313`.

## Writing a post

Create a file in `content/posts/` following the naming convention:

```
content/posts/YYYY-MM-DD-slug.md
```

Required frontmatter:

```yaml
---
title: "Titre de l'article"
date: YYYY-MM-DD
tags: [tag1, tag2]
format: experiment   # experiment | showcase | guide — see Writing style
description: "Résumé en une phrase (50 à 160 caractères)."
---
```

Permalink format is `/:year/:month/:day/:slug/` (`config/_default/hugo.toml`). The slug is derived from the title, so **changing the title of a published post changes its URL**: add the old path to `aliases` in the same edit, otherwise existing links break. Check the live URL before renaming.

When making a significant update to an existing post (content change, not a typo fix), add or update the `lastmod` field in the frontmatter:

```yaml
lastmod: YYYY-MM-DD
```

Without this field, Hugo falls back to the file's modification date, which changes on every `git checkout` and produces an unreliable freshness signal in the sitemap.

### Feature image

Not every post has a feature image, and that is expected: Blowfish simply shows no hero when there is none.

- Set one only when a real photo or screenshot of the project represents the post. Never use stock photos, generated images or decorative pictures to fill the slot.
- When there is one, set both fields: `featureimage` (hero at the top of the post, plus `featureimagecaption` describing what the reader sees) and `images` (the picture used when the post is shared, read by Hugo's Open Graph template).
- Local images go in `assets/images/<slug>/` so Blowfish can resize them; `featureimage` takes the path without a leading slash (`images/<slug>/file.png`), `images` with one (`/images/<slug>/file.png`). A remote URL works for both.
- Do not repeat the feature image in the body of the post.

Posts without a feature image use the site default (the author photo) when shared.

## Shortcodes

Use Blowfish shortcodes where they serve the reader, not to decorate. Full list and parameters: [shortcodes](https://blowfish.page/docs/shortcodes/).

| Shortcode | Use it for |
|---|---|
| `lead` | The first paragraph of every post |
| `alert` | Prerequisites, warnings, security limits, deprecated tools (`icon="list-check"`, `"check"`, default triangle) |
| `tabs` / `tab` | Alternatives the reader picks from (two domains, two OSes, several probes) |
| `mermaid` | Diagrams; never ASCII art |
| `figure` | Screenshots and photos in the body, with a `caption` that says what to look at |
| `github` | Repository card for the project the post is about |
| `icon` | Inline brand or UI icons (`{{< icon "github" >}}`), from the theme's icon set |

Inside a shortcode, Markdown reference links (`[text][ref]`) do not resolve: use inline links there.

## Before writing a new post

Before drafting any article, always ask the user for:

1. **Date** — publication date in `YYYY-MM-DD` format (used in both the filename and the frontmatter).
2. **Tags** — propose a list based on existing tags already used across posts. Prefer reusing existing tags over creating new ones; only propose new tags when no existing tag covers the concept. When a new tag is created, add `content/tags/<slug>/_index.md` with its display name as `title` (e.g. `Claude Code`, `BME280`), otherwise Hugo shows the capitalized slug.
3. **Description** — write a one-sentence summary (50-160 characters) for the `description` frontmatter field. Used for the meta description, Open Graph, Twitter cards, post cards and search.
4. **Series** — ask whether the post belongs to an existing series or starts a new one (see the Series section below).
5. **Format** — propose `experiment`, `showcase` or `guide` based on the topic (see Writing style) and set it in the `format` frontmatter field. The format drives structure, tone and length.
6. **Feature image** — ask whether a real photo or screenshot represents the post (see Feature image). No image is a valid answer.

After publishing a new post, check `static/robots.txt`: if you added a path that should be blocked from indexing (e.g. a draft or admin page), add a `Disallow` rule. Standard posts need no change.

## Series

A series is a group of thematically linked articles published under a shared title. Blowfish shows a series box at the top of each post in the series, listing the other parts.

### Adding an article to a series

Add two fields to the post frontmatter:

```yaml
series: ["kubernetes-migration"]
series_order: 1
```

`series` is an array containing the series name (one series per post). `series_order` is the article's position in the series (integer); Blowfish uses it to order the series box.

### Creating a new series

No registration step required. Adding `series: ["my-new-series"]` to a post frontmatter is enough — Hugo creates the taxonomy page at `/series/my-new-series/` automatically.

To add a title and description to the series index page, create `content/series/my-new-series/_index.md`:

```yaml
---
title: "Titre lisible de la série"
description: "Une phrase qui décrit la série."
---

Un paragraphe qui présente la série.
```

## External links

Use standard Markdown links. Blowfish opens external links in a new tab (`article.externalLinkForceNewTab`), so no HTML `<a target>` is needed. Prefer reference-style links with the definitions at the bottom of the post, except inside shortcodes (see Shortcodes).

## Language

The blog is written in French only. There are no translations and no `translationKey`. The English versions were removed in November 2026; their URLs, and the former `/fr/...` URLs, are kept as `aliases` on the French posts so old links redirect.

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
series_order: <integer matching release order, 7.0=1>
tags: [php, ...]
```

The `series_order` field follows the chronological release order across the full series (7.0 = part 1, 7.1 = part 2, etc.).

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
series_order: <integer matching release order, 3.3=1>
tags: [symfony, ...]
```

The `series_order` field follows the chronological release order across the full series starting from the first article (3.3 = part 1).

## Writing style

Every post belongs to one of three types: **experiment**, **showcase** or **guide**. Decide the type before drafting (propose one, confirm with the user if unclear). The type sets the structure, the length and how much of the story goes into the text. The **Voice** rules apply to all three.

The common thread: these are posts by someone who built and ran the thing, not documentation and not marketing. The reader should feel that a specific person wrote it.

### Voice (all types)

- First person, direct, casual. Write like explaining to a curious colleague over coffee, not like writing a README.
- Address the reader as "vous" — never "tu", "ton", "ta", "toi", "t'". "On" is fine for general statements, not as a way to dodge addressing the reader.
- Give opinions in the body, not only at the end. If something is bad, awkward or overrated, say so and say why.
- Specifics beat adjectives: the time lost, the exact error message, the number, the hardware model. "It took a while" is weaker than "three evenings".
- Vary sentence length. A long explanatory sentence followed by a short blunt one creates pacing. Avoid stacks of three uniform paragraphs in a row — they read as documentation regardless of pronoun choice.
- Humor only when it comes from the situation itself. No forced jokes, no decorative emoji in headers.
- Avoid: « Dans cet article, nous allons… », « Comme vous pouvez le voir… », bullet-point summaries of what was just said, superlatives (« révolutionnaire », « puissant », « game changer »).

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
- Topic headers and numbered steps are allowed here — and only here. Blowfish shows a table of contents on every post; never write one by hand.
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

Run this review whenever the user asks for one, and on every draft before presenting it as finished.

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
- [ ] Filler intensifiers: « vraiment », « simplement », « essentiellement », « juste », « en fait » — delete unless they change the meaning.

### 4. Specificity and voice

- [ ] Every claim of difficulty or effort comes with a concrete detail (duration, error message, number, hardware).
- [ ] Sentences that could appear in any tech blog are cut or made specific.
- [ ] At least one personal opinion appears in the body of the post.
- [ ] The register stays consistent; a guide may become neutral inside its steps, but not elsewhere.

### 5. Reader address

- [ ] "Vous" throughout. `grep -nwE "tu|ton|ta|tes|toi" <file>` and check for `t'` before a verb.

### 6. French

The text must read as written in French, not translated from English. Read it aloud: a sentence you wouldn't say to a French colleague gets rewritten.

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
- [ ] Every image has alt text describing what the reader should see; body screenshots use `figure` with a caption.

### 8. Frontmatter

- [ ] `format` is set. `featureimage`, `featureimagecaption` and `images` are all set, or none of them (see Feature image).
- [ ] If the title changed on a published post, the old URL is in `aliases`.
- [ ] `description` is 50–160 characters and matches the type (a guide says it's a guide).
- [ ] `lastmod` is updated if the content changed significantly.
- [ ] Tags reuse existing ones.
