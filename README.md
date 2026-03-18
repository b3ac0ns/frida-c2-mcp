# FridaC2MCP

An MCP server with streamable HTTP transport that runs Frida's dynamic instrumentation **entirely on-device** — no client-side tooling required.

[![TLDR Featured](https://img.shields.io/badge/Featured%20in-TLDR%20InfoSec-blue)](https://tldr.tech/infosec/2026-01-30)

**This project runs directly on rooted Android and jailbroken iOS devices**, exposing Frida as a set of tools callable over HTTP. Any MCP-compatible client can connect to a device on the network and perform dynamic instrumentation without installing any Frida tooling locally.

**Going forward, development is targeting [Gemini CLI](https://github.com/google-gemini/gemini-cli) and [Claude Code](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview)** as primary clients, since both are building native multi-agent orchestration and MCP support — which aligns with this project's direction of composing multiple MCP servers (e.g. FridaC2MCP + Jadx-MCP) into automated workflows. [5ire](https://github.com/nanbingxyz/5ire) was used extensively during early development and remains a great GUI option for interacting with the server.

**Gemini CLI — register the device MCP server** (replace `<DEVICE_IP>` with your phone’s LAN address):

```bash
gemini mcp add --transport http frida-c2-mcp http://<DEVICE_IP>:6767/mcp
```

**Android Demo**

https://github.com/user-attachments/assets/7cee77c5-ed40-4797-b6b5-3edb5fdd03ce


**iOS Demo**

https://github.com/user-attachments/assets/065955cb-cce5-41b7-aba4-f5d8b94bd057


## Core Functionality

- **Process & Application Management:** List installed applications, enumerate running processes, and get information about the foreground application.
- **Application Control:** Start and terminate applications by their identifier or PID.
- **Dynamic Instrumentation:** Create interactive sessions by attaching to running processes.
- **Remote Script Execution:** Execute custom Frida (JavaScript) scripts within an attached session. Scripts are compiled on-the-fly to ensure compatibility with Frida 17+. This supports both one-shot scripts and persistent hooks (`keep_alive=True`).

## Intended Use Case

The goal is fully automated, agent-driven mobile security testing. By exposing Frida as an MCP server, a pentester or an AI agent can inspect and manipulate an application's behavior without a direct USB connection or local shell access after initial setup. The client needs zero Frida tooling — everything runs on the target device.

A typical workflow:
1. Point your AI agent (Claude Code, Gemini CLI, etc.) at the device's MCP endpoint.
2. The agent starts a target application, attaches a Frida session, and injects hooks.
3. Security controls (root/jailbreak detection, SSL pinning, etc.) are bypassed dynamically.
4. The agent observes results and iterates — all without human intervention.

This extends naturally to multi-device setups: run multiple phones on the same network, each with its own MCP server, and orchestrate them from a single agent.

## Platform Support

### Android (`cli.py`)

Runs in Termux on rooted Android. Uses `esbuild` for on-the-fly script bundling and `frida-java-bridge` for Java/Android runtime instrumentation. While `frida-compile` is technically available via `frida-tools`, it cannot be used on Termux because Android's `untrusted_app` SELinux domain blocks the W^X memory mappings required by V8's JIT compiler — causing a SIGSEGV on any non-trivial script. `esbuild` is a statically linked Go binary with no JIT, so it runs cleanly in any SELinux context.

### iOS (`ios.py`)

Runs on rootless jailbroken iOS (tested on iPhone X, iOS 16.7.12 with palera1n). Uses `frida-compile` for script bundling and `frida-objc-bridge` for Objective-C runtime instrumentation. **Node.js is not required on-device**.

The iOS port required custom-compiling an entire Python 3.12 toolchain for arm64, since none of these packages were available for rootless jailbroken iOS:

| Package | Version | Build System | Why |
|---------|---------|-------------|-----|
| **Python** | 3.12.5 | configure + Make | Base runtime — not available via Procursus |
| **frida-python** | 17.6.0 | Meson + setuptools | Core Frida bindings |
| **pydantic-core** | 2.41.5 | maturin (Rust/PyO3) | Required by FastMCP |
| **cryptography** | 47.0.0.dev1 | maturin (Rust) + OpenSSL 3.x | Dependency chain; needed SIGILL fix for A11 |
| **cffi** | 2.0.1.dev0 | setuptools + libffi | Required by cryptography |
| **pyyaml** | 7.0.0.dev0 | setuptools + libyaml | Required by FastMCP deps |
| **rpds-py** | 0.30.0 | maturin (Rust/PyO3) | Required by FastMCP deps |
| **lupa** | 2.6 | setuptools + Lua | Lua embedding |

Precompiled wheels and the Python `.deb` are included in `ios_precompiled/` so you don't have to build them yourself. See `setup.md` for iOS installation instructions.

## Architecture & Design Rationale

-   **On-Device Execution:** Both the Frida server and client run on the target device. The MCP server translates HTTP requests into Frida commands. This is a departure from the traditional model where a Frida client runs on a separate computer.

-   **Streamable HTTP Transport:** Uses streamable HTTP instead of stdio to support multiple concurrent connections and multi-device orchestration.

-   **On-Device Script Bundling:** Frida 17+ requires bundled JavaScript. On Android, `esbuild` handles bundling (it runs in any SELinux context without JIT). On iOS, `frida-compile` is used with ObjC bridge imports injected automatically.

-   **Agent-First Design:** Built to be consumed by AI agents rather than human operators. The MCP tool interface is designed so that an LLM can autonomously navigate a full pentesting workflow — spawn apps, attach sessions, write and inject hooks, read results — in a single conversation.

## Vision

FridaC2MCP is scaffolding for an automated mobile pentesting framework. The idea is to compose multiple specialized MCP servers — for example: FridaC2MCP for dynamic instrumentation, [Jadx-MCP](https://github.com/pari-27/jadx-mcp) for static analysis — and let AI agents orchestrate the full workflow. Decompile an APK, identify interesting methods, then hook them live. No manual tool-switching, no copy-pasting between terminals.

This pairs naturally with Gemini CLI and Claude Code, which already support multi-tool MCP workflows natively. The framework doesn't need to implement its own agent orchestration — it just needs to provide the right MCP servers and let existing agents do what they're good at.

## Known issues

- **iOS OpenSSL / SSL:** The Python 3.12 build’s OpenSSL integration is not fully correct. This has not blocked the MCP server (which does not rely on HTTPS for its own transport), but it may matter if a future dependency needs a working TLS context on-device. See `devlog.md` for detail.

- **FastMCP 2.x only:** FastMCP 3.x introduced breaking API changes. **Android and iOS** both pin below 3.x. Install via `requirements-android.txt` or `requirements-ios.txt` — do not upgrade to an unpinned `fastmcp` until this repo migrates.

## Acknowledgments

This project is a networked evolution of the original [FridaMCP](https://github.com/dnakov/frida-mcp) by Daniel Nakov. While the original project pioneered the use of Frida as an MCP server over stdio, **FridaC2MCP** adapts this concept for remote, multi-device C2 environments using Streamable HTTP and modern Frida 17 bundling.

The iOS Python 3.12 runtime is based on [python3.12-ios-arm64](https://github.com/k1tty-xz/python3.12-ios-arm64) by k1tty-xz. The original project targeted rootful jailbreaks — significant modifications were required to adapt it for rootless jailbreak environments (palera1n, `/var/jb/` layout).

---

### **⚠️ Disclaimer**

This is a proof-of-concept. It lacks proper session management, graceful error handling, and **any form of security**. All communication is unencrypted and unauthenticated. **Use at your own risk** and only in secure, isolated network environments.
