# Network Allowlist

WhisPaste is a local-first desktop dictation app. This page lists every host
the app can contact on its own — no telemetry beacon, no hidden background
sync. Everything else you might click (GitHub, Ko-fi, X, Votepit, …) is a
regular link opened in your browser, not a call the app makes for you.

## App-initiated background connections

| Host | Purpose | Trigger | Data sent |
| --- | --- | --- | --- |
| `api.openai.com` | Cloud speech-to-text (OpenAI Whisper API) | Only if you choose OpenAI as your STT provider and add your own API key (BYOK) | Recorded audio, your API key |
| `api.deepgram.com` | Cloud speech-to-text (Deepgram Nova-3) | Only if you choose Deepgram as your STT provider and add your own API key (BYOK) | Recorded audio, your API key |
| `api.github.com` | Update check (release metadata) | Every app start / manual "Check for updates" | Nothing beyond the HTTPS request itself (no identifiers) |
| `github.com` (`releases/.../appcast.xml`) | Auto-update feed, stable channel | Same as above | Same as above |
| `raw.githubusercontent.com` | Auto-update feed, beta channel (only if you opted into the beta channel) | Same as above | Same as above |
| `huggingface.co` | Downloads on-device model weights (whisper.cpp GGML, Parakeet ONNX, and the optional Smart Mode Gemma model) | Only when you first select/download that model in Settings | Nothing beyond the HTTPS request itself |
| `*.ingest.de.sentry.io` | Crash reporting (Sentry, hosted in the EU) | Only if you keep crash reporting enabled (opt-out in Settings) — sends a report only when the app actually crashes | Crash stack trace, sanitized to strip file paths/PII before sending |
| Supabase project host (URL injected at build time, not hardcoded) | Feedback form submission | Only if you submit feedback via the in-app feedback form | The feedback text you wrote, no account/identity data |

## Not app-initiated (opened in your default browser instead)

`github.com`, `github.com/sponsors/...`, `ko-fi.com`, `x.com`, `app.votepit.com`,
and the WhisPaste website (`whispaste.de`) are only ever opened as external
links from About/Settings/feedback screens — the app never contacts them in
the background.

## Notes

- On-device transcription (the default) makes **no** network connection at
  all once the model is downloaded — audio never leaves your machine.
- Cloud STT (OpenAI/Deepgram) is opt-in and bring-your-own-key (BYOK): your
  API key is stored in the OS-native secure credential store, never sent
  anywhere except directly to that provider's own API.
- Crash reporting and the feedback form are both optional and can be turned
  off or simply never used.

See [`SECURITY.md`](./SECURITY.md) for how these connections are secured
(HTTPS-only, no plaintext fallbacks) and how to report a vulnerability.
