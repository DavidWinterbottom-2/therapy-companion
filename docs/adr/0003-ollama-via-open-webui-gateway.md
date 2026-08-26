# Talk to Ollama through the existing Open WebUI gateway, not directly

Even though this app's GUI is custom (ADR 0002), model calls are routed
through the existing Open WebUI instance on `desktop-docker` rather than
straight to Ollama's native API — matching how every other app in this
household's infra (e.g. `lisas-nutrition-tracker`'s vision feature) reaches
Ollama. Open WebUI's `/api/v1/models` REST API (`model/update`, `sync`)
supports pushing a fresh system prompt per session programmatically, so
directness isn't required to get per-session persona reload. Keeping one
gateway for all AI access outweighs the minor extra hop, which is
same-host anyway.
