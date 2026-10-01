# Yuri's Umbrel Apps

Community app store for umbrelOS. Add this repo once on your Umbrel (**App Store → ⋯ → Add community app store**), then install any app published here. App IDs are prefixed with the store ID `yuri` (for example `yuri-maple-proxy`).

This store currently ships **Maple Proxy**. More apps can be added as sibling folders next to it (each with `umbrel-app.yml` + `docker-compose.yml`).

---

## Maple Proxy

System-wide OpenAI-compatible gateway to Maple's TEE inference. Install it once; it restarts with the box. Other apps on the Umbrel reach it at `http://yuri-maple-proxy_server_1:8080/v1`. Packaging version: **0.3.4** (container image currently pinned to OpenSecret's published `0.3.2` multi-arch build until a newer tag exists).

### Install & configure

1. Add this store on the Umbrel, then install **Maple Proxy**. It has no UI; it just runs.
2. Create a **Maple API key** at https://trymaple.ai (billing identity).
3. Point an OpenAI-compatible client at the proxy. Example — **Hermes Agent**:

```
model.provider  custom
model.base_url  http://yuri-maple-proxy_server_1:8080/v1
model.api_key   <your Maple API key>
model.default   kimi-k3        # or gpt-oss-120b / deepseek-v4-pro / gemma4-31b
```

### Verifying it works

From another app's shell on the Umbrel (or from the host):

```bash
curl -N http://yuri-maple-proxy_server_1:8080/v1/models \
  -H "Authorization: Bearer YOUR_MAPLE_KEY"
```

A JSON model list means the proxy is up and your key is valid. Then a streaming completion:

```bash
curl -N http://yuri-maple-proxy_server_1:8080/v1/chat/completions \
  -H "Authorization: Bearer YOUR_MAPLE_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-oss-120b","messages":[{"role":"user","content":"ping"}],"stream":true}'
```

### Known constraints

- **Streaming only.** Maple returns streamed responses exclusively; `stream=true` is mandatory on every request.
- **Image pin.** Compose pins `ghcr.io/opensecretcloud/maple-proxy:0.3.2` + its index digest for reproducible installs. Bump the tag/digest and the `version:` in `umbrel-app.yml` when a newer release is published.
- **No MAPLE_API_KEY is baked in**, so each app passes its own key and bills its own account. Add the env var if you want one shared default key.
- The proxy holds no state; nothing is written to `${APP_DATA_DIR}`.

### Design note

`app_proxy` is included so the app behaves like a normal umbrelOS app and the proxy is reachable from the LAN behind Umbrel's auth. Apps on the box should still use the container name (`yuri-maple-proxy_server_1:8080`), not the LAN URL. For a strictly internal service, delete the `app_proxy` service.
