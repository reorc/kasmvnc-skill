# KasmVNC CDP Browser Skill

这个仓库提供一个 OpenClaw skill，让 OpenClaw 在 Clawkeeper VNC 已启用后，直接使用容器内 Chrome 的 CDP，而不是去操作 KasmVNC 网页画面。

核心约定：

- VNC 给用户人工接管使用。
- OpenClaw 自动化使用 `http://127.0.0.1:9223`。
- 两者共享 `openclaw-kasm-chrome` 容器里的同一个 Chrome profile。
- 如果实例已经配置 VNC/browser handoff，OpenClaw 必须优先使用容器内 CDP 浏览器，不应使用
  OpenClaw 原生 managed browser、默认 `openclaw` profile，或重新启动一个本地浏览器。
- 只要环境里存在 `kasm-cdp` browser profile，普通网页浏览、登录复用、表单填写、抓取、人工接管
  这类任务都应优先走 `profile="kasm-cdp"`，即使用户没有显式提到 Kasm/VNC。
- 如果默认浏览器或本地浏览器报 `No supported browser found`，应立即 fallback 到
  `profile="kasm-cdp"`，不要先切到 Playwright CLI。

## 前置条件

目标实例已经通过 Clawkeeper 启用 VNC/browser handoff 环境，并且实例内可以访问：

```bash
curl -fsS http://127.0.0.1:9223/json/version
```

正常返回里应包含 `Browser` 和 `webSocketDebuggerUrl`。

## 安装到 OpenClaw

方式一：复制到当前 workspace 的 skills 目录。

```bash
mkdir -p /path/to/openclaw-workspace/skills/kasmvnc-cdp
cp /Users/yann/opensources/kasmvnc-skill/SKILL.md /path/to/openclaw-workspace/skills/kasmvnc-cdp/SKILL.md
```

方式二：直接把这个仓库作为额外 skill 目录加载。

编辑 `~/.openclaw/openclaw.json`：

```json5
{
  skills: {
    load: {
      extraDirs: ["/Users/yann/opensources/kasmvnc-skill"]
    }
  }
}
```

保存后新开一个 OpenClaw session，或重启 gateway：

```bash
openclaw gateway restart
openclaw skills list
```

列表中应出现 `kasm_cdp_browser`。

## 配置 OpenClaw 使用容器 CDP

编辑 `~/.openclaw/openclaw.json`，加入或合并下面的 browser 配置：

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "kasm-cdp",
    profiles: {
      "kasm-cdp": {
        cdpUrl: "http://127.0.0.1:9223",
        color: "#16A34A",
        attachOnly: true
      }
    }
  }
}
```

如果 OpenClaw agent 运行在 sandbox 中，需要让 browser tool 能访问宿主机上的 CDP。推荐在调用 browser tool 时固定 `target="host"`；如需默认允许 host browser control，可在 sandbox 配置中启用对应开关：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        browser: {
          allowHostControl: true
        }
      }
    }
  }
}
```

## 使用方式

安装和配置完成后，直接让 OpenClaw 执行浏览器任务即可，例如：

```text
使用 kasm_cdp_browser 打开 https://example.com，检查当前登录状态。
```

OpenClaw 执行时应使用 browser tool 的 `profile="kasm-cdp"`。如果 agent 运行在 sandbox 中，
同时使用 `target="host"`。

任务过程中可以关闭不再需要的标签页，减少实例内存和 CPU 占用。关闭标签页通常不会清除登录态；
cookies、local storage 和 Chrome profile 会继续保留。不要关闭仍有未保存表单、支付流程、
实时会话或用户正在 VNC 中处理的人工接管页面。

当遇到登录、人机验证、2FA、扫码、验证码、授权弹窗、支付确认，或任何 Agent 无法自行完成的阻塞性步骤时，OpenClaw 必须暂停并请求人类帮助，让用户从 Clawkeeper Portal 打开同一实例的云浏览器/VNC 入口手动处理。用户完成后，OpenClaw 继续通过 CDP 使用同一个 profile 中的登录态。

登录墙、登录弹窗遮挡搜索结果、账号选择页、扫码登录页、要求继续登录才能查看内容等情况，都属于人工接管触发条件。此时不要切换到 OpenClaw Browser Relay，不要让用户接入本地 Chrome 标签页，也不要索要账号密码；应继续保留 `profile="kasm-cdp"`，让用户在 Clawkeeper Portal 的云浏览器里完成登录。

## 排障

在实例内检查：

```bash
curl -fsS http://127.0.0.1:9223/json/version
docker ps --filter name=openclaw-kasm-chrome
docker logs --tail 200 openclaw-kasm-chrome
curl -kI https://127.0.0.1:6901
systemctl status openclaw-kasm-autoheal.timer
```

常见问题：

- CDP 不通：容器未运行、健康检查失败，或 `9223` 没有映射到宿主机回环地址。
- profile 名不一致：当前标准 profile 是 `kasm-cdp`，不是旧文档里的 `kasm_cdp`。
- 默认浏览器失败：如果看到 `No supported browser found`，优先重试 `profile="kasm-cdp"` 和
  `target="host"`，不要先改用 Playwright CLI。
- 登录墙或登录弹窗挡住内容：暂停任务，让用户在 Clawkeeper Portal 云浏览器/VNC 里手动登录；
  不要改用 OpenClaw Browser Relay 或要求用户接入本地 Chrome。
- 用户在 VNC 中关闭了 Chrome：等待几秒后重试。新版 Clawkeeper 镜像会自动拉起新的可见
  Chrome；如果 Docker health 长时间异常，宿主机 autoheal timer 会重启容器兜底。
- VNC 能打开但 OpenClaw 看不到登录态：用户操作的 Chrome 与 CDP Chrome 没有使用同一个 profile。
- sandbox 内无法连接：browser tool 需要使用 `target="host"`，并允许 host browser control。
- 不要把 CDP 端口配置到 frpc、Caddy 或公网安全组；CDP 只能作为 OpenClaw 内部控制通道使用。
