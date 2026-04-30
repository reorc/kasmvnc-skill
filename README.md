# KasmVNC CDP Browser Skill

这个仓库提供一个 OpenClaw skill，让 OpenClaw 在 Clawkeeper VNC 已启用后，直接使用容器内 Chrome 的 CDP，而不是去操作 KasmVNC 网页画面。

核心约定：

- VNC 给用户人工接管使用。
- OpenClaw 自动化使用 `http://127.0.0.1:9223`。
- 两者共享 `openclaw-kasm-chrome` 容器里的同一个 Chrome profile。

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
    defaultProfile: "kasm_cdp",
    profiles: {
      kasm_cdp: {
        cdpUrl: "http://127.0.0.1:9223",
        color: "#16A34A"
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

当遇到登录、2FA、扫码、验证码或授权弹窗时，OpenClaw 应暂停并让用户从 Clawkeeper Portal 打开同一实例的 VNC 入口完成操作。用户完成后，OpenClaw 继续通过 CDP 使用同一个 profile 中的登录态。

## 排障

在实例内检查：

```bash
curl -fsS http://127.0.0.1:9223/json/version
docker ps --filter name=openclaw-kasm-chrome
docker logs --tail 200 openclaw-kasm-chrome
curl -kI https://127.0.0.1:6901
```

常见问题：

- CDP 不通：容器未运行、健康检查失败，或 `9223` 没有映射到宿主机回环地址。
- VNC 能打开但 OpenClaw 看不到登录态：用户操作的 Chrome 与 CDP Chrome 没有使用同一个 profile。
- sandbox 内无法连接：browser tool 需要使用 `target="host"`，并允许 host browser control。
- 不要把 CDP 端口配置到 frpc、Caddy 或公网安全组；CDP 只能作为 OpenClaw 内部控制通道使用。
