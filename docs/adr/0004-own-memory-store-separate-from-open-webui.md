# Own SQLite + embeddings memory store, kept separate from Open WebUI's Memory

Open WebUI ships a native Memory feature (`/api/v1/memories/`) that's
REST-queryable outside the chat UI — it could have doubled as this app's
semantic recall layer. We're building a separate SQLite + embeddings store
instead, for two reasons: Open WebUI's Memory holds discrete fact/preference
snippets rather than full-conversation recall, which doesn't fit the "recall
whole past exchanges" requirement; and building it ourselves produces a
small, reusable "local store + semantic search" component this project's
author can lift into unrelated, non-AI projects later — a deliberate side
goal of building this as a learning exercise, not just the fastest path to a
working app.
