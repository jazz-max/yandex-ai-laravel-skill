# yandex-ai-laravel skill

Claude Code skill for working with [Yandex AI Studio](https://aistudio.yandex.ru/) APIs in Laravel via the [`jazz-max/yandex-ai-laravel`](https://github.com/jazz-max/yandex-ai-laravel) SDK.

## What this skill provides

Expert guidance for integrating Yandex AI Studio into Laravel applications:

- **Responses API** — text generation, streaming, multi-turn dialogs, function calling, background tasks, reasoning
- **Vision OCR** — image and PDF document recognition with 12 OCR models
- **Embeddings** — vector embeddings for semantic search with cosine similarity
- **Cost calculation** — token pricing in RUB
- **Critical gotchas** — all known Yandex API quirks that differ from OpenAI

## Install

```bash
npx skills add jazz-max/yandex-ai-laravel-skill
```

## What the skill knows

- Full API reference for `ResponsesClient`, `VisionClient`, `EmbeddingsClient`
- `FunctionTool::make()` usage and empty-properties serialization bug
- Authorization format (`Api-Key`, not `Bearer`)
- Model URI format (`gpt://folder_id/model`)
- Responses API parameter mapping (vs OpenAI Chat Completions)
- OCR model selection, language limits, async JSON Lines parsing
- Embedding models and cosine similarity
- Pricing in RUB per 1000 tokens, cached tokens at 50%
- Reasoning mode (Pro models only)
- Laravel Octane/Swoole config caching

## Triggers

The skill activates on keywords: `Yandex AI`, `YandexGPT`, `Yandex OCR`, `Yandex Vision`, `Yandex Embeddings`, `yandex-ai-laravel`, `Responses API Yandex`.

## Related

- [jazz-max/yandex-ai-laravel](https://github.com/jazz-max/yandex-ai-laravel) — the SDK package
- [Yandex AI Studio](https://aistudio.yandex.ru/) — official platform
- [docs/GOTCHAS.md](https://github.com/jazz-max/yandex-ai-laravel/blob/main/docs/GOTCHAS.md) — production pitfalls

## License

MIT
