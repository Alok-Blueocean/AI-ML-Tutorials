# Videos — Observability, OpenTelemetry, and Production Monitoring

A note on how this list was built: YouTube's site could not be reached directly from the
research environment used to compile this module (no page-level fetch, no oEmbed lookup),
so titles and URLs below come from verified web-search results rather than a direct watch-through.
Where a channel or duration could not be independently confirmed, that field is marked
**unconfirmed** instead of guessed — verify the channel/runtime yourself before assigning it
as required viewing. Prefer the ★★★★★ items first; they map most directly to this module's
backbone (privacy-safe telemetry, TTFT/P95/P99, OTel spans, Prometheus/Grafana, Langfuse/LangSmith).

---

## ★★★★★ Core OpenTelemetry signal model

### "OpenTelemetry Fundamentals: Traces, Metrics & Logs Explained"
- **Creator/Channel:** independent OpenTelemetry-focused tutorial creator (channel name unconfirmed — verify on YouTube before citing in a syllabus)
- **URL:** https://www.youtube.com/watch?v=ItZouStG_nk
- **Difficulty:** Beginner
- **Why it's worth watching:** Walks through the three OTel signal types (traces, metrics, logs) as a single mental model before any code — exactly the ordering this module needs: understand *why* three signals exist before instrumenting an LLM service with all three.
- **Complements:** The "why three signals, not one" framing section, before the hands-on OTel-Python instrumentation walkthrough.

### "Logs, Metrics, Traces: How to Correlate Them with OpenTelemetry"
- **Creator/Channel:** unconfirmed (verify before citing)
- **URL:** https://www.youtube.com/watch?v=kft86_LA-Pg
- **Difficulty:** Intermediate
- **Why it's worth watching:** The single hardest practical skill in observability is correlating a slow trace with the log lines and metric spikes that explain it — this is precisely the skill needed to debug "P99 latency spiked, which retrieval or generation span caused it, and what did the log line say." Directly reinforces the module's telemetry.json + span correlation pattern.
- **Complements:** The section on tying `trace_id` into every structured log line and Grafana panel drill-down.

---

## ★★★★☆ OpenTelemetry mechanics

### "OpenTelemetry in Node.js - Traces, Metrics and Logs"
- **Creator/Channel:** unconfirmed (verify before citing)
- **URL:** https://www.youtube.com/watch?v=NbVVZlSsvvM
- **Difficulty:** Intermediate
- **Why it's worth watching:** Even for a Python-first course, watching an OTel SDK wired up in a second language reinforces that the SDK/API split, span/attribute model, and exporter configuration are language-agnostic — useful if your LLM gateway is polyglot (e.g., a Node.js BFF calling a Python inference service).
- **Complements:** The "SDK vs. API vs. Collector" architecture explanation.

### "How OpenTelemetry Works - How to Collect Logs"
- **Creator/Channel:** unconfirmed (verify before citing)
- **URL:** https://www.youtube.com/watch?v=bIxt1b0GOU4
- **Difficulty:** Beginner
- **Why it's worth watching:** Logs are the newest and least mature OTel signal (compared to traces/metrics) and the one most people configure wrong first — this fills the gap between "I emit print statements" and "I emit OTel log records with trace context attached."
- **Complements:** The privacy-safe telemetry section — watch alongside, then contrast with the module's rule of never logging raw user text.

### Playlist: "Metrics, Events, Logs, Traces with OpenTelemetry"
- **Creator/Channel:** playlist ID pattern (`PLdsu0umqbb8...`) matches other official OpenTelemetry-project playlists; treat as likely-official but verify the channel banner before citing
- **URL:** https://www.youtube.com/playlist?list=PLdsu0umqbb8MIdK1H0b6F8D9RJaH34s6_
- **Difficulty:** Beginner → Intermediate (progresses across the playlist)
- **Why it's worth watching:** A multi-video arc rather than a single talk — useful as a "watch one video per OTel concept" companion track alongside this module's chapters.
- **Complements:** Use as a supplementary deep-dive track after the main chapter, one video per sub-topic (context propagation, exporters, sampling).

---

## Recommended channel-level follow-ups (no single verified video, but channel is real and on-topic)

These are real, well-established channels confirmed to publish in this space; search each channel directly for the freshest 2025–2026 talk rather than relying on one pinned URL, since specific KubeCon/GrafanaCON session titles change every conference cycle:

- **CNCF [Cloud Native Computing Foundation]** — publishes recorded KubeCon + CloudNativeCon talks. Given the OpenTelemetry GenAI Semantic Conventions SIG has been active since April 2024, search "OpenTelemetry GenAI" on this channel for the latest KubeCon session on LLM/agent observability. ★★★★☆ Advanced, conference-talk depth.
- **Grafana Labs** — publishes GrafanaCON talks and product walkthroughs on building Grafana dashboards over Prometheus/Tempo/Loki data; search "Grafana LLM observability" or "Grafana AI observability" for current-year sessions. ★★★★☆ Intermediate.
- **OpenTelemetry project channel** — hosts SIG meeting recordings (including the GenAI SIG) and end-to-end demo walkthroughs of the official `opentelemetry-demo` ("Astronomy Shop") application. ★★★☆☆ Advanced — meeting recordings are unpolished but authoritative on where the GenAI semantic conventions are heading.

## How to use this list
1. Start with the two ★★★★★ videos to build the mental model (three signals, then correlation).
2. Use the ★★★★☆ OTel-mechanics videos to see SDK instrumentation in practice.
3. Search the channel-level recommendations for the freshest conference talk on GenAI observability — this space moves fast enough (semantic conventions were still marked "experimental" as of early 2026) that a 2024 talk will already show a different attribute schema than production code should use today.
