# Public exposure via Azure Entra OIDC, not Tailscale/LAN-only

Despite the sensitivity of the content (marriage-conflict and mental-health
material), this is exposed publicly — a `winterbottom.xyz` subdomain behind
Entra OIDC (`entra-app`, single-tenant, `ALLOWED_EMAILS` restricted to the
author only) — rather than kept Tailscale/LAN-only. The requirement is a
companion that's genuinely available "anytime," including away from home,
and Tailscale would add a VPN-toggle dependency that cuts against that. This
follows the same auth pattern already used for other personal apps in this
infra (`nutrition-tracker`, `german-story-generator`), so it isn't a new
security posture — just applying an existing, trusted pattern to more
sensitive content.
