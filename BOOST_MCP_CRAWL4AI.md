# Laravel Boost MCP + Crawl4AI

This setup lets an MCP client use:

- Crawl4AI for scraping rendered pages.
- Laravel Boost for generating Laravel modules with `generate-module`.

## Recommended Dokploy Stack

Use `dokploy-boost-crawl4ai-stack.yml` if you want both containers in the same Raw provider stack.

Minimum variables:

```env
CRAWL4AI_APP_NAME=crawl4ai
CRAWL4AI_PORT=11235
CRAWL4AI_TAG=0.8.6
CRAWL4AI_SHM_SIZE=1gb

BOOST_APP_NAME=laravel-boost-mcp
BOOST_IMAGE=ghcr.io/smt197/laravel-boost-mcp
BOOST_TAG=latest
BOOST_PORT=8000
APP_KEY=base64:your-laravel-app-key
APP_URL=http://your-vm-ip:8000

CRAWL4AI_BASE_URL=http://crawl4ai:11235
```

Optional LLM keys for Crawl4AI:

```env
LLM_PROVIDER=openai/gpt-4o-mini
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=
GROQ_API_KEY=
DEEPSEEK_API_KEY=
TOGETHER_API_KEY=
MISTRAL_API_KEY=
GEMINI_API_TOKEN=
```

Useful endpoints:

- Crawl4AI health: `http://your-vm-ip:11235/health`
- Crawl4AI playground: `http://your-vm-ip:11235/playground`
- Crawl4AI MCP schema: `http://your-vm-ip:11235/mcp/schema`
- Laravel Boost MCP: `http://your-vm-ip:8000/api/mcp`
- Laravel Boost debug route in your test project: `http://your-vm-ip:8000/api/debug-boost`

## MCP Workflow

Use the tools in this order:

1. Scrape with Crawl4AI, preferably the `md` tool or `/md` endpoint.
2. Convert the scraped content into a module spec.
3. Call Boost `generate-module`.
4. Apply the returned dry-run files into the target Laravel project.

`generate-module` expects fields as strings:

```json
{
  "module_name": "products",
  "fields": [
    "name:string:required",
    "description:textarea:nullable",
    "price:number:required",
    "image:File:nullable"
  ],
  "identifier_field": "slug",
  "roles": ["user", "admin"],
  "initial_data": [
    {
      "name": "Scraped product",
      "description": "Content extracted from Crawl4AI",
      "price": 100
    }
  ]
}
```

Valid Boost field types:

```text
string, number, boolean, Date, File, textarea, quill-editor, email, password, link, url
```

## Optional: Add a Crawl4AI Tool Inside Boost

If you want the Boost MCP server itself to expose a `crawl4ai-markdown` tool, add this file to `test-package`:

`app/Mcp/Tools/Crawl4AiMarkdown.php`

```php
<?php

declare(strict_types=1);

namespace App\Mcp\Tools;

use Illuminate\Contracts\Foundation\Application;
use Illuminate\Contracts\JsonSchema\JsonSchema;
use Illuminate\JsonSchema\Types\Type;
use Illuminate\Support\Facades\Http;
use Laravel\Mcp\Request;
use Laravel\Mcp\Response;
use Laravel\Mcp\Server\Attributes\Name;
use Laravel\Mcp\Server\Tool;

#[Name('crawl4ai-markdown')]
class Crawl4AiMarkdown extends Tool
{
    protected string $description = 'Scrape a URL with the Crawl4AI container and return clean markdown.';

    /**
     * @return array<string, Type>
     */
    public function schema(JsonSchema $schema): array
    {
        return [
            'url' => $schema->string()
                ->description('Absolute http/https URL to scrape.')
                ->required(),
            'filter' => $schema->string()
                ->enum(['fit', 'raw', 'bm25', 'llm'])
                ->description('Crawl4AI markdown filter. Defaults to fit.'),
            'query' => $schema->string()
                ->description('Optional query for bm25 or llm filtering.'),
            'cache' => $schema->string()
                ->description('Use "1" to enable cache, "0" to bypass. Defaults to 0.'),
            'timeout' => $schema->integer()
                ->description('HTTP timeout in seconds. Defaults to 120.'),
        ];
    }

    public function handle(Request $request, Application $app): Response
    {
        $baseUrl = rtrim((string) env('CRAWL4AI_BASE_URL', 'http://crawl4ai:11235'), '/');
        $timeout = (int) $request->get('timeout', 120);

        $response = Http::timeout($timeout)->post($baseUrl.'/md', [
            'url' => $request->get('url'),
            'f' => $request->get('filter', 'fit'),
            'q' => $request->get('query'),
            'c' => (string) $request->get('cache', '0'),
        ]);

        if ($response->failed()) {
            return Response::error('Crawl4AI failed with status '.$response->status().': '.$response->body());
        }

        return Response::json($response->json());
    }
}
```

Then register it in `config/boost.php`:

```php
'mcp' => [
    'tools' => [
        'include' => [
            App\Mcp\Tools\Crawl4AiMarkdown::class,
        ],
        'exclude' => [],
    ],
],
```

Clear config cache after deploying the Laravel image:

```bash
php artisan config:clear
php artisan route:clear
```

The Boost MCP server will then expose both:

- `crawl4ai-markdown`
- `generate-module`

## Separate Dokploy Projects

If Crawl4AI and Laravel Boost are deployed as two separate Dokploy projects, put both on the same external Docker network.

Create the network once on the VM:

```bash
docker network create mcp-network
```

Then in both compose files:

```yaml
networks:
  mcp-network:
    external: true
```

And attach each service to `mcp-network`. The Laravel Boost container should use:

```env
CRAWL4AI_BASE_URL=http://crawl4ai:11235
```
