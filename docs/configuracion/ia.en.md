# AI — the AI Gateway (BYOK)

Didacta's artificial intelligence is **multi-provider BYOK (Bring Your Own Key)**: no provider is hardwired into the product. Each installation — and each tenant — decides which LLM provider to use and with which key.

## How it works

All of the platform's AI goes through a single **AI Gateway** in the API (`apps/api/src/ai/`), serving two purposes:

- **`chat`** — text generation (conversational tutor, rubric-based grading, content generation).
- **`embed`** — embeddings for the tutor's RAG (stored in pgvector `vector` columns).

Bundled adapters: **OpenAI, Anthropic, Gemini, Ollama and Voyage**, plus any compatible endpoint through an alternative `BASE_URL` (proxies, gateways, self-hosted models).

## Who uses it

| Module | What it does with AI |
| --- | --- |
| `mod.ai-tutor` | A per-course conversational tutor with RAG over the published content; it cites the relevant lessons. |
| `mod.ai-grader` | Grading of open-ended answers against the instructor's rubrics; it proposes a score + feedback and the instructor confirms. |
| `mod.ai-content` | Drafts of summaries, flashcards and quizzes from a lesson — always in draft state (human-in-the-loop). |

If you configure no provider at all, these modules return a clear "provider not configured" error; the rest of the platform works as usual.

## Per-tenant configuration (recommended)

Each organization configures its provider and key under **Administration → AI providers**. Keys are encrypted at rest with AES-256-GCM.

!!! warning "Encryption key required for per-tenant AI"
    Managing per-tenant AI configurations requires `AI_CONFIG_ENCRYPTION_KEY`: a **hex key of exactly 64 characters** (32 bytes).
    ```bash
    AI_CONFIG_ENCRYPTION_KEY=$(openssl rand -hex 32)
    ```
    Without it, the AI configuration module fails to initialise (the rest of the app stays operational).

## Global fallback provider (optional)

For single-tenant installations, or as a fallback when a tenant has configured nothing:

```bash
# Chat
DEFAULT_AI_CHAT_PROVIDER=openai        # openai | anthropic | gemini | ollama | voyage
DEFAULT_AI_CHAT_API_KEY=sk-...
DEFAULT_AI_CHAT_MODEL=...
DEFAULT_AI_CHAT_BASE_URL=...           # optional

# Embeddings
DEFAULT_AI_EMBED_PROVIDER=openai
DEFAULT_AI_EMBED_API_KEY=sk-...
DEFAULT_AI_EMBED_MODEL=...
DEFAULT_AI_EMBED_BASE_URL=...          # optional
```

Resolution order: tenant configuration → global `DEFAULT_AI_*` provider → "provider not configured" error.
