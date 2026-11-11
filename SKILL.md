---
name: yandex-ai-laravel
description: Use when working with Yandex AI Studio APIs in Laravel — text generation (Responses API), Vision OCR, Embeddings. Invoke for YandexGPT integration, Yandex OCR, vector embeddings, function calling with Yandex models.
license: MIT
metadata:
  author: jazz-max
  version: "1.2.0"
  domain: backend
  triggers: Yandex AI, YandexGPT, Yandex OCR, Yandex Vision, Yandex Embeddings, yandex-ai-laravel, Responses API Yandex, function calling Yandex
  role: specialist
  scope: implementation
  output-format: code
  tags:
    - laravel
    - yandex
    - ai
    - gpt
    - ocr
    - vision
    - embeddings
  related-skills: laravel-specialist
---

# Yandex AI Laravel Specialist

Expert in integrating Yandex AI Studio APIs with Laravel via the `jazz-max/yandex-ai-laravel` SDK (v1.2.0+). Covers text generation (Responses API), Vision OCR, and vector Embeddings. Includes built-in fallback parser for unreliable function calling and tool validation.

## When to Use This Skill

- Integrating YandexGPT text generation into a Laravel app
- Implementing streaming responses from Yandex AI
- Building multi-turn dialogs with `previous_response_id`
- Setting up function calling (tool use) with Yandex models
- Recognizing text from images or PDF documents (Yandex Vision OCR)
- Working with vector embeddings for semantic search
- Calculating API costs in RUB
- Debugging Yandex AI API errors and quirks

## Role Definition

You are a specialist in the `jazz-max/yandex-ai-laravel` SDK. You understand the critical differences between Yandex AI Studio and OpenAI APIs, know all the gotchas, and write reliable integration code.

## Core Workflow

1. **Check configuration** — Ensure `YANDEX_AI_API_KEY` and `YANDEX_AI_FOLDER_ID` are set in `.env`
2. **Choose the right client** — `responses()` for text, `vision()` for OCR, `embeddings()` for vectors
3. **Write integration code** — Follow SDK patterns, avoid known pitfalls
4. **Handle errors** — Catch `YandexAiException` with status codes and response bodies
5. **Track costs** — Use `calculateCostRub()` and `priceRub()` for billing

---

## Package Setup

```bash
composer require jazz-max/yandex-ai-laravel
php artisan vendor:publish --tag=yandex-ai-config
```

```env
YANDEX_AI_API_KEY=your-api-key
YANDEX_AI_FOLDER_ID=your-folder-id
```

Namespace: `JazzMax\YandexAi\`
Facade: `JazzMax\YandexAi\Facades\YandexAi`
Config key: `yandex-ai`

---

## Architecture

`YandexAi` is a singleton registered in `YandexAiServiceProvider`. It lazy-loads three clients sharing a Guzzle HTTP instance:

```
YandexAi (singleton)
├── responses() → ResponsesClient  — text generation, streaming, function calling, dialog, background tasks
├── vision()    → VisionClient      — sync image OCR, async PDF OCR
└── embeddings()→ EmbeddingsClient  — document/query vectors, cosine similarity
```

DTOs: `Responses\Response`, `Vision\OcrResult`
Helpers: `Tools\FunctionTool`, `Enums\OcrModel`, `Exceptions\YandexAiException`

---

## API Reference

### ResponsesClient — Text Generation

```php
use JazzMax\YandexAi\Facades\YandexAi;

$client = YandexAi::responses();
```

#### create(array $params): Response

```php
$response = YandexAi::responses()->create([
    'model'            => 'yandexgpt-5-lite',   // auto-formatted to gpt://folder_id/...
    'instructions'     => 'You are a helper.',   // NOT 'system'
    'input'            => 'Hello!',              // NOT 'messages'; string or array
    'temperature'      => 0.3,
    'max_output_tokens'=> 500,                   // NOT 'max_tokens'
]);

echo $response->text;              // extracted message text
echo $response->id;                // response ID for dialog continuation
echo $response->usage['total_tokens'];
```

#### stream(array $params, Closure $onDelta, ?Closure $onComplete, ?Closure $onFunctionCall): Response

```php
YandexAi::responses()->stream(
    ['model' => 'yandexgpt-5-lite', 'input' => 'Tell a story'],
    onDelta: fn($chunk) => echo $chunk,
    onComplete: fn($resp) => logger()->info("Tokens: {$resp->totalTokens()}")
);
```

Streaming with function calls (v1.2.0+):
```php
YandexAi::responses()->stream(
    ['model' => 'yandexgpt-5-lite', 'input' => 'Weather?', 'tools' => $tools],
    onDelta: fn($chunk) => echo $chunk,
    onComplete: fn($resp) => null,
    onFunctionCall: function (array $call) {
        // Called when function call arguments streaming is complete
        // $call = ['id' => '...', 'name' => '...', 'arguments' => [...]]
        $result = executeFunction($call['name'], $call['arguments']);
    }
);
```

#### continue(string $prevId, array $input, array $extra = []): Response

```php
$reply = YandexAi::responses()->continue($response->id, [
    ['role' => 'user', 'content' => 'Tell me more'],
]);
```

#### Function Calling

```php
use JazzMax\YandexAi\Tools\FunctionTool;

$tools = [
    // IMPORTANT: descriptions MUST be < 15 words, English only
    // FunctionTool::make() validates this and logs warnings automatically (v1.2.0+)
    FunctionTool::make('get_weather', 'Get weather for a city.', [
        'type'       => 'object',
        'properties' => [
            'city' => ['type' => 'string', 'description' => 'City name'],
        ],
        'required' => ['city'],
    ]),
];

$response = YandexAi::responses()->create([
    'model'        => 'yandexgpt-5-lite',
    'instructions' => 'You are a helpful assistant with tools.', // SHORT, English, no "answer as code" or "use markdown"
    'input'        => 'Weather in Moscow?',
    'tools'        => $tools,
    'temperature'  => 0.3, // low temperature improves function calling reliability
]);

if ($response->hasFunctionCall()) {
    $call = $response->functionCall; // ['id' => ..., 'name' => ..., 'arguments' => ...]

    // Check if function call was extracted from text via fallback parser (v1.2.0+)
    // This means the model returned tool call as text, not as proper function_call item
    if ($response->isFallbackFunctionCall) {
        logger()->notice("Function call '{$call['name']}' extracted from text via fallback parser.");
    }

    $result = getWeather($call['arguments']['city']);
    $final = YandexAi::responses()->submitToolOutput(
        $response->id, $call['id'], json_encode($result)
    );
}
```

#### Tool Validation (v1.2.0+)

`FunctionTool::make()` automatically validates tool definitions and logs warnings:
- Description > 15 words → warning (causes model to return text instead of function_call)
- Non-ASCII characters in description → warning (Russian descriptions break function calling)
- Empty `properties` array → warning (PHP serialization bug)

You can also validate manually:
```php
$warnings = FunctionTool::validate('my_tool', 'My description', $params);
// Returns string[] of warning messages (empty if OK)
```

#### Building function_call_output (v1.2.0+)

```php
// Helper to build function_call_output items
$output = FunctionTool::output($call['id'], json_encode($result));
// Returns: ['type' => 'function_call_output', 'call_id' => '...', 'output' => '...']
```

#### Background Tasks

```php
$task = YandexAi::responses()->createBackground([
    'model' => 'yandexgpt-5-pro',
    'input' => 'Write a long essay...',
]);

$completed = YandexAi::responses()->poll($task->id, timeout: 120, interval: 3);
echo $completed->text;
```

#### Reasoning (Pro models only)

```php
$response = YandexAi::responses()->create([
    'model'     => 'yandexgpt-5-pro',
    'input'     => 'Complex math problem...',
    'reasoning' => ['effort' => 'high'], // low, medium, high
]);
// Reasoning tokens in: $response->usage['reasoning_tokens']
```

#### Cost Calculation

```php
$cost = YandexAi::responses()->calculateCostRub(
    model: 'yandexgpt-5-lite',
    promptTokens: $response->usage['prompt_tokens'],
    completionTokens: $response->usage['completion_tokens'],
    cachedTokens: $response->usage['cached_tokens']  // charged at 50%
);
```

#### Utility Methods

```php
$client->formatModel('yandexgpt-5-lite');      // → 'gpt://folder_id/yandexgpt-5-lite'
$client->resolveModelName('gpt://f/model');    // → 'model'
$client->supportsReasoning('yandexgpt-5-pro'); // → true
```

### Response DTO

```php
$response->id                        // string
$response->model                     // string
$response->status                    // 'completed' | 'in_progress' | 'failed'
$response->text                      // ?string — extracted text (null if function call extracted from text)
$response->functionCall              // ?array — ['id', 'name', 'arguments']
$response->output                    // array — raw output
$response->usage                     // array — token counts
$response->raw                       // array — full API response
$response->isFallbackFunctionCall    // bool — true if function call was extracted from text via fallback parser (v1.2.0+)
$response->hasFunctionCall()         // bool
$response->isCompleted()             // bool
$response->promptTokens()            // int
$response->completionTokens()        // int
$response->totalTokens()             // int

// Usage details (v1.2.0+):
$response->usage['prompt_tokens']     // int — input tokens
$response->usage['completion_tokens'] // int — output tokens
$response->usage['total_tokens']      // int — total
$response->usage['cached_tokens']     // int — cached input tokens (charged at 50%)
$response->usage['reasoning_tokens']  // int — reasoning tokens (Pro models only)
```

---

### VisionClient — OCR

```php
use JazzMax\YandexAi\Facades\YandexAi;
use JazzMax\YandexAi\Enums\OcrModel;

$client = YandexAi::vision();
```

#### recognizeText(string $imageData, OcrModel $model = Page): OcrResult

```php
$result = YandexAi::vision()->recognizeText(
    file_get_contents('photo.jpg'),
    OcrModel::Page
);
echo $result->text();         // best text representation
echo $result->fullText;       // raw recognized text
echo $result->priceRub();     // cost in RUB
```

#### recognizeDocument(string $pdfData, OcrModel $model, int $timeout = 120): OcrResult

```php
$result = YandexAi::vision()->recognizeDocument(
    file_get_contents('document.pdf'),
    OcrModel::Markdown,
    timeout: 180
);
echo $result->text();
echo $result->pages;          // number of pages
```

#### OcrModel Enum

| Model | Languages | Use case |
|-------|-----------|----------|
| `Page` | `*` | General text |
| `PageColumnSort` | `*` | Multi-column layouts |
| `Handwritten` | `ru, en` only | Handwriting |
| `Table` | `ru, en` only | Tables |
| `Markdown` | `*` | Structured markdown |
| `MathMarkdown` | `*` | Math formulas (LaTeX) |
| `Passport` | `*` | Russian passport (template) |
| `DriverLicenseFront/Back` | `*` | Driver license (template) |
| `VehicleRegistrationFront/Back` | `*` | Vehicle registration (template) |
| `LicensePlates` | `*` | License plates |

Template models (`isTemplate() === true`) return structured `entities` instead of plain text.

#### OcrResult DTO

```php
$result->fullText       // string
$result->markdown       // ?string (Markdown models)
$result->entities       // array (template models)
$result->model          // OcrModel
$result->pages          // int
$result->text()         // best representation (markdown > entities > fullText)
$result->formatEntities() // entities with Russian labels
$result->priceRub()     // cost
```

---

### EmbeddingsClient — Vectors

```php
$client = YandexAi::embeddings();
```

```php
// Index a document
$docVector = YandexAi::embeddings()->embedDocument('Document text...');

// Search query
$queryVector = YandexAi::embeddings()->embedQuery('search query');

// Compare
$similarity = JazzMax\YandexAi\Embeddings\EmbeddingsClient::cosineSimilarity(
    $docVector, $queryVector
);
// 0.0 = unrelated, 1.0 = identical
```

Models used internally: `text-search-doc/latest` (documents), `text-search-query/latest` (queries).

---

### Exception Handling

```php
use JazzMax\YandexAi\Exceptions\YandexAiException;

try {
    $response = YandexAi::responses()->create([...]);
} catch (YandexAiException $e) {
    $e->getMessage();    // error description
    $e->statusCode;      // HTTP status (401, 429, 500...)
    $e->response;        // raw API response body
}
```

---

## Critical Gotchas (MUST KNOW)

### 1. Authorization: Api-Key, NOT Bearer
Yandex uses `Authorization: Api-Key YOUR_KEY` header + `x-folder-id` header. NOT `Bearer`. The SDK handles this automatically.

### 2. Model URI format
Models must be `gpt://folder_id/model_name`. Plain names like `yandexgpt-5-lite` won't work. `ResponsesClient::formatModel()` handles this — just pass the short name.

### 3. Responses API, NOT Chat Completions
Parameter mapping from OpenAI:
- `system` → `instructions`
- `messages` → `input`
- `max_tokens` → `max_output_tokens`
- Dialog uses `previous_response_id`, not message arrays

### 4. Function calling is fragile
- Tool descriptions MUST be **< 15 words** and in **English** — otherwise the model writes `[tool_name]{}` as plain text
- `FunctionTool::make()` validates descriptions automatically (v1.2.0+) and logs warnings for long/non-English descriptions
- System prompt for function calling should be **< 250 chars, English**
- **NEVER** include "answer as code", "use markdown", "format output as code block" or similar formatting instructions in `instructions` when using tools — this causes the model to output function calls as formatted text/code instead of proper `function_call` items
- NEVER use empty `properties: {}` — PHP's `stdClass` serializes to `[]` not `{}`
- Always use `FunctionTool::make()` which adds a `_unused` dummy property automatically and validates descriptions
- The SDK includes a **fallback parser** (`FunctionCallFallbackParser`) that extracts function calls from text responses. It handles `[tool]{}`, `tool{}`, and `tool()` patterns. Enabled by default; disable via `YANDEX_AI_FUNCTION_CALL_FALLBACK=false`
- Check `$response->isFallbackFunctionCall` to monitor when the model returns tool calls as text — this indicates degraded behavior that should be investigated
- Use low temperature (0.1–0.3) for function calling requests — higher values reduce reliability
- Always test with curl before deploying

### 5. Images: only qwen3.6-35b-a3b
`gemma-3-27b-it` was removed from Yandex AI Studio in 2026 and now returns 400 Bad Request. `qwen3.6-35b-a3b` is the only model that accepts images; `aliceai-llm*` and `yandexgpt-*` return empty output for them. Qwen reasons by default — pass `'reasoning' => ['effort' => 'none']` for fast, clean output, or give it 1000+ `max_output_tokens` for hard images (curved/rotated text). List what your folder has via `GET /v1/models`. Images must be base64-embedded (`data:image/jpeg;base64,...`). URL references are NOT supported.

### 6. Pricing is RUB per 1000 tokens
NOT USD per 1M. Formula: `(tokens / 1000) * price`. Cached tokens cost 50% of prompt price. Do NOT apply OpenAI-style division.

### 7. Async OCR returns JSON Lines
`recognizeDocument()` internally parses JSON Lines (one JSON per line), not a JSON array. The SDK handles this, but if you call the API directly, parse with `explode("\n", $body)`.

### 8. OCR language limits
`Handwritten` and `Table` models only support `['ru', 'en']`. Sending `['*']` causes errors. The SDK handles this via `OcrModel::languages()`.

### 9. Reasoning is Pro-only
Only `yandexgpt-5-pro` and `yandexgpt-5.1` support the `reasoning` parameter. Sending it to lite models may error. Check with `supportsReasoning()`.

### 10. Octane/Swoole config caching
Config is cached in worker memory. After `.env` changes: `php artisan config:clear && php artisan octane:reload`. Queue workers: `php artisan queue:restart`.

---

## Available Models

### Text Generation
| Model | Pricing (RUB/1K tokens) | Reasoning | Images |
|-------|------------------------|-----------|--------|
| `yandexgpt-5-lite` | 0.2 prompt / 0.2 completion | No | No |
| `yandexgpt-5-pro` | 1.2 / 1.2 | Yes | No |
| `yandexgpt-5.1` | 0.8 / 0.8 | Yes | No |
| `aliceai-llm` | 0.5 / 1.2 | No | No |
| `qwen3.6-35b-a3b` | ~0.4 / 0.4 (verify) | Yes, on by default | Yes (base64) |
| ~~`gemma-3-27b-it`~~ | removed in 2026 (400) | — | — |

### Embeddings
- `text-search-doc/latest` — for indexing documents
- `text-search-query/latest` — for search queries

---

## Configuration Reference

Config file: `config/yandex-ai.php`

| Key | Env Variable | Default |
|-----|-------------|---------|
| `api_key` | `YANDEX_AI_API_KEY` | — |
| `folder_id` | `YANDEX_AI_FOLDER_ID` | — |
| `base_url` | `YANDEX_AI_BASE_URL` | `https://ai.api.cloud.yandex.net/v1` |
| `ocr_base_url` | `YANDEX_AI_OCR_URL` | `https://ocr.api.cloud.yandex.net/ocr/v1` |
| `timeout` | `YANDEX_AI_TIMEOUT` | `120` |
| `connect_timeout` | `YANDEX_AI_CONNECT_TIMEOUT` | `10` |
| `proxy` | `YANDEX_AI_PROXY` | `null` |
| `verify_ssl` | `YANDEX_AI_VERIFY_SSL` | `false` |
| `default_model` | `YANDEX_AI_MODEL` | `yandexgpt-5-lite` |
| `function_call_fallback` | `YANDEX_AI_FUNCTION_CALL_FALLBACK` | `true` |

---

## Constraints

### MUST DO
- Always use `FunctionTool::make()` for building tool definitions — it validates descriptions and adds dummy property for empty params
- Keep tool descriptions < 15 words, in English
- Check `$response->isFallbackFunctionCall` after function calling requests to detect degraded behavior
- When using tools, keep `instructions` short (< 250 chars), in English, and avoid any formatting requirements ("answer as code", "use markdown", etc.)
- Use low temperature (0.1–0.3) for function calling requests
- Use `OcrModel` enum for OCR model selection
- Catch `YandexAiException` for API error handling
- Use `calculateCostRub()` for cost tracking
- Check `supportsReasoning()` before adding reasoning parameter
- Use `previous_response_id` for multi-turn dialogs (not message history arrays)

### MUST NOT DO
- Use `Bearer` token auth (Yandex requires `Api-Key`)
- Send plain model names without URI format (SDK handles this, but don't bypass)
- Use empty `properties` in function tools (causes serialization bug)
- Write tool descriptions in Russian or > 15 words
- Include "answer as code", "format as markdown", "use code blocks" or similar formatting requirements in `instructions` when `tools` are provided — this causes the model to print function calls as text/code instead of returning proper `function_call` output items
- Send images to any model except `qwen3.6-35b-a3b` (`gemma-3-27b-it` is removed)
- Use image URLs (only base64 supported)
- Apply OpenAI pricing formulas (per 1M USD) — Yandex is per 1K RUB
- Send `['*']` languages to `handwritten`/`table` OCR models
- Add `reasoning` parameter to models other than Pro / 5.1 / Qwen
