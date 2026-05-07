---
name: kasm_cdp_browser
description: Prefer this for browser automation whenever a Clawkeeper instance has the `kasm-cdp` browser profile, VNC/Cloud Browser is enabled, the user mentions Kasm/VNC/shared browser/cloud browser, or the default/local browser profile fails. Use the shared Kasm Chrome through container-local CDP instead of launching Playwright, OpenClaw's native browser, or a separate local browser.
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

On Clawkeeper instances with a configured `kasm-cdp` browser profile, treat Kasm CDP as the default
browser backend for website navigation, login reuse, scraping, form filling, and human handoff
flows. Do this even when the user only says "open a website" and does not explicitly mention
Kasm, VNC, or Cloud Browser.

Control the browser through Chrome DevTools Protocol (CDP), not through the KasmVNC web UI.

If Clawkeeper VNC/browser handoff is already configured for the instance, you must use the
container CDP browser. Do not use OpenClaw's native managed browser, the default `openclaw`
profile, or any newly launched local browser for the same task. The configured container browser is
the source of truth because it shares the VNC-visible Chrome profile and user-completed login state.

Use the OpenClaw browser tool with:

- `profile="kasm-cdp"`
- `target="host"` when the agent run is sandboxed or when routing is ambiguous

The expected CDP endpoint is:

```text
http://127.0.0.1:9223
```

The VNC URL is only for human handoff. Do not automate the VNC page, remote desktop canvas, or
KasmVNC coordinates.

## Browser Backend Priority

For browser tasks in this environment:

1. Use the OpenClaw browser tool with `profile="kasm-cdp"` and `target="host"` when the
   `kasm-cdp` profile exists or CDP at `http://127.0.0.1:9223` is reachable.
2. If the default `openclaw` or local browser profile fails with errors such as "No supported
   browser found", immediately retry with `profile="kasm-cdp"` before considering Playwright CLI or
   another browser backend.
3. Use other profiles only when the user explicitly asks for a different browser environment or the
   task requires a non-Kasm browser state.
4. Treat Playwright CLI as a fallback for environments without a configured OpenClaw browser
   profile, not as the first choice on Clawkeeper VNC instances.

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

If CDP is healthy, use OpenClaw browser actions through the `kasm-cdp` profile:

- open/navigate target pages
- inspect snapshots
- click/type through DOM-aware browser actions
- capture screenshots
- continue after human handoff

If CDP health is unknown but the `kasm-cdp` browser profile is available, try that profile before
launching any separate browser. A configured browser profile is a stronger signal than generic
browser-task wording.

## Tab Management

Close tabs that are no longer needed for the current task. The Kasm Chrome profile persists cookies,
local storage, and login state on disk, so closing an unused tab does not normally sign the user out
or remove saved session state. Keeping only relevant tabs open reduces memory and CPU pressure on
the instance.

Before closing a tab, make sure it is not holding unsaved form input, an active checkout/payment
step, a live chat/session, or a human handoff page that the user still needs to see in VNC.

## Human Handoff

When login, human verification, 2FA, QR code, CAPTCHA, consent, payment confirmation, or any other
blocking step cannot be completed by the agent itself, you must request human help instead of trying
to bypass, guess, or continue blindly.

Login walls, login popups, search results hidden behind a login modal, account selection screens,
QR-code login prompts, and "continue in app/browser" blocks are all human handoff triggers. When any
of these appear, do not switch to Browser Relay, do not ask the user to attach a local Chrome tab,
and do not ask for credentials. Keep using the same `kasm-cdp` browser profile and ask the user to
complete the login in the Clawkeeper Portal Cloud Browser/VNC window.

1. Pause the automation.
2. Tell the user to open the Clawkeeper Portal Cloud Browser/VNC entry for the same instance.
3. Ask the user to complete the blocking step manually in the visible Kasm Chrome browser.
4. After the user confirms completion, continue through CDP with `profile="kasm-cdp"`.

The user and OpenClaw share the same browser profile, so cookies, local storage, and login state
should persist across handoff.

## Safety Boundaries

- Never expose the CDP URL to ordinary end users.
- Never publish or proxy `127.0.0.1:9223` through FRP, Caddy, VNC, or public routes.
- Do not ask the user to open CDP in their browser.
- Do not ask the user to use OpenClaw Browser Relay or attach a local Chrome tab when the
  `kasm-cdp` profile is available.
- Do not use the VNC URL as the automation target.
- Treat CDP as a privileged internal control channel.

## Troubleshooting

If browser automation fails before navigation:

```bash
curl -fsS http://127.0.0.1:9223/json/version
docker ps --filter name=openclaw-kasm-chrome
docker logs --tail 200 openclaw-kasm-chrome
systemctl status openclaw-kasm-autoheal.timer
```

Common causes:

- CDP is not reachable: the container is stopped, unhealthy, or port `9223` is not mapped.
- Profile naming mismatch: the browser profile is `kasm-cdp`. Do not use the legacy
  `kasm_cdp` profile name unless the browser tool explicitly lists it.
- Default browser failed: if the error says no supported local browser was found, retry with
  `profile="kasm-cdp"` and `target="host"` before switching to Playwright CLI.
- The user closed Chrome in VNC: wait a few seconds and retry CDP. The Clawkeeper image should
  restart the visible Chrome process automatically; the host autoheal timer restarts the container
  if Docker health remains bad.
- Login state is missing: the profile mount is wrong or the profile directory was removed.
- User sees a different browser state: Chrome processes are not using the same profile.
- Sandboxed OpenClaw cannot reach host CDP: use `target="host"` and enable host browser control in
  OpenClaw sandbox settings.
