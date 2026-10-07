# Triển khai SEO metadata trong dự án Laravel

Hướng dẫn này dành cho dự án Laravel sử dụng Blade, theo mô hình server-rendered HTML. Mục tiêu là xây dựng một lớp SEO có cấu trúc, dễ kiểm thử và có thể mở rộng thay vì khai báo các mảng metadata rời rạc trong từng controller.

Các ví dụ ưu tiên Laravel 13.x (PHP 8.3+) nhưng chủ yếu sử dụng các API Laravel phổ biến (`route()`, `asset()`, `Storage`, Blade component, route model binding) nên có thể áp dụng cho Laravel 12 với rất ít thay đổi. Khi tích hợp vào dự án đang chạy phiên bản khác, đối chiếu release/upgrade guide của phiên bản đó.

> Phạm vi: title, meta description, canonical, robots, Open Graph, Twitter/X Card, JSON-LD structured data, hreflang, sitemap/robots.txt, pagination, reverse proxy và automated tests.

---

## 1. Kiến trúc đề xuất

Không nên để mỗi controller tự xây một `$meta` array khác nhau. Kiến trúc khuyến nghị:

```text
Route
  → Controller lấy business data
  → SeoFactory tạo SeoData
  → View nhận $seo
  → <x-seo :seo="$seo" /> trong <head>
  → HTML server trả về đầy đủ title/meta/canonical/OG/Twitter/JSON-LD
```

Cấu trúc file gợi ý:

```text
config/seo.php
app/SEO/SeoData.php
app/SEO/SeoFactory.php
app/SEO/SeoText.php
app/SEO/Schema/ArticleSchema.php
app/SEO/Schema/ProductSchema.php          # nếu có sản phẩm
app/SEO/Schema/OrganizationSchema.php     # nếu cần schema toàn site
app/View/Components/Seo.php
resources/views/components/seo.blade.php
resources/views/layouts/app.blade.php
routes/web.php
app/Http/Controllers/HomeController.php
app/Http/Controllers/PostController.php
```

Nếu dự án nhỏ, có thể giản lược `SeoFactory` hoặc các schema class, nhưng nên giữ `SeoData` để contract rõ ràng và tránh typo key.

---

## 2. Contract dữ liệu SEO

Tạo `app/SEO/SeoData.php`:

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

### Tại sao dùng object thay array

Object giúp:

- tránh lỗi typo như `desciption`, `canonial`;
- IDE/autocomplete tốt hơn;
- dễ unit test;
- dễ mở rộng fields;
- controller không cần biết chi tiết render.

Dùng tên `canonical` thay vì `url` vì trường này biểu diễn URL chuẩn của trang. `og:url` sẽ dùng cùng giá trị canonical nếu không có lý do đặc biệt để khác.

---

## 3. Cấu hình mặc định

Tạo `config/seo.php`:

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
        'Mô tả mặc định của website.'
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

`.env` production:

```dotenv
APP_NAME="Tên doanh nghiệp"
APP_URL=https://www.example.com
SEO_TITLE="Tên doanh nghiệp"
SEO_DESCRIPTION="Mô tả mặc định của website."
SEO_IMAGE=images/seo/default.jpg
SEO_LOCALE=vi_VN
SEO_ROBOTS=index,follow
SEO_TWITTER_CARD=summary_large_image
# SEO_TWITTER_SITE=@example
```

`.env` staging/dev:

```dotenv
SEO_ROBOTS=noindex,nofollow
```

Không nên phụ thuộc hoàn toàn vào `APP_ENV` để quyết định index/noindex. Dùng biến cấu hình SEO riêng giúp tránh staging bị index do cấu hình môi trường không như dự kiến.

Đặt ảnh mặc định tại:

```text
public/images/seo/default.jpg
```

`asset()` chỉ tạo URL, không kiểm tra file tồn tại. Sau deploy phải kiểm tra URL ảnh thực tế trả về `200` và public không yêu cầu đăng nhập.

---

## 4. Chuẩn hóa text dùng cho SEO

Tạo `app/SEO/SeoText.php`:

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

`160` chỉ là giới hạn biên tập hợp lý, không phải giới hạn bắt buộc của Google. Nếu description rỗng sau xử lý, `SeoFactory` nên dùng fallback mặc định.

---

## 5. Structured Data / JSON-LD

Structured data giúp công cụ tìm kiếm hiểu loại nội dung của trang. Với bài viết, nên dùng `Article`, `BlogPosting` hoặc `NewsArticle` tùy loại nội dung.

Tạo `app/SEO/Schema/ArticleSchema.php`:

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

Google không bắt buộc mọi property nhưng nên cung cấp các property phù hợp như `headline`, `image`, `datePublished`, `dateModified`, `author`.

Nếu có ảnh nhiều tỷ lệ cho article, có thể cung cấp 1:1, 4:3 và 16:9 để hỗ trợ tốt hơn cho rich results.

Không nối chuỗi thủ công để tạo JSON-LD.

---

## 6. SeoFactory

Tạo `app/SEO/SeoFactory.php`:

```php
<?php

namespace App\SEO;

use App\Models\Post;
use App\SEO\Schema\ArticleSchema;
use Illuminate\Support\Facades\Storage;

final class SeoFactory
{
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
            title: 'Trang chủ - ' . config('seo.site_name'),
            description: 'Giới thiệu dịch vụ chính và lợi ích dành cho khách hàng.',
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
            title: 'Bài viết'
                . ($page > 1 ? ' - Trang ' . $page : '')
                . ' - ' . config('seo.site_name'),
            description: 'Các bài viết và thông tin mới nhất từ doanh nghiệp.',
            canonical: $canonical,
            image: $default->image,
            type: 'website',
            robots: $default->robots,
            locale: $default->locale,
            siteName: $default->siteName,
            imageAlt: 'Bài viết - ' . config('seo.site_name'),
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

Nếu dùng S3/CDN hoặc model đã lưu URL tuyệt đối, thay logic tạo `$image` bằng URL public của hệ thống. Không đưa đường dẫn nội bộ như `storage/app/...` vào `og:image`, không dùng URL có auth/token ngắn hạn cho ảnh chia sẻ lâu dài.

---

## 7. Route model binding

Thay query slug thủ công bằng route model binding.

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

Laravel tự trả `404` nếu không tìm thấy model tương ứng.

Có thể thay `{post:slug}` bằng `{post}` nếu model khai báo:

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
            // Bổ sung điều kiện published theo schema thực tế.
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
        // Nếu có author dùng cho schema, eager-load trước khi build SEO.
        // $post->loadMissing('author');

        return view('posts.show', [
            'post' => $post,
            'seo' => $seoFactory->forPost($post),
        ]);
    }
}
```

Nếu trang chỉ được truy cập khi published/có quyền, route binding hoặc controller vẫn phải áp dụng cùng scope/quyền như business logic hiện tại. Không vì SEO mà vô tình cho phép truy cập bản nháp/private.

---

## 9. Blade component SEO

Tạo `app/View/Components/Seo.php`:

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

Tạo `resources/views/components/seo.blade.php`:

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

Không dùng `{!! ... !!}` cho title/description/OG. Riêng JSON-LD cần raw JSON nhưng phải encode bằng `json_encode()` với các `JSON_HEX_*` flags như trên; không chèn JSON do người dùng cung cấp trực tiếp.

Chỉ khai báo `og:image:width`/`height` khi biết đúng kích thước của ảnh. Không hard-code `1200x630` nếu ảnh động có thể khác kích thước.

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

Nếu muốn tránh gọi service container trong Blade, có thể yêu cầu mọi controller truyền `$seo`. Tuy nhiên fallback trên hữu ích cho các trang cũ trong quá trình migration.

Nếu dự án đã có `<title>`, meta description, OG hoặc canonical cũ, phải xóa/di chuyển sang component này để mỗi trang chỉ render một bộ metadata.

---

## 11. View nội dung

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
    <h1>Bài viết</h1>

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
        {{-- Nội dung bài viết --}}
    </article>
@endsection
```

View con không tạo thêm title/meta riêng nếu layout đã dùng `<x-seo>`.

---

## 12. Canonical URL

Nguyên tắc:

| Loại trang | Canonical khuyến nghị |
| --- | --- |
| Trang chủ | `route('home')` |
| Trang tĩnh | Route chuẩn của trang |
| Chi tiết bài viết | Route chứa slug của bài viết |
| Chi tiết sản phẩm | Route chi tiết sản phẩm |
| Pagination page 1 | URL không có `?page=1` |
| Pagination page 2+ | Self-canonical có `?page=N` |
| Tracking params | Canonical loại `utm_*`, `fbclid`, `gclid` nếu nội dung không đổi |
| Search/filter | Theo policy sản phẩm; không tự động canonical tất cả về một URL |

Không tự động xóa mọi query string. Một số query thay đổi nội dung thật sự.

`url()->current()` không gồm query string nên phù hợp làm fallback. Với pagination/search/filter, controller/SeoFactory phải tạo canonical rõ ràng.

Không canonical page 2 về page 1 nếu page 2 chứa tập nội dung khác.

---

## 13. Pagination

Mẫu trong `SeoFactory::postsIndex()` dùng:

```text
/bai-viet
/bai-viet?page=2
/bai-viet?page=3
```

và mỗi trang tự canonical về chính nó.

Không cần thêm `rel="next"`/`rel="prev"` chỉ để Google xử lý pagination; tập trung vào link HTML crawlable và canonical hợp lý.

---

## 14. Search và filter pages

Ví dụ search route:

```php
$seo = new SeoData(
    title: 'Tìm kiếm: ' . $query,
    description: 'Kết quả tìm kiếm cho: ' . $query,
    canonical: route('search', ['query' => $query]),
    image: asset(config('seo.default_image')),
    robots: 'noindex,follow',
    locale: config('seo.locale'),
    siteName: config('seo.site_name'),
);
```

`noindex,follow` là policy phổ biến cho search nội bộ, không phải luật bắt buộc. Nếu filter/search page là landing page SEO có nội dung ổn định và có giá trị, có thể cho index.

---

## 15. Hreflang cho website đa ngôn ngữ

Nếu website chỉ có một ngôn ngữ, bỏ qua phần này.

Nếu có các URL tương đương:

```text
/vi/san-pham/abc
/en/products/abc
```

truyền:

```php
alternates: [
    'vi' => route('products.show.vi', $product),
    'en' => route('products.show.en', $product),
    'x-default' => route('products.show.vi', $product),
],
```

Component sẽ render:

```html
<link rel="alternate" hreflang="vi" href="...">
<link rel="alternate" hreflang="en" href="...">
<link rel="alternate" hreflang="x-default" href="...">
```

Các URL alternate nên trỏ qua lại nhất quán giữa các phiên bản ngôn ngữ.

---

## 16. Sitemap và robots.txt

Meta robots và `robots.txt` không thay thế nhau.

- `meta robots` điều khiển index/follow ở mức trang;
- `robots.txt` chủ yếu điều khiển crawling;
- sitemap giúp liệt kê URL canonical quan trọng cần discovery.

### robots.txt tối thiểu

`public/robots.txt` production:

```text
User-agent: *
Allow: /

Sitemap: https://www.example.com/sitemap.xml
```

Staging có thể dùng:

```text
User-agent: *
Disallow: /
```

nhưng vẫn nên giữ `SEO_ROBOTS=noindex,nofollow` như lớp bảo vệ bổ sung. Không dựa chỉ vào `robots.txt` để yêu cầu de-index một URL đã được biết đến.

### Sitemap

Có thể tự tạo route/controller XML hoặc dùng package nếu website có nhiều loại resource. Sitemap chỉ nên chứa URL canonical, public, indexable.

---

## 17. View Composer: khi nào dùng

Laravel View Composer hợp lý cho dữ liệu global như:

- site name;
- organization schema chung;
- locale mặc định;
- navigation/footer data.

Không khuyến nghị dùng View Composer để quyết định SEO dynamic của Post/Product vì dependency bị ẩn và khó trace hơn `Controller → SeoFactory → View`.

---

## 18. Khi nào dùng SEO package

Tự implement như tài liệu này phù hợp khi cần:

- title/description;
- canonical;
- OG/Twitter;
- JSON-LD;
- hreflang;
- sitemap đơn giản.

Cân nhắc package nếu có:

- CMS chỉnh SEO fields từ admin;
- sitemap rất lớn/nhiều nguồn;
- nhiều loại JSON-LD phức tạp;
- tự sinh OG image;
- hệ thống hreflang lớn;
- SEO audit hoặc model traits dùng rộng khắp.

Không nên thêm dependency chỉ để thay vài thẻ meta đơn giản.

---

## 19. Reverse proxy, HTTPS và URL generation

`route()`, `asset()` và URL generation phụ thuộc host/scheme Laravel nhận từ request/cấu hình. `APP_URL` không tự sửa mọi lỗi khi app chạy sau load balancer/reverse proxy.

Kiểm tra:

- trusted proxies;
- forwarded headers;
- HTTPS termination;
- URL của CDN/storage disk;
- config cache;
- canonical/OG image thực tế trong **View Page Source**.

Không chỉ kiểm tra DOM sau JavaScript; metadata SEO phải có trong HTML server trả về nếu website là Blade SSR như tài liệu này.

---

## 20. Storage image

Nếu Post image nằm trên local `public` disk:

```sh
php artisan storage:link
```

Kiểm tra:

```text
/public/storage/...  → public
/storage/app/public/... → không dùng trực tiếp làm URL meta
```

Nếu dùng S3/CDN, `og:image` và Twitter image phải là URL public ổn định, không dùng pre-signed URL ngắn hạn trừ khi có chủ đích đặc biệt.

---

## 21. Automated tests

SEO dễ regression vì UI thường vẫn hiển thị bình thường. Nên có Feature Tests.

Ví dụ `tests/Feature/PostSeoTest.php`:

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

Nên có test cho:

- default fallback;
- canonical đúng slug;
- `?page=2` self-canonical;
- page 1 không canonical thành `?page=1`;
- noindex trên staging/config tương ứng;
- HTML trong description được strip;
- ký tự `&`, `'`, `"`, Unicode được escape đúng;
- JSON-LD encode đúng;
- ảnh fallback;
- trang không tồn tại trả 404;
- không render duplicate title/meta/canonical.

---

## 22. Validation và kiểm thử external

Sau khi deploy:

```sh
php artisan route:list
php artisan config:cache
php artisan view:cache
```

Nếu dự án dùng route cache:

```sh
php artisan route:cache
```

Các kiểm tra nên thực hiện:

- **View Page Source**: metadata có trong HTML server trả về;
- canonical/og:url dùng đúng HTTPS/domain;
- OG image public và trả status thành công;
- Rich Results Test cho JSON-LD;
- URL Inspection/Search Console khi cần;
- social sharing debugger phù hợp nền tảng nếu preview sai;
- sitemap trả XML hợp lệ và chỉ chứa canonical URLs.

---

## 23. Checklist nghiệm thu

- [ ] Trang chủ, list, detail render thành công.
- [ ] Slug không tồn tại trả 404.
- [ ] Mỗi page có đúng một `<title>`.
- [ ] Mỗi page có đúng một meta description.
- [ ] Mỗi page có đúng một canonical.
- [ ] `og:title`, `og:description`, `og:type`, `og:url`, `og:image` đúng.
- [ ] Twitter/X Card đúng.
- [ ] `og:site_name`, `og:locale`, `og:image:alt` đúng nếu được cấu hình.
- [ ] `og:image:width/height` chỉ render khi có dữ liệu chính xác.
- [ ] Article page có JSON-LD hợp lệ.
- [ ] Description không chứa markup HTML.
- [ ] Canonical và image dùng absolute URL.
- [ ] Page 2 có `?page=2`; page 1 không có `?page=1`.
- [ ] Tracking query không làm canonical thay đổi khi nội dung không đổi.
- [ ] Search/filter có policy index/noindex rõ ràng.
- [ ] Staging/dev có `noindex,nofollow` theo config.
- [ ] `robots.txt` phù hợp môi trường.
- [ ] sitemap chỉ chứa public canonical URLs.
- [ ] hreflang đúng nếu multilingual.
- [ ] Không có duplicate metadata từ layout/plugin cũ.
- [ ] Reverse proxy không làm URL thành HTTP hoặc sai domain.
- [ ] Feature tests SEO chạy pass.

---

## 24. Lỗi thường gặp

| Triệu chứng | Nguyên nhân / cách xử lý |
| --- | --- |
| `Route [...] not defined` | Kiểm tra `php artisan route:list`; sửa route name |
| 404 với `{post:slug}` | Slug không tồn tại, route binding không match hoặc global scope loại model |
| Ảnh share không tải | Kiểm tra storage link, disk URL, CDN, permission và fallback |
| `og:image` là relative URL | Dùng `asset()`, `url()` hoặc URL CDN tuyệt đối |
| Canonical của detail trỏ về list | Dùng route detail với model/slug |
| Page 2 canonical về page 1 | Tạo self-canonical theo page hiện tại |
| Domain/scheme sai | Kiểm tra trusted proxies, forwarded headers, APP_URL, CDN, config cache |
| Staging xuất hiện trên Google | `SEO_ROBOTS` sai, robots.txt sai hoặc staging từng được index trước đó |
| JSON-LD lỗi | Không nối JSON thủ công; dùng `json_encode()` và validate bằng Rich Results Test |
| Duplicate title/meta | Xóa block cũ hoặc package/component đang render song song |
| `.env` đổi nhưng output không đổi | Clear/rebuild config cache theo deployment process |
| Description chứa HTML | Dùng `SeoText::description()` |
| Twitter preview khác OG | Kiểm tra `twitter:*` fields và cache của nền tảng |

---

## 25. Quyết định kiến trúc cuối cùng

### Khuyến nghị mặc định

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

### Không khuyến nghị làm kiến trúc chính

```text
Controller tự build $meta array ở mọi action
```

vì logic SEO dễ bị lặp, khó refactor và khó type-check.

### Có thể dùng bổ trợ

- View Composer: cho dữ liệu SEO global;
- package SEO: khi CMS/schema/sitemap/hreflang phức tạp;
- custom schema builders: khi có Product, BreadcrumbList, Organization, LocalBusiness...

---

## 26. Nguồn tham khảo

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

## 27. Ghi chú migration từ implementation cũ

Nếu dự án hiện tại đã dùng `$meta` array:

1. tạo `SeoData`, `SeoFactory`, `<x-seo>` trước;
2. giữ fallback trong layout để các trang cũ vẫn hoạt động;
3. migrate từng controller sang `SeoFactory`;
4. xóa partial `$meta` cũ khi không còn page sử dụng;
5. chạy test để phát hiện duplicate metadata;
6. kiểm tra source HTML trên staging trước production.

Không cần migrate tất cả page trong một lần nếu dự án lớn, miễn layout chỉ render một implementation SEO tại mỗi request.
