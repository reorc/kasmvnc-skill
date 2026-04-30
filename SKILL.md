---
name: kasm_cdp_browser
description: Use this when a task requires OpenClaw to automate the shared Kasm Chrome browser provided by Clawkeeper VNC. The skill connects to the container-local Chrome DevTools Protocol endpoint instead of controlling the VNC web page.
metadata:
  openclaw:
    requires:
      config:
        - browser.enabled
---

# Kasm CDP Browser

Use this skill for browser automation on Clawkeeper instances where VNC has been enabled with the
`openclaw-kasm-chrome` container.

## Core Rule

Control the browser through Chrome DevTools Protocol (CDP), not through the KasmVNC web UI.

If Clawkeeper VNC/browser handoff is already configured for the instance, you must use the
container CDP browser. Do not use OpenClaw's native managed browser, the default `openclaw`
profile, or any newly launched local browser for the same task. The configured container browser is
the source of truth because it shares the VNC-visible Chrome profile and user-completed login state.

Use the OpenClaw browser tool with:

- `profile="kasm_cdp"`
- `target="host"` when the agent run is sandboxed or when routing is ambiguous

The expected CDP endpoint is:

```text
http://127.0.0.1:9223
```

The VNC URL is only for human handoff. Do not automate the VNC page, remote desktop canvas, or
KasmVNC coordinates.

## Expected Environment

The instance should already have the Clawkeeper VNC/browser handoff environment enabled:

- Docker container: `openclaw-kasm-chrome`
- KasmVNC web endpoint: `https://127.0.0.1:6901`
- Chrome CDP endpoint: `http://127.0.0.1:9223`
- Shared Chrome profile mounted from `/opt/openclaw-kasm/profile` to `/data/chrome-profile`

## Before Browser Work

If the task involves website navigation, login reuse, CAPTCHA/2FA handoff, form filling, scraping,
or browser state inspection, first verify CDP health when possible:

```bash
curl -fsS http://127.0.0.1:9223/json/version
```

Success should include a `Browser` field and a `webSocketDebuggerUrl`.

If CDP is healthy, use OpenClaw browser actions through the `kasm_cdp` profile:

- open/navigate target pages
- inspect snapshots
- click/type through DOM-aware browser actions
- capture screenshots
- continue after human handoff

## Human Handoff

When login, 2FA, QR code, CAPTCHA, consent, or payment confirmation blocks automation:

1. Pause the automation.
2. Tell the user to open the Clawkeeper Portal VNC entry for the same instance.
3. Ask the user to complete the manual step in the visible Chrome browser.
4. After the user confirms completion, continue through CDP with `profile="kasm_cdp"`.

The user and OpenClaw share the same browser profile, so cookies, local storage, and login state
should persist across handoff.

## Safety Boundaries

- Never expose the CDP URL to ordinary end users.
- Never publish or proxy `127.0.0.1:9223` through FRP, Caddy, VNC, or public routes.
- Do not ask the user to open CDP in their browser.
- Do not use the VNC URL as the automation target.
- Treat CDP as a privileged internal control channel.

## Troubleshooting

If browser automation fails before navigation:

```bash
curl -fsS http://127.0.0.1:9223/json/version
docker ps --filter name=openclaw-kasm-chrome
docker logs --tail 200 openclaw-kasm-chrome
```

Common causes:

- CDP is not reachable: the container is stopped, unhealthy, or port `9223` is not mapped.
- Login state is missing: the profile mount is wrong or the profile directory was removed.
- User sees a different browser state: Chrome processes are not using the same profile.
- Sandboxed OpenClaw cannot reach host CDP: use `target="host"` and enable host browser control in
  OpenClaw sandbox settings.
