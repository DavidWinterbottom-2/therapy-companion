# Single 14B model tier, not a 7-8B/14B split

The original plan split daily interactive chat (7-8B, fast) from periodic
pattern analysis (14B, slow but thorough). A live test on 2026-08-26 showed
14B alone, run as an Open WebUI workspace model, felt fine for interactive
back-and-forth. We collapsed to a single 14B tier for both daily chat and
periodic analysis: one system prompt/persona config to keep in sync instead
of two, and no evidence yet that the split's complexity earns its keep. If
14B chat ever feels sluggish in practice, the split can be reintroduced.
