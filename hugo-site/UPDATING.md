# Updating this site

This site is built with [Hugo](https://gohugo.io) and deploys automatically to GitHub Pages
every time you push to `main`. You do not need to run anything locally to publish a change —
edit a file on GitHub, commit, and the site rebuilds itself in a couple of minutes.

## Adding a new publication or working paper

1. In `content/publication/`, create a new folder named with a short, permanent, kebab-case slug
   (e.g. `content/publication/my-new-paper/`). **Never rename this folder later** — the folder
   name becomes the paper's permanent URL.
2. Inside it, create `index.md` with front matter like this:

   ```yaml
   ---
   title: "Full Paper Title"
   date: 2026-01-01
   authors: ["A. Nilesh Fernando", "Co-Author Name"]
   publication_types: ["journal_article"]   # or ["report"] for a working paper
   publication: "Journal Name, Vol. X, Issue Y"
   abstract: "One paragraph."
   research_areas: ["labor-markets-migration"]   # pick from data/research_areas.json
   tags: ["a few keywords"]
   links:
     - type: pdf
       url: "files/my-new-paper.pdf"
     - type: doi
       url: "https://doi.org/..."
   ---
   ```
3. Put the PDF in `static/files/` with a matching filename.
4. Commit and push. The paper appears on the Research page, the relevant Research Area accordion
   on the homepage, and gets "See Also" links automatically — you don't need to do anything else.

## Adding a talk, dataset, or software project

The site doesn't currently have dedicated sections for these (there wasn't any content for them
at launch). To add one:
- A **dataset** tied to a paper: add `dataverse_url` and `related_paper: <slug-of-the-paper>` to
  that paper's front matter — a "Replication data for..." banner will appear on the paper's page.
- A **talk or software project**: ask your AI assistant to add a `content/talk/` or
  `content/software/` section following the same pattern as `content/publication/`.

## Updating your bio, contact email, or teaching info

There's no separate Bio or Contact page — the short bio, contact email, C.V., and research
statement all live in the homepage hero, in `layouts/landing/list.html`. It's plain HTML rather
than Markdown front matter (because the bio text has inline links to J-PAL/BREAD and the email
needs to stay obfuscated — see below), so ask your AI assistant to edit that file's `<p
class="gk-hero-intro">` and `<p class="gk-hero-contact">` lines rather than editing it by hand.

Teaching info is a normal Markdown page: `content/teaching/index.md`.

To change the contact email, don't just paste the new address in — it needs to stay run through
the `email-link.html` partial (`{{ partial "email-link.html" (dict "user" "..." "domain" "...")
}}`) so it stays base64-encoded in the page source rather than sitting there as plain, harvestable
text. Same partial/shortcode (`{{< email user="..." domain="..." >}}`) is available for use in any
Markdown page too.

## Co-author links on the Research page

Author names in citation rows link to each co-author's best external page (personal site >
university page > LinkedIn), resolved in `data/coauthors.json`. **These links were found by an
automated web search, not verified by hand** — please check that every name points to the right
person. To fix a wrong link, replace its `url`; to add a link for someone currently unlinked (like
Layton Hall), add a new entry keyed by their exact name as it appears in your papers' `authors:`
fields; to remove a link, delete their entry.

## Adding or renaming a Research Area

Edit `data/research_areas.json`. Each area needs a `slug`, `name`, and one-line `description`.
Reference the `slug` in a publication's `research_areas:` front matter field to file it under
that area.

## Updating Policy & Impact or Data & Code

Both are plain Markdown pages: `content/policy/index.md` and `content/data-code/index.md`. Add a
new numbered/bulleted entry the same way the existing ones are written.

## Local preview (optional)

If you want to preview changes on your own computer before pushing:

```bash
cd hugo-site
hugo server
```

This requires Hugo (extended) and Node.js installed locally. Everything else (search, PDFs,
citation metadata) is generated automatically at deploy time and doesn't need any manual steps.
