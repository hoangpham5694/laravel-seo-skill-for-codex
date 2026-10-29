---
name: laravel-seo
summary: Implement, review, refactor, and test technical SEO for server-rendered Laravel Blade applications using typed SeoData, SeoFactory, Blade components, canonical URLs, Open Graph, Twitter/X cards, JSON-LD, hreflang, robots, sitemap guidance, and automated tests.
description: Use this skill when working on SEO metadata or technical SEO in a Laravel Blade project. It covers architecture, implementation, migration from ad-hoc $meta arrays, structured data, canonical rules, pagination, environment indexing policy, reverse-proxy URL correctness, and test coverage. Prefer native Laravel code over third-party SEO packages unless the project clearly needs CMS-level SEO features.
---

# Laravel SEO Skill

Use this skill to design, implement, review, or refactor SEO for **server-rendered Laravel + Blade** applications.

The default architecture is:

```text
Route
  -> Controller loads business data
  -> SeoFactory creates SeoData
  -> View receives $seo
  -> <x-seo :seo="$seo" /> renders <head>
  -> HTML response contains SEO metadata without client-side JavaScript
```

## When to use

Apply this skill when the task involves one or more of:

- `<title>` or meta description
- canonical URLs
- Open Graph metadata
- Twitter/X cards
- `robots` meta policy
- JSON-LD structured data
- article/product/organization schema
- SEO for pagination, search, filters, categories, products, or posts
- `hreflang`
- sitemap or `robots.txt`
- SEO behavior in staging/production
- duplicate metadata
- SEO regression tests
- migrating an existing Laravel project from per-controller `$meta` arrays

Do not use this skill for SPA-only SEO where metadata is rendered exclusively by a JavaScript framework unless Laravel is also responsible for the server-rendered document head.

## Required design rules

1. Prefer a typed `SeoData` object over associative `$meta` arrays for medium/large projects.
2. Put page-specific SEO construction in `SeoFactory` or equivalent domain-specific builders, not directly in Blade.
3. Keep controllers focused on business data; they may select which SEO factory method to call.
4. Use a single Blade component or partial as the only renderer for page-level SEO metadata.
5. Use `canonical` as the domain term instead of a generic `url` field.
6. Generate canonical URLs with named routes where possible.
7. `og:url` should normally equal canonical unless the product has an explicit reason for a different value.
8. Open Graph image URLs must be absolute and publicly accessible.
9. Never emit internal filesystem paths such as `storage/app/...` as social images.
10. Never build JSON-LD through manual string concatenation; serialize arrays/objects safely.
11. Keep staging/dev indexing policy explicit via SEO configuration such as `SEO_ROBOTS=noindex,nofollow`.
12. Do not automatically strip every query parameter. Preserve query parameters that materially change page content.
13. Pagination pages with distinct content should normally self-canonicalize, e.g. page 2 canonical -> page 2.
14. Tracking parameters should normally be excluded from canonical URLs.
15. Search/filter pages require an explicit indexing policy; do not hard-code `search == noindex` as a universal rule.
16. Render metadata server-side and verify it in **View Page Source**, not only the browser DOM after JavaScript runs.
17. Blade output must remain escaped with `{{ ... }}` for metadata values. Do not use raw `{!! !!}` for ordinary meta values.
18. If description content originates from rich HTML, normalize it to plain text before using it as metadata.
19. Add automated feature tests for important metadata and canonical behavior.
20. Preserve existing project conventions when they are already correct; do not create duplicate routes, layouts, or metadata renderers.

## Default implementation structure

For a project that needs reusable SEO infrastructure, prefer:

```text
config/seo.php
app/SEO/SeoData.php
app/SEO/SeoFactory.php
app/SEO/SeoText.php
app/SEO/Schema/ArticleSchema.php
app/SEO/Schema/ProductSchema.php          # when relevant
app/SEO/Schema/OrganizationSchema.php     # when relevant
app/View/Components/Seo.php
resources/views/components/seo.blade.php
```

For a very small site, simplify the number of classes, but keep one consistent SEO contract and one rendering location.

## Workflow

### 1. Inspect the existing project before editing

Find and review:

- Laravel/PHP version
- `routes/web.php` or the active web route files
- base Blade layout(s)
- existing `<title>`, description, canonical, Open Graph, Twitter, JSON-LD
- existing SEO package/service/component
- controllers/models for the affected pages
- slug and route-model-binding conventions
- storage/CDN image URL behavior
- proxy/trusted-proxy configuration if generated URLs have incorrect host or scheme
- current tests

Search for duplicate metadata before adding a new renderer.

### 2. Choose the smallest suitable architecture

Use this decision rule:

```text
Small site, few static pages
  -> one SEO Blade component + small SeoData/DTO or equivalent

Multiple content types / production application
  -> SeoData + SeoFactory + Blade component

CMS-level editing, large sitemap, multilingual SEO, automatic OG images,
advanced schema authoring, or many SEO-managed models
  -> evaluate a maintained SEO package before custom-building everything
```

Do not introduce a package only to render title/description/canonical/OG.

### 3. Build the SEO contract

The preferred `SeoData` fields are:

```text
title
description
canonical
image
type
robots
locale
siteName
imageAlt
imageWidth
imageHeight
twitterCard
twitterSite
alternates
schemas
```

Fields may be reduced if the project does not need them.

### 4. Build metadata from page data

Examples:

- Home: configured site title/description/default image/home route.
- Article: post title, normalized excerpt/body, public article image, article route, `type=article`, Article JSON-LD.
- Product: product name/description/image/product canonical, Product schema when business requirements justify it.
- Category: unique category metadata and category canonical.
- Pagination: include the page query in canonical when each page is distinct.
- Search/filter: define robots and canonical based on product SEO policy rather than a blanket rule.

Use route model binding when it simplifies slug routes.

### 5. Render exactly once in `<head>`

Preferred usage:

```blade
<x-seo :seo="$seo" />
```

The component should render, where applicable:

- title
- description
- robots
- canonical
- Open Graph
- Twitter/X card
- hreflang alternates
- JSON-LD

Do not leave a second legacy title/meta block elsewhere in the layout.

### 6. Structured data

Choose schema based on actual page semantics, not merely the route name.

For article-like pages consider:

- `Article`
- `BlogPosting`
- `NewsArticle`

Useful properties include:

- `headline`
- `image`
- `datePublished`
- `dateModified`
- `author`

Do not invent values that are not present in the application data.

### 7. Environment/indexing safety

Production commonly uses:

```dotenv
SEO_ROBOTS=index,follow
```

Staging/dev commonly uses:

```dotenv
SEO_ROBOTS=noindex,nofollow
```

Prefer an explicit SEO variable over inferring everything from `APP_ENV`.

#### Per-page robots override

Default page robots must come from `config('seo.robots')`. SEO factory page builders must support `bool $noindex = false`:

- Omitted or `false`: preserve the configured robots value unchanged.
- Only when config is exactly `index,follow`, `true` changes robots to `noindex,follow`.
- Any other configured value remains unchanged for both boolean values. Use strict equality; do not normalize or reorder directives before comparing.
- The parameter expresses noindex intent; do not couple it to page types such as `$catalog` or sample-data states such as `reference_data`.
- Prefer a named argument (`noindex: true`) at call sites. Private pages may retain a separate `noindex,nofollow` policy.

Exclude all noindex pages from the sitemap. In environments that allow crawling, do not block these URLs in `robots.txt`: crawlers must be able to read their noindex directive. Preserve any existing environment-wide crawl-blocking policy.

Remember: meta robots and `robots.txt` serve different purposes. Do not claim they are interchangeable. See the implementation guide for the factory example and required test matrix.

### 8. Canonical rules

Use these defaults unless product requirements say otherwise:

```text
/                       -> /
/posts/foo              -> /posts/foo
/posts?page=2           -> /posts?page=2
/posts?page=1           -> /posts
/posts/foo?utm_source=x -> /posts/foo
```

Do not canonicalize genuinely different content to one URL just because the pages belong to the same category.

### 9. URL and image correctness

Verify generated canonical/OG URLs in the deployed environment.

`APP_URL` alone may not repair an incorrect scheme/host when an application is behind a reverse proxy. Check trusted proxies, forwarded headers, CDN/disk URL configuration, and actual rendered HTML.

For public-disk images, confirm the storage link exists when the project relies on it:

```sh
php artisan storage:link
```

### 10. Tests

Add feature/unit tests appropriate to the change. Cover important cases such as:

- page returns 200
- missing slug returns 404
- correct title
- correct description
- correct canonical
- correct OG image
- article schema presence
- pagination canonical
- fallback metadata
- HTML escaping
- robots matrix: `index,follow` changes only for `$noindex = true`; `index,nofollow`, `noindex,follow`, and `noindex,nofollow` stay unchanged for both boolean values; omitted `$noindex` preserves config
- exactly one robots meta tag with the effective value in server-rendered HTML
- noindex pages excluded from sitemap and crawlable in environments that allow crawling
- no duplicate metadata renderer

Prefer stable semantic assertions. Avoid brittle full-document snapshots unless the project already uses them.

## Review checklist

Before finishing an implementation or code review, verify:

- [ ] metadata is present in server-rendered source
- [ ] only one title/description/canonical set is emitted
- [ ] canonical uses the intended route and slug
- [ ] tracking parameters do not leak into canonical unintentionally
- [ ] pagination canonical is correct
- [ ] OG image is absolute/public
- [ ] description contains plain text, not editor markup
- [ ] dynamic metadata is escaped
- [ ] JSON-LD is valid JSON and matches visible content
- [ ] default robots comes from config; `noindex: true` changes only exact `index,follow` to `noindex,follow`
- [ ] server-rendered HTML contains exactly one robots meta tag with the effective value
- [ ] noindex URLs are excluded from sitemap and remain crawlable where crawling is allowed
- [ ] staging/dev cannot be indexed when policy says noindex
- [ ] production does not accidentally inherit noindex
- [ ] reverse proxy does not generate HTTP or wrong-domain URLs
- [ ] sitemap/robots/hreflang requirements are addressed when relevant
- [ ] automated tests cover the important paths

## Package/library decision

Prefer native Laravel + the architecture above when requirements are limited to metadata, canonical, social cards, and a manageable amount of JSON-LD.

Consider a maintained SEO package when the application needs several of these at once:

- admin-editable SEO fields across many models
- large/dynamic sitemap generation
- multilingual/hreflang management
- many structured-data types
- OG image generation
- SEO auditing or reusable model traits

If evaluating a package, inspect maintenance activity, supported Laravel/PHP versions, API surface, migration footprint, and lock-in before recommending it.

## Detailed reference

For full code samples, configuration, `SeoData`, `SeoFactory`, `SeoText`, schema examples, Blade component implementation, controllers, routes, sitemap/robots notes, hreflang, reverse-proxy guidance, and test examples, read:

- `references/implementation-guide.md`

Use that reference as the canonical implementation guide for this skill. Adapt names and fields to the actual repository instead of copying examples blindly.

## Output expectations

When implementing:

- make the smallest coherent set of code changes;
- describe any assumptions caused by missing project context;
- flag SEO policy choices that require product/business confirmation;
- do not claim success until tests or relevant verification have been performed when the environment allows it.

When reviewing only:

- distinguish correctness issues from optional improvements;
- prioritize indexation/canonical/duplicate metadata problems over cosmetic metadata enhancements;
- give concrete file-level or code-level recommendations.
