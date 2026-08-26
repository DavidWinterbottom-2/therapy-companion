# Custom Flask GUI, not Open WebUI as the interface

The design needs a dedicated "paste WhatsApp thread" input, a focus-areas
panel with add/edit/retire controls, and a persona reloaded fresh every
session — none of which are stock Open WebUI screens. Open WebUI's Tools/
Actions can render inline `HTMLResponse` widgets for this, but can't add a
persistent sidebar/nav panel without forking its Svelte frontend. We're
building a small custom Flask app instead (same stack/pattern as
`every-day-calender`), styled with the shared winterbottom design system,
and letting it own the chat/UI surface entirely rather than growing it inside
someone else's app.

Superseded consideration: staying inside Open WebUI as a workspace model was
the initial live-test setup and remains viable for prototyping, but was
rejected as the permanent interface for the reasons above.
