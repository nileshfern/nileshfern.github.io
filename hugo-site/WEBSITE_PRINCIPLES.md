# Architecture playbook

This file documents the decisions behind this site, for whoever (human or AI) maintains it next.

## Stack

Hugo (extended) + [Hugo Blox](https://hugoblox.com) `blox-tailwind` v0.10.0 (vendored under
`_vendor/`, Tailwind CSS v4 pipeline) + Pagefind (static search) + GitHub Actions → GitHub Pages.
The repo is `nileshfern.github.io`, a **user site**, so it deploys to the domain root rather than
a `/reponame/` subpath. It serves on the owner's own domain, `https://www.nileshfernando.com/`
(GitHub Pages custom domain — `static/CNAME` holds the domain, `baseURL` in `hugo.yaml` and in
`.github/workflows/deploy.yml` must both match it exactly, and DNS at the registrar points the
domain at GitHub Pages). The bare `nileshfern.github.io` address still works as a fallback if DNS
ever lapses. The legacy site's `/research/` page is now `/publication/` here — `aliases:
[/research/]` on `content/publication/_index.md` 301s the old path so existing citations/bookmarks
to it don't break; the legacy `/teaching/` and `/policy/` paths already match exactly and needed
no alias.

The vendored theme in this Hugo Blox release is a general-purpose block/section builder (hero,
resume blocks, FAQ, etc.), not the older academic-CV theme the original build brief assumed.
Rather than fight the theme's block system, this build uses the theme only for its shell
(`baseof.html`, `site_head.html`, the Tailwind CSS pipeline, the navbar/search-modal
infrastructure, favicon/OG-image generation) and writes fully custom, project-level templates for
everything academic-specific: the homepage, the Research list, and publication pages.
Project-level `layouts/` always win over the vendored theme's `layouts/`, per Hugo's lookup
order.

## Content model

Every publication (published article or working paper) is a directory under
`content/publication/<slug>/index.md`. **The directory name is the permanent URL slug — never
rename it.** `publication_types: [journal_article]` vs `[report]` drives which Research tab an
item appears under (see `layouts/publication/list.html`). There are no `talk/` or `software/`
sections because there was no content for them at build time; `UPDATING.md` explains how to add
one later if needed. (Note: the section/URL is still `publication` — only the nav label and
on-page heading read "Research", per the owner's request; renaming the URL itself wasn't asked
for and would touch every internal `GetPage`/`Section` lookup in the templates.)

`content/policy/index.md` and `content/data-code/index.md` are plain prose pages (like teaching)
mirroring content from the owner's legacy site verbatim — content, ordering, and links preserved
rather than paraphrased, per his request.

`data/research_areas.json` is the single source of truth for the five research-area labels and
descriptions shown in the homepage accordions and the Research page's sidebar filter. A
publication opts into an area via its own `research_areas: [slug, ...]` front matter — there's no
reverse mapping to maintain.

There is no standalone Bio, C.V., Contact, or People page — the owner asked for the first three to
be folded into a single short bio + contact email in the homepage hero, and for the fourth to be
removed as a page but its function (co-author names linking out) kept, inline, on the Research
page and on each publication's own page instead. `layouts/_partials/authors-line.html` renders
that: it looks up every author except the owner in `data/coauthors.json` (personal site >
university page > LinkedIn) and links the ones it finds; call it wherever an author byline
renders (`publication/single.html`, `publication/list.html`, and the homepage's research-area
preview rows) rather than re-implementing the lookup inline — see `UPDATING.md` for the standing
warning that those links are auto-resolved, not hand-verified.

## "See Also" (`layouts/_partials/related_finder.html`)

Runs at build time on every publication page: explicit `related_papers:` overrides always win;
then same-`research_areas` siblings; then a scoring backfill (title-token overlap, shared
authors, shared tags) with a relaxed threshold if fewer than 3 results were found. Results are
deduplicated by normalized title before rendering and capped at 8.

**Gotcha worth remembering:** Hugo's `Scratch.Get("parent.child")` dotted-path syntax does *not*
reliably read back a value written with `Scratch.SetInMap "parent" "child" value` in this Hugo
version — it silently returns nil, which looks exactly like "not seen yet" and defeats any dedup
gate built on that pattern. `related_finder.html` uses **flat scratch keys** (`$scratch.Get $key`
/ `$scratch.Set $key true`) instead. If you add a new dedup pass, follow the same flat-key
pattern.

## JSON-LD and `<script type="application/ld+json">`

Hugo's `html/template` autoescaper treats content inside `<script>` as JS and will **double-escape
an already-`jsonify`'d string** if you splice it in with a bare `{{ ... | jsonify }}` — the output
looks like `"headline":"\"Title\""` instead of `"headline":"Title"`, which breaks JSON parsing.
Fix: build the whole object as a `dict`, run `jsonify` on the *entire* dict once, and mark the
single result `| safeJS` before emitting it. See
`layouts/_partials/hooks/head-end/mysite-and-scholar.html` for the pattern; don't hand-assemble
JSON-LD as literal `{ "key": {{ ... }} }` text with interpolated `jsonify` fragments.

## Scholarly / AI-visibility metadata

`layouts/_partials/hooks/head-end/mysite-and-scholar.html` is a **head-end hook** (auto-included
on every page via Hugo Blox's `get_hook` mechanism — no template override needed). It emits, on
every publication page regardless of the mysite settings below: Google Scholar `citation_*` meta
tags and a `ScholarlyArticle` JSON-LD block. Separately, gated behind `params.mysite.discovery`
(currently `false`, per the owner's information form), it would also emit an invisible
`generator` meta tag and a homepage `Person` JSON-LD block — both currently suppressed.

`params.mysite.credit` (currently `false`) gates the "Created using GaryKing.org/mysite" footer
line in `layouts/_partials/site_footer.html`.

## Spam-proof email (`layouts/_partials/email-link.html`)

The contact email is base64-encoded at build time (`{{ printf "%s@%s" .user .domain |
base64Encode }}`) into a `data-e` attribute; a small script decodes it and builds the real
`mailto:` link only in the visitor's browser. Deliberately **not** a lighter "name [at] domain"
text obfuscation — that pattern is common enough that scrapers routinely reverse it, so the raw
page source should contain no recognizable fragment of the address at all (no `@`, no username,
no domain/TLD substring), not just a disguised one. Both the `{{< email >}}` shortcode (for
Markdown pages) and the homepage template call this same partial — don't duplicate the logic.

## Taxonomies

Hugo's built-in `tags`/`categories` taxonomy pages are disabled (`disableKinds: [taxonomy, term]`
in `hugo.yaml`). `tags:` front matter is still used — just as plain data read directly by
`related_finder.html` for scoring — but no `/tags/...` pages are generated, since nothing in the
site links to them and they'd otherwise be unstyled orphan pages outside the site's actual
information architecture (Research Areas, not raw tags, is the browsing structure here).

## Local environment note (not relevant to CI)

If you build this locally on Apple Silicon under a Rosetta-translated (Intel) Homebrew `hugo`,
the Tailwind CSS v4 native bindings (`@tailwindcss/oxide`, `lightningcss`) may need their
`-darwin-x64` optional variants force-installed into `node_modules` alongside the `-darwin-arm64`
ones npm picks by default (`npm install <pkg>-darwin-x64@<version> --force --no-save`). This is a
local toolchain quirk only — GitHub Actions runs on Linux and resolves the correct platform
binaries on its own via `npm ci`.
