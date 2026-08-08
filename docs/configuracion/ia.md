# IA — el AI Gateway (BYOK)

La inteligencia artificial de Didacta es **BYOK (Bring Your Own Key) multi-proveedor**: ningún proveedor está cableado en el producto. Cada instalación — y cada tenant — decide qué proveedor LLM usa y con qué clave.

## Cómo funciona

Toda la IA de la plataforma pasa por un único **AI Gateway** en la API (`apps/api/src/ai/`), con dos propósitos:

- **`chat`** — generación de texto (tutor conversacional, corrección con rúbrica, generación de contenido).
- **`embed`** — embeddings para el RAG del tutor (se guardan en columnas `vector` de pgvector).

Adaptadores incluidos: **OpenAI, Anthropic, Gemini, Ollama y Voyage**, más cualquier endpoint compatible vía `BASE_URL` alternativa (proxies, gateways, modelos self-hosted).

## Quién la usa

| Módulo | Qué hace con la IA |
| --- | --- |
| `mod.ai-tutor` | Tutor conversacional por curso con RAG sobre el contenido publicado; cita las lecciones relevantes. |
| `mod.ai-grader` | Corrección de respuestas abiertas con rúbricas del formador; propone nota + feedback y el formador confirma. |
| `mod.ai-content` | Borradores de resúmenes, flashcards y quizzes desde una lección — siempre en estado borrador (human-in-the-loop). |

Si no configuras ningún proveedor, estos módulos devuelven un error claro de «proveedor no configurado»; el resto de la plataforma funciona igual.

## Configuración por tenant (recomendada)

Cada organización configura su proveedor y clave en **Administración → Proveedores de IA**. Las claves se cifran at-rest con AES-256-GCM.

!!! warning "Clave de cifrado obligatoria para IA por tenant"
    La gestión de configuraciones de IA por tenant necesita `AI_CONFIG_ENCRYPTION_KEY`: una clave **hex de exactamente 64 caracteres** (32 bytes).
    ```bash
    AI_CONFIG_ENCRYPTION_KEY=$(openssl rand -hex 32)
    ```
    Sin ella, el módulo de configuración de IA falla al inicializar (el resto de la app sigue operativa).

## Proveedor global de respaldo (opcional)

Para instalaciones mono-tenant o como fallback cuando un tenant no ha configurado nada:

```bash
# Chat
DEFAULT_AI_CHAT_PROVIDER=openai        # openai | anthropic | gemini | ollama | voyage
DEFAULT_AI_CHAT_API_KEY=sk-...
DEFAULT_AI_CHAT_MODEL=...
DEFAULT_AI_CHAT_BASE_URL=...           # opcional

# Embeddings
DEFAULT_AI_EMBED_PROVIDER=openai
DEFAULT_AI_EMBED_API_KEY=sk-...
DEFAULT_AI_EMBED_MODEL=...
DEFAULT_AI_EMBED_BASE_URL=...          # opcional
```

Orden de resolución: configuración del tenant → proveedor global `DEFAULT_AI_*` → error «proveedor no configurado».
