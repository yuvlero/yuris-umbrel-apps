# Maple Proxy for umbrelOS — community app package

A one-app Community App Store that puts **Maple Proxy** on the Umbrel as a
system-wide service: any app on the box can then use Maple's TEE inference
through an OpenAI-compatible endpoint. Install it once; it restarts with the
box and updates by bumping `version` in this repo.

## Why this shape (decided 2026-09-29)

| Option | Verdict |
|---|---|
| **Community app store app (this)** | **Chosen.** A real umbrelOS app: managed lifecycle, auto-restart, no host port conflicts, and other apps reach it by container name on the shared `umbrel_main_network` — exactly the "any app can use it" requirement. |
| Portainer | Rejected as primary: Portainer runs its **own nested Docker daemon** (`docker:dind` on a private `dind0` bridge), so its containers are *not* on the umbrel network. Reachable only via a published host port, and bind-mount data is lost when the Portainer app restarts. |
| Manual `docker run` over SSH | Works, but lives outside umbrelOS app management and needs SSH access to the box. Keep as a last resort. |
| Direct `enclave.trymaple.ai` from each app | Not used: the proxy is what handles TEE attestation and the encryption handshake. Only the proxy is documented for non-desktop clients. |

## Install

1. **Push this repo to GitHub** (public). It can be a template copy of
   `getumbrel/umbrel-community-app-store`; the only required shape is:
   `umbrel-app-store.yml` at the root + one folder per app containing
   `umbrel-app.yml` and `docker-compose.yml`.
   App IDs must start with the store ID (`yuri` → `yuri-maple-proxy`).
2. On the Umbrel: **App Store → ⋯ → Add community app store** → paste the repo URL.
3. Install **Maple Proxy** from that store. It has no UI; it just runs.
4. Create a **Maple API key** at https://trymaple.ai (this is the billing identity).
5. Install the **Hermes Agent** app (official store) and point it at the proxy:

```
model.provider  custom
model.base_url  http://yuri-maple-proxy_server_1:8080/v1
model.api_key   <your Maple API key>
model.default   kimi-k3        # or gpt-oss-120b / deepseek-v4-pro / gemma4-31b
```

## Verifying it works

From another app's shell on the Umbrel (or from the host):

```bash
curl -N http://yuri-maple-proxy_server_1:8080/v1/models \
  -H "Authorization: Bearer YOUR_MAPLE_KEY"
```

A JSON model list means the proxy is up and your key is valid. Then a streaming
completion:

```bash
curl -N http://yuri-maple-proxy_server_1:8080/v1/chat/completions \
  -H "Authorization: Bearer YOUR_MAPLE_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-oss-120b","messages":[{"role":"user","content":"ping"}],"stream":true}'
```

## Known constraints

- **Streaming only.** Maple returns streamed responses exclusively; `stream=true`
  is mandatory on every request.
- **`latest` is not pinned.** The compose pins `0.3.2` + its index digest so an
  install is reproducible. To update: change the tag/digest, bump `version:` in
  `umbrel-app.yml`, commit — umbrelOS then offers an update.
- **No MAPLE_API_KEY is baked in**, so each app passes its own key and bills its
  own account. Add the env var if you would rather have one shared default key.
- The proxy holds no state; nothing is written to `${APP_DATA_DIR}`.

## Notes on the design choice worth revisiting

`app_proxy` is included so the app behaves like a normal umbrelOS app and the
proxy is reachable from the LAN behind Umbrel's auth. Apps on the box should
still use the container name (`yuri-maple-proxy_server_1:8080`), not the LAN
URL. If you want it strictly internal, delete the `app_proxy` service — the
umbrelOS `tailscale` app proves an app can run without one.
