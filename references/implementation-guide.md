# Implementing SEO metadata in a Laravel project

This guide is for Laravel projects that use Blade to render HTML on the server. The goal is to build a structured, testable, extensible SEO layer instead of defining separate metadata arrays in each controller.

The examples target Laravel 13.x (PHP 8.3+) but mostly use common Laravel APIs (`route()`, `asset()`, `Storage`, Blade components, and route model binding), so they can be applied to Laravel 12 with very few changes. When integrating with a project running another version, consult its release notes and upgrade guide.

> Scope: title, meta description, canonical, robots, Open Graph, Twitter/X Card, JSON-LD structured data, hreflang, sitemap/robots.txt, pagination, reverse proxies, and automated tests.

---

## 1. Proposed architecture

Avoid having each controller build its own `$meta` array. Recommended architecture:

```text
Route
  → Controller retrieves business data
  → SeoFactory creates SeoData
  → View receives $seo
  → <x-seo :seo="$seo" /> inside <head>
  → Server returns HTML containing all title/meta/canonical/OG/Twitter/JSON-LD metadata
```

Suggested file structure:

```text
config/seo.php
app/SEO/SeoData.php
app/SEO/SeoFactory.php
app/SEO/SeoText.php
app/SEO/Schema/ArticleSchema.php
app/SEO/Schema/ProductSchema.php          # if the site has products
app/SEO/Schema/OrganizationSchema.php     # if site-wide schema is needed
app/View/Components/Seo.php
resources/views/components/seo.blade.php
resources/views/layouts/app.blade.php
routes/web.php
app/Http/Controllers/HomeController.php
app/Http/Controllers/PostController.php
```

For a small project, you can simplify `SeoFactory` or the schema classes, but keep `SeoData` to provide a clear contract and avoid misspelled keys.

---

## 2. SEO data contract

Create `app/SEO/SeoData.php`:

```php
<?php

namespace App\SEO;

final readonly class SeoData
{
    public function __construct(
        public string $title,
        public string $description,
        public string $canonical,
        public string $image,
        public string $type = 'website',
        public string $robots = 'index,follow',
        public ?string $locale = null,
        public ?string $siteName = null,
        public ?string $imageAlt = null,
        public ?int $imageWidth = null,
        public ?int $imageHeight = null,
        public string $twitterCard = 'summary_large_image',
        public ?string $twitterSite = null,
        public array $alternates = [],
        public array $schemas = [],
    ) {
    }
}
```

### Why use an object instead of an array

An object provides:

- protection against typos such as `desciption` and `canonial`;
- better IDE support and autocomplete;
- easier unit testing;
- easier field extension;
- separation between controllers and rendering details.

Use the name `canonical` instead of `url` because this field represents the canonical URL of the page. Use the same canonical value for `og:url` unless there is a specific reason to use a different one.

---

## 3. Default configuration

Create `config/seo.php`:

```php
<?php

return [
    'site_name' => env('APP_NAME', 'Website'),

    'default_title' => env(
        'SEO_TITLE',
        env('APP_NAME', 'Website')
    ),

    'default_description' => env(
        'SEO_DESCRIPTION',
        'Default website description.'
    ),

    'default_image' => env(
        'SEO_IMAGE',
        'images/seo/default.jpg'
    ),

    'locale' => env('SEO_LOCALE', 'vi_VN'),

    'robots' => env('SEO_ROBOTS', 'index,follow'),

    'twitter' => [
        'card' => env('SEO_TWITTER_CARD', 'summary_large_image'),
        'site' => env('SEO_TWITTER_SITE'),
    ],
];
```

Production `.env`:

```dotenv
APP_NAME="Business name"
APP_URL=https://www.example.com
SEO_TITLE="Business name"
SEO_DESCRIPTION="Default website description."
SEO_IMAGE=images/seo/default.jpg
SEO_LOCALE=vi_VN
SEO_ROBOTS=index,follow
SEO_TWITTER_CARD=summary_large_image
# SEO_TWITTER_SITE=@example
```

Staging/development `.env`:

```dotenv
SEO_ROBOTS=noindex,nofollow
```

Do not rely entirely on `APP_ENV` to decide between index and noindex. A dedicated SEO configuration variable helps prevent staging from being indexed when the environment configuration is unexpected.

Place the default image at:

```text
public/images/seo/default.jpg
```

`asset()` only generates a URL; it does not check whether the file exists. After deployment, verify that the actual image URL returns `200` and is publicly accessible without authentication.

---

## 4. Normalizing text for SEO

Create `app/SEO/SeoText.php`:

```php
<?php

namespace App\SEO;

use Illuminate\Support\Str;

final class SeoText
{
    public static function description(
        ?string $html,
        int $limit = 160
    ): string {
        $text = html_entity_decode(
            (string) $html,
            ENT_QUOTES | ENT_HTML5,
            'UTF-8'
        );

        $text = Str::squish(strip_tags($text));

        return Str::limit($text, $limit);
    }
}
```

`160` is a reasonable editorial limit, not a mandatory Google limit. If the description is empty after processing, `SeoFactory` should use the default fallback.

---

## 5. Structured Data / JSON-LD

Structured data helps search engines understand the type of content on a page. For posts, use `Article`, `BlogPosting`, or `NewsArticle` as appropriate for the content.

Create `app/SEO/Schema/ArticleSchema.php`:

```php
<?php

namespace App\SEO\Schema;

use App\Models\Post;

final class ArticleSchema
{
    public static function make(Post $post, string $image): array
    {
        $schema = [
            '@context' => 'https://schema.org',
            '@type' => 'Article',
            'headline' => $post->title,
            'image' => [$image],
            'datePublished' => $post->published_at?->toIso8601String(),
            'dateModified' => $post->updated_at?->toIso8601String(),
        ];

        if ($post->relationLoaded('author') && $post->author) {
            $schema['author'] = [
                '@type' => 'Person',
                'name' => $post->author->name,
            ];

            if (! empty($post->author->url)) {
                $schema['author']['url'] = $post->author->url;
            }
        }

        return array_filter(
            $schema,
            static fn ($value) => $value !== null
        );
    }
}
```

Google does not require every property, but you should provide relevant properties such as `headline`, `image`, `datePublished`, `dateModified`, and `author`.

If article images are available in multiple aspect ratios, you can provide 1:1, 4:3, and 16:9 versions to better support rich results.

Do not build JSON-LD by manually concatenating strings.

---

## 6. SeoFactory

Create `app/SEO/SeoFactory.php`:

```php
<?php

namespace App\SEO;

use App\Models\Post;
use App\SEO\Schema\ArticleSchema;
use Illuminate\Support\Facades\Storage;

final class SeoFactory
{
    public static function page(
        string $title,
        string $description,
        string $canonical,
        ?string $image = null,
        bool $noindex = false,
    ): SeoData {
        $robots = config('seo.robots');

        if ($robots === 'index,follow' && $noindex) {
            $robots = 'noindex,follow';
        }

        return new SeoData(
            title: $title,
            description: $description,
            canonical: $canonical,
            image: $image ?? asset(config('seo.default_image')),
            robots: $robots,
            locale: config('seo.locale'),
            siteName: config('seo.site_name'),
            imageAlt: $title,
            twitterCard: config('seo.twitter.card'),
            twitterSite: config('seo.twitter.site'),
        );
    }

    public function default(?string $canonical = null): SeoData
    {
        $image = asset(config('seo.default_image'));

        return new SeoData(
            title: config('seo.default_title'),
            description: config('seo.default_description'),
            canonical: $canonical ?? url()->current(),
            image: $image,
            type: 'website',
            robots: config('seo.robots'),
            locale: config('seo.locale'),
            siteName: config('seo.site_name'),
            imageAlt: config('seo.default_title'),
            twitterCard: config('seo.twitter.card'),
            twitterSite: config('seo.twitter.site'),
        );
    }

    public function home(): SeoData
    {
        $default = $this->default(route('home'));

        return new SeoData(
            title: 'Home - ' . config('seo.site_name'),
            description: 'An introduction to our main services and customer benefits.',
            canonical: route('home'),
            image: $default->image,
            type: 'website',
            robots: $default->robots,
            locale: $default->locale,
            siteName: $default->siteName,
            imageAlt: $default->imageAlt,
            twitterCard: $default->twitterCard,
            twitterSite: $default->twitterSite,
        );
    }

    public function postsIndex(int $page = 1): SeoData
    {
        $canonical = route(
            'posts.index',
            $page > 1 ? ['page' => $page] : []
        );

        $default = $this->default($canonical);

        return new SeoData(
            title: 'Posts'
                . ($page > 1 ? ' - Page ' . $page : '')
                . ' - ' . config('seo.site_name'),
            description: 'The latest posts and updates from our business.',
            canonical: $canonical,
            image: $default->image,
            type: 'website',
            robots: $default->robots,
            locale: $default->locale,
            siteName: $default->siteName,
            imageAlt: 'Posts - ' . config('seo.site_name'),
            twitterCard: $default->twitterCard,
            twitterSite: $default->twitterSite,
        );
    }

    public function forPost(Post $post): SeoData
    {
        $description = SeoText::description($post->description);

        if ($description === '') {
            $description = config('seo.default_description');
        }

        $image = $post->image
            ? url(Storage::disk('public')->url($post->image))
            : asset(config('seo.default_image'));

        $canonical = route('posts.show', $post);

        return new SeoData(
            title: $post->title . ' - ' . config('seo.site_name'),
            description: $description,
            canonical: $canonical,
            image: $image,
            type: 'article',
            robots: config('seo.robots'),
            locale: config('seo.locale'),
            siteName: config('seo.site_name'),
            imageAlt: $post->title,
            twitterCard: config('seo.twitter.card'),
            twitterSite: config('seo.twitter.site'),
            schemas: [
                ArticleSchema::make($post, $image),
            ],
        );
    }
}
```

### Per-page robots policy

Page robots default to `config('seo.robots')`. The `page()` builder supports `bool $noindex = false`; apply the same contract to other page builders when exposing this override. Omission or `false` preserves config unchanged. Only when config is exactly `index,follow` does `true` change robots to `noindex,follow`. Every other configured value remains unchanged regardless of `$noindex`. Use strict equality without normalizing whitespace, case, or directive order.

The parameter expresses noindex intent. Do not couple the override to page types such as `$catalog` or sample-data states such as `reference_data`. Private pages may continue to use a separate `noindex,nofollow` policy.

Prefer named arguments to make the caller's intent clear (adapt the route to the project):

```php
$seo = SeoFactory::page(
    title: 'Shop RX',
    description: 'Explore our products.',
    canonical: route('user.shop'),
    noindex: true,
);
```

Required test matrix:

| Config | `$noindex` | Resulting robots |
| --- | --- | --- |
| `index,follow` | `false` | `index,follow` |
| `index,follow` | `true` | `noindex,follow` |
| `index,nofollow` | `false` | `index,nofollow` |
| `index,nofollow` | `true` | `index,nofollow` |
| `noindex,follow` | `false` | `noindex,follow` |
| `noindex,follow` | `true` | `noindex,follow` |
| `noindex,nofollow` | `false` | `noindex,nofollow` |
| `noindex,nofollow` | `true` | `noindex,nofollow` |

Also test omitted `$noindex` against the configured values. Cover a non-exact value such as `index, follow` to verify it is preserved. Feature tests must verify the effective robots value in server-rendered HTML and exactly one `meta[name="robots"]` element in the document.

If you use S3/CDN or the model already stores an absolute URL, replace the `$image` generation logic with the public URL provided by your system. Do not put internal paths such as `storage/app/...` in `og:image`, and do not use URLs requiring authentication or short-lived tokens for images intended for long-term sharing.

---

## 7. Route model binding

Replace manual slug queries with route model binding.

`routes/web.php`:

```php
<?php

use App\Http\Controllers\HomeController;
use App\Http\Controllers\PostController;
use Illuminate\Support\Facades\Route;

Route::get('/', [HomeController::class, 'index'])
    ->name('home');

Route::get('/bai-viet', [PostController::class, 'index'])
    ->name('posts.index');

Route::get('/bai-viet/{post:slug}', [PostController::class, 'show'])
    ->name('posts.show');
```

Laravel automatically returns `404` if the corresponding model is not found.

You can replace `{post:slug}` with `{post}` if the model declares:

```php
public function getRouteKeyName(): string
{
    return 'slug';
}
```

---

## 8. Controller

`HomeController`:

```php
<?php

namespace App\Http\Controllers;

use App\SEO\SeoFactory;

class HomeController extends Controller
{
    public function index(SeoFactory $seoFactory)
    {
        return view('home', [
            'seo' => $seoFactory->home(),
        ]);
    }
}
```

`PostController`:

```php
<?php

namespace App\Http\Controllers;

use App\Models\Post;
use App\SEO\SeoFactory;

class PostController extends Controller
{
    public function index(SeoFactory $seoFactory)
    {
        $posts = Post::query()
            // Add a published condition based on the actual schema.
            ->orderByDesc('id')
            ->paginate(10);

        return view('posts.index', [
            'posts' => $posts,
            'seo' => $seoFactory->postsIndex(
                $posts->currentPage()
            ),
        ]);
    }

    public function show(Post $post, SeoFactory $seoFactory)
    {
        // If the schema uses an author, eager-load it before building SEO data.
        // $post->loadMissing('author');

        return view('posts.show', [
            'post' => $post,
            'seo' => $seoFactory->forPost($post),
        ]);
    }
}
```

If a page is accessible only when published or with appropriate permissions, route binding or the controller must apply the same scopes and permissions as the existing business logic. SEO changes must not accidentally allow access to drafts or private content.

---

## 9. SEO Blade component

Create `app/View/Components/Seo.php`:

```php
<?php

namespace App\View\Components;

use App\SEO\SeoData;
use Closure;
use Illuminate\Contracts\View\View;
use Illuminate\View\Component;

class Seo extends Component
{
    public function __construct(public SeoData $seo)
    {
    }

    public function render(): View|Closure|string
    {
        return view('components.seo');
    }
}
```

Create `resources/views/components/seo.blade.php`:

```blade
<title>{{ $seo->title }}</title>
<meta name="description" content="{{ $seo->description }}">
<meta name="robots" content="{{ $seo->robots }}">
<link rel="canonical" href="{{ $seo->canonical }}">

<meta property="og:title" content="{{ $seo->title }}">
<meta property="og:description" content="{{ $seo->description }}">
<meta property="og:type" content="{{ $seo->type }}">
<meta property="og:url" content="{{ $seo->canonical }}">
<meta property="og:image" content="{{ $seo->image }}">

@if ($seo->siteName)
    <meta property="og:site_name" content="{{ $seo->siteName }}">
@endif

@if ($seo->locale)
    <meta property="og:locale" content="{{ $seo->locale }}">
@endif

@if ($seo->imageAlt)
    <meta property="og:image:alt" content="{{ $seo->imageAlt }}">
@endif

@if ($seo->imageWidth)
    <meta property="og:image:width" content="{{ $seo->imageWidth }}">
@endif

@if ($seo->imageHeight)
    <meta property="og:image:height" content="{{ $seo->imageHeight }}">
@endif

<meta name="twitter:card" content="{{ $seo->twitterCard }}">
<meta name="twitter:title" content="{{ $seo->title }}">
<meta name="twitter:description" content="{{ $seo->description }}">
<meta name="twitter:image" content="{{ $seo->image }}">

@if ($seo->twitterSite)
    <meta name="twitter:site" content="{{ $seo->twitterSite }}">
@endif

@foreach ($seo->alternates as $hreflang => $url)
    <link
        rel="alternate"
        hreflang="{{ $hreflang }}"
        href="{{ $url }}"
    >
@endforeach

@foreach ($seo->schemas as $schema)
    <script type="application/ld+json">{!! json_encode(
        $schema,
        JSON_UNESCAPED_SLASHES
        | JSON_UNESCAPED_UNICODE
        | JSON_HEX_TAG
        | JSON_HEX_AMP
        | JSON_HEX_APOS
        | JSON_HEX_QUOT
    ) !!}</script>
@endforeach
```

Do not use `{!! ... !!}` for title, description, or OG metadata. JSON-LD requires raw JSON, but it must be encoded with `json_encode()` using the `JSON_HEX_*` flags shown above; do not insert user-supplied JSON directly.

Only declare `og:image:width`/`height` when the actual image dimensions are known. Do not hard-code `1200x630` if dynamically selected images may have different dimensions.

---

## 10. Layout

`resources/views/layouts/app.blade.php`:

```blade
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="csrf-token" content="{{ csrf_token() }}">

    <x-seo :seo="$seo ?? app(\App\SEO\SeoFactory::class)->default()" />

    @stack('styles')
</head>
<body>
    @yield('content')
    @stack('scripts')
</body>
</html>
```

To avoid calling the service container from Blade, you can require every controller to pass `$seo`. However, the fallback above is useful for existing pages during migration.

If the project already has `<title>`, meta description, OG, or canonical tags, remove them or move them into this component so that each page renders only one set of metadata.

---

## 11. Content views

`resources/views/home.blade.php`:

```blade
@extends('layouts.app')

@section('content')
    <h1>{{ config('seo.site_name') }}</h1>
@endsection
```

`resources/views/posts/index.blade.php`:

```blade
@extends('layouts.app')

@section('content')
    <h1>Posts</h1>

    @foreach ($posts as $post)
        <a href="{{ route('posts.show', $post) }}">
            {{ $post->title }}
        </a>
    @endforeach

    {{ $posts->links() }}
@endsection
```

`resources/views/posts/show.blade.php`:

```blade
@extends('layouts.app')

@section('content')
    <article>
        <h1>{{ $post->title }}</h1>
        {{-- Post content --}}
    </article>
@endsection
```

Child views should not add their own title or meta tags if the layout already uses `<x-seo>`.

---

## 12. Canonical URL

Guidelines:

| Page type | Recommended canonical |
| --- | --- |
| Home page | `route('home')` |
| Static page | Canonical route for the page |
| Post detail | Route containing the post slug |
| Product detail | Product detail route |
| Pagination page 1 | URL without `?page=1` |
| Pagination page 2+ | Self-canonical with `?page=N` |
| Tracking params | Canonical excludes `utm_*`, `fbclid`, and `gclid` if the content is unchanged |
| Search/filter | Follow product policy; do not automatically canonicalize everything to one URL |

Do not automatically remove all query strings. Some queries genuinely change the content.

`url()->current()` excludes the query string, making it suitable as a fallback. For pagination, search, and filters, the controller or SeoFactory must explicitly build the canonical URL.

Do not canonicalize page 2 to page 1 if page 2 contains a different set of content.

---

## 13. Pagination

The example in `SeoFactory::postsIndex()` uses:

```text
/bai-viet
/bai-viet?page=2
/bai-viet?page=3
```

and each page uses its own URL as the canonical.

You do not need to add `rel="next"`/`rel="prev"` solely for Google to handle pagination; focus on crawlable HTML links and appropriate canonical URLs.

---

## 14. Search and filter pages

Example for a search route:

```php
$seo = SeoFactory::page(
    title: 'Search: ' . $query,
    description: 'Search results for: ' . $query,
    canonical: route('search', ['query' => $query]),
    image: asset(config('seo.default_image')),
    noindex: true,
);
```

With config exactly `index,follow`, this example produces `noindex,follow`; all other configured values are preserved. `noindex,follow` is a common policy for internal search, not a mandatory rule. If a filter or search page serves as an SEO landing page with stable, valuable content, you can allow it to be indexed.

---

## 15. Hreflang for multilingual websites

If the website has only one language, skip this section.

If equivalent URLs exist:

```text
/vi/san-pham/abc
/en/products/abc
```

pass:

```php
alternates: [
    'vi' => route('products.show.vi', $product),
    'en' => route('products.show.en', $product),
    'x-default' => route('products.show.vi', $product),
],
```

The component will render:

```html
<link rel="alternate" hreflang="vi" href="...">
<link rel="alternate" hreflang="en" href="...">
<link rel="alternate" hreflang="x-default" href="...">
```

Alternate URLs should consistently reference one another across language versions.

---

## 16. Sitemap and robots.txt

Meta robots and `robots.txt` do not replace one another.

- `meta robots` controls index/follow at the page level;
- `robots.txt` primarily controls crawling;
- a sitemap lists important canonical URLs for discovery.

### Minimal robots.txt

Production `public/robots.txt`:

```text
User-agent: *
Allow: /

Sitemap: https://www.example.com/sitemap.xml
```

Staging can use:

```text
User-agent: *
Disallow: /
```

but you should still keep `SEO_ROBOTS=noindex,nofollow` as an additional safeguard. Do not rely solely on `robots.txt` to request removal of an already known URL from the index.

### Sitemap

You can create an XML route/controller yourself or use a package if the website has many resource types. The sitemap should contain only canonical, public, indexable URLs. Exclude every noindex page, including pages effectively changed to noindex by `noindex: true`, pages inheriting noindex from config, and private pages with a separate noindex policy. A `noindex: true` argument alone does not determine sitemap exclusion when config prevents the override. Use the effective page indexing policy when selecting sitemap entries so it stays consistent with rendered metadata.

In environments that allow crawling, do not disallow these noindex URLs in `robots.txt`; crawlers need to fetch them to read the noindex directive. Preserve any existing environment-wide crawl block, such as the staging policy above; it is separate from per-page noindex and can prevent crawlers from reading page metadata.

---

## 17. View Composer: when to use it

Laravel View Composers are appropriate for global data such as:

- site name;
- shared organization schema;
- default locale;
- navigation/footer data.

Using a View Composer to determine dynamic SEO for Post/Product pages is not recommended because dependencies are hidden and harder to trace than with `Controller → SeoFactory → View`.

---

## 18. When to use an SEO package

A custom implementation like the one in this guide is suitable when you need:

- title/description;
- canonical;
- OG/Twitter;
- JSON-LD;
- hreflang;
- a simple sitemap.

Consider a package if you have:

- a CMS with editable SEO fields in the admin interface;
- a very large sitemap or multiple sources;
- many complex JSON-LD types;
- automatic OG image generation;
- a large hreflang system;
- SEO audits or model traits used throughout the application.

Do not add a dependency just to replace a few simple meta tags.

---

## 19. Reverse proxies, HTTPS, and URL generation

`route()`, `asset()`, and URL generation depend on the host and scheme Laravel receives from the request or configuration. `APP_URL` does not automatically fix every issue when the app runs behind a load balancer or reverse proxy.

Check:

- trusted proxies;
- forwarded headers;
- HTTPS termination;
- CDN/storage disk URLs;
- config cache;
- the actual canonical and OG image URLs in **View Page Source**.

Check more than the DOM after JavaScript runs; SEO metadata must be present in the HTML returned by the server for a Blade SSR website like the one in this guide.

---

## 20. Images in storage

If post images are stored on the local `public` disk:

```sh
php artisan storage:link
```

Check:

```text
/public/storage/...  → public
/storage/app/public/... → do not use directly as a metadata URL
```

If you use S3/CDN, `og:image` and the Twitter image must use stable public URLs. Avoid short-lived pre-signed URLs unless there is a specific intentional reason.

---

## 21. Automated tests

SEO is prone to regressions because the UI often still looks normal. Add feature tests.

Example `tests/Feature/PostSeoTest.php`:

```php
<?php

namespace Tests\Feature;

use App\Models\Post;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class PostSeoTest extends TestCase
{
    use RefreshDatabase;

    public function test_post_page_contains_expected_seo_metadata(): void
    {
        $post = Post::factory()->create([
            'title' => 'Hello World',
            'slug' => 'hello-world',
            'description' => '<p>Hello <strong>SEO</strong></p>',
        ]);

        $response = $this->get(route('posts.show', $post));

        $response->assertOk();

        $response->assertSee(
            '<title>Hello World - ' . config('seo.site_name') . '</title>',
            false
        );

        $response->assertSee(
            '<link rel="canonical" href="' . route('posts.show', $post) . '">',
            false
        );

        $response->assertSee(
            '<meta property="og:type" content="article">',
            false
        );

        $response->assertSee(
            'application/ld+json',
            false
        );
    }
}
```

Include tests for:

- default fallback;
- canonical URLs using the correct slug;
- self-canonical URLs with `?page=2`;
- page 1 canonical URLs excluding `?page=1`;
- the per-page robots matrix in section 6, including omitted `$noindex` and preserving every config other than exact `index,follow`;
- exactly one robots meta tag with the effective value in server-rendered HTML;
- noindex on staging or with the corresponding configuration;
- noindex URLs absent from sitemap and not blocked by robots.txt in environments that allow crawling;
- HTML being stripped from descriptions;
- characters `&`, `'`, `"`, and Unicode being escaped correctly;
- correct JSON-LD encoding;
- fallback images;
- nonexistent pages returning 404;
- no duplicate title/meta/canonical tags being rendered.

---

## 22. Validation and external testing

After deployment:

```sh
php artisan route:list
php artisan config:cache
php artisan view:cache
```

If the project uses route caching:

```sh
php artisan route:cache
```

Recommended checks:

- **View Page Source**: metadata is present in the HTML returned by the server;
- canonical/og:url uses the correct HTTPS scheme and domain;
- the OG image is public and returns a successful status;
- Rich Results Test for JSON-LD;
- URL Inspection/Search Console when needed;
- the appropriate social sharing debugger for the platform if the preview is incorrect;
- the sitemap returns valid XML and contains only canonical URLs.

---

## 23. Acceptance checklist

- [ ] Home, listing, and detail pages render successfully.
- [ ] A nonexistent slug returns 404.
- [ ] Each page has exactly one `<title>`.
- [ ] Each page has exactly one meta description.
- [ ] Each page has exactly one canonical.
- [ ] `og:title`, `og:description`, `og:type`, `og:url`, and `og:image` are correct.
- [ ] Twitter/X Card metadata is correct.
- [ ] `og:site_name`, `og:locale`, and `og:image:alt` are correct when configured.
- [ ] `og:image:width/height` is rendered only when accurate data is available.
- [ ] Article pages have valid JSON-LD.
- [ ] Descriptions contain no HTML markup.
- [ ] Canonical and image URLs are absolute.
- [ ] Page 2 includes `?page=2`; page 1 excludes `?page=1`.
- [ ] Tracking queries do not change the canonical when the content is unchanged.
- [ ] Search/filter pages have a clear index/noindex policy.
- [ ] Omitted/false `$noindex` preserves config; true changes only exact `index,follow` to `noindex,follow`; all other configs remain unchanged.
- [ ] Server-rendered HTML contains exactly one robots meta tag with the effective value.
- [ ] Staging/development defaults to `noindex,nofollow` according to configuration; explicit per-page overrides follow section 6.
- [ ] `robots.txt` matches the environment.
- [ ] The sitemap contains only public, canonical, indexable URLs and excludes all noindex pages.
- [ ] Noindex URLs remain crawlable in environments that allow crawling.
- [ ] hreflang is correct for multilingual sites.
- [ ] There is no duplicate metadata from an old layout or plugin.
- [ ] The reverse proxy does not cause URLs to use HTTP or the wrong domain.
- [ ] SEO feature tests pass.

---

## 24. Common issues

| Symptom | Cause / resolution |
| --- | --- |
| `Route [...] not defined` | Check `php artisan route:list`; fix the route name |
| 404 with `{post:slug}` | The slug does not exist, route binding does not match, or a global scope excludes the model |
| Shared image does not load | Check the storage link, disk URL, CDN, permissions, and fallback |
| `og:image` is a relative URL | Use `asset()`, `url()`, or an absolute CDN URL |
| Detail page canonical points to the listing | Use the detail route with the model/slug |
| Page 2 canonical points to page 1 | Build a self-canonical URL for the current page |
| Incorrect domain/scheme | Check trusted proxies, forwarded headers, APP_URL, CDN, and config cache |
| Staging appears on Google | Incorrect `SEO_ROBOTS`, incorrect robots.txt, or staging was previously indexed |
| Invalid JSON-LD | Do not concatenate JSON manually; use `json_encode()` and validate with Rich Results Test |
| Duplicate title/meta | Remove the old block or the package/component rendering duplicate metadata |
| `.env` changes but output stays the same | Clear/rebuild the config cache according to the deployment process |
| Description contains HTML | Use `SeoText::description()` |
| Twitter preview differs from OG | Check `twitter:*` fields and the platform cache |

---

## 25. Final architecture decision

### Default recommendation

```text
Controller
  → SeoFactory
      → SeoData (typed DTO)
          → <x-seo>
              → title/meta/canonical
              → Open Graph
              → Twitter/X Card
              → hreflang
              → JSON-LD
```

### Not recommended as the primary architecture

```text
Controller builds its own $meta array in every action
```

because SEO logic is easily duplicated and is harder to refactor and type-check.

### Optional supporting tools

- View Composer: for global SEO data;
- SEO package: when CMS/schema/sitemap/hreflang requirements are complex;
- custom schema builders: for Product, BreadcrumbList, Organization, LocalBusiness, etc.

---

## 26. References

- Laravel 13 documentation / release notes: https://laravel.com/framework/docs/releases
- Laravel URL generation: https://laravel.com/docs/13.x/urls
- Laravel route model binding: https://laravel.com/docs/13.x/routing#route-model-binding
- Laravel Blade components: https://laravel.com/docs/13.x/blade#components
- Laravel views / view composers: https://laravel.com/docs/13.x/views
- Google canonical guidance: https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls
- Google Article structured data: https://developers.google.com/search/docs/appearance/structured-data/article
- Open Graph protocol: https://ogp.me/
- Schema.org: https://schema.org/

---

## 27. Migration notes for an existing implementation

If the current project already uses a `$meta` array:

1. create `SeoData`, `SeoFactory`, and `<x-seo>` first;
2. keep a fallback in the layout so existing pages continue to work;
3. migrate each controller to `SeoFactory`;
4. remove the old `$meta` partial once no pages use it;
5. run tests to detect duplicate metadata;
6. check the HTML source on staging before production.

For a large project, you do not need to migrate every page at once, as long as the layout renders only one SEO implementation per request.
