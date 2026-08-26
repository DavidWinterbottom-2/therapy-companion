# Nightly backup to NAS + Wasabi from day one, not git-based

`desktop-docker` has no blanket backup for its services — it's opt-in per
service (Immich has its own NAS backup job; Flatnotes relies on git). The
JSONL/SQLite journal data here is irreplaceable and highly sensitive, so we
built backup in from the start rather than deferring it. Rejected a
Flatnotes-style git backup specifically because of the content: a private
GitHub repo still means a third-party code-hosting platform holding years of
personal reflections and relationship detail, which is a worse fit than
infrastructure fully under the author's control. Nightly `rsync`/`restic` to
the NAS, mirrored to Wasabi for an offsite copy.
