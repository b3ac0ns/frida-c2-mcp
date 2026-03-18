Critical issues / Challenges / Research / Solutions
===================

MCP Transport to Streamable HTTP
---------------

* Challenge: A2A Framework will likely work better with streamable HTTP than stdio implementation, this way we can have multiple devices

  * In an A2A mcp tutorial here: https://github.com/Tsadoq/a2a-mcp-tutorial - "If you try to spawn multiple STDIO-based MCP servers, you will notice that they will not work at the same time. To solve this problem, we can use an SSE-based MCP server with some tricks."

  * SSE has shifted to streamable HTTP as seen here: https://brightdata.com/blog/ai/sse-vs-streamable-http

* Solution - Streamable HTTP was implemented

Running Frida _client_ on Android architecture
---------------
* Challenge: Compiling the frida proved to be difficult directly on Android devices

  * Used this as a reference but was not able to follow the flow properly - https://github.com/frida/frida/discussions/2411

  * `make` was failing to build. Essentially running into this issue here: https://github.com/frida/frida/issues/2864

* Solution - Hijacked the internal toolchain by swapping out broken binaries with Termux-native versions

Running Frida scripts with 17 (Android)
--------------

* Challenge: Running scripts on Frida 17 has some level of requirement of frida-compile and minification, but frida-compile does not run and cannot install frida-compile through node

  * Java not defined is a known error here: https://github.com/frida/frida/issues/3473 , basically just need to minify the script

* Initial solution: Used `esbuild` as a lightweight bundler since `frida-compile` couldn't be installed via node on Android. Implemented a temporary file write from the users payload and added an import statement for java if one was not found. My logic is that folks will still attempt to use frida scripts compatible with 16 but not 17

* Investigated alternative: `frida-tools` can be installed over the base `frida` library (`pip install frida-tools`), which provides `frida-compile`. However, `frida-compile` uses V8's JIT compiler, which requires W^X memory mappings. On Termux, the process runs under the `untrusted_app` SELinux domain, which blocks these mappings — causing a SIGSEGV (return code -11) on any non-trivial script. The same files compile fine via `adb shell` (which runs under the `magisk` SELinux domain) or on macOS. A `su -c` workaround is possible but prompts the user for superuser rights on every compilation, which is unacceptable UX. `esbuild` is a statically linked Go binary with no JIT, so it runs cleanly in any SELinux context — it remains the bundler for Android.

iOS: Compiling a full Python 3.12 toolchain for rootless jailbreak
--------------

* Challenge: Nothing exists for running a modern Python + Frida client on rootless jailbroken iOS. No Python 3.12 in Procursus, no prebuilt wheels for any of the native dependencies that FastMCP requires.

* Solution: Cross-compiled the entire stack from macOS targeting `arm64-apple-ios` (rootless layout at `/var/jb/`):
  * **Python 3.12.5** — based on [k1tty-xz/python3.12-ios-arm64](https://github.com/k1tty-xz/python3.12-ios-arm64), which targeted rootful jailbreaks. Modified extensively for rootless jailbreak support (`/var/jb/` prefix, different library paths, updated signing). Built as shared lib with `--enable-shared`, packaged as `.deb`
  * **OpenSSL 3.x** and **libffi** — built as dependencies for Python and cffi
  * **frida-python 17.6.0** — Meson build with rpath patching and `install_name_tool` fixups
  * **pydantic-core** and **rpds-py** — Rust/PyO3 via maturin, targeting `aarch64-apple-ios`
  * **cryptography** — Rust via maturin + custom OpenSSL build; needed a fix for SIGILL on A11 (iPhone X)
  * **cffi** — linked against iOS-built libffi
  * **pyyaml** — C extension against cross-compiled libyaml
  * **lupa** — Lua 51-54 extensions via setuptools

* Key pain points:
  * Rust targets (`aarch64-apple-ios`) required careful SDK/sysroot configuration for each maturin build
  * `install_name_tool` and `ldid` (code signing) were required post-build for every native extension
  * The cryptography package caused SIGILL crashes on A11 due to arm64e assumptions — had to pin build flags
  * frida-python needed rpath patching to find `libpython3.12.dylib` at the rootless jailbreak path

iOS: frida-swift-bridge decision
--------------

* Challenge: Initially attempted to support `frida-swift-bridge` for Swift runtime introspection on iOS

* Decision: Dropped it. frida-swift-bridge is effectively alpha. The compilation burden was not worth the minimal coverage it would provide.

* Result: iOS server (`ios.py`) supports only `frida-objc-bridge`. Should cover the vast majority of real-world iOS pentesting use cases.

Fixing one-shot script execution (Android + iOS)
--------------

* Challenge: After fixing the `on_message` handler to resolve `result_future` on `agent:result` instead of `agent:ready`, Android one-shot scripts started failing with `TypeError: not a function`.

* Root cause: The original code masked this bug by resolving the future on `agent:ready` (which fires before user code runs), so execution errors were never surfaced. The real problem was that the `wrapped_code` template used `Java.perform({ onComplete: ..., onError: ... })` — the object-form API. When `esbuild --bundle` inlines the npm `frida-java-bridge` package, that object form isn't supported. Only `Java.perform(function() { ... })` works.

* Solution: Rewrote `wrapped_code` to use `Java.perform(function() { ... })`. This also brought the Android template in line with iOS (which uses `if (ObjC.available) { ... }` — no object-form API either).

Fixing get_session_messages (Android + iOS)
--------------

* Challenge: `get_session_messages` always returned an empty list, regardless of script output.

* Root cause (multi-part):
  1. `on_message` crashed on string payloads (e.g. raw `console.log` output) because it called `payload.get('type')` without checking `isinstance(payload, dict)` first.
  2. One-shot scripts had no mechanism to capture `console.log` output — it went to the terminal but never into any data structure the MCP response could return.
  3. Persistent scripts also had no mechanism to route `console.log` output into `_script_messages`.

* Solution:
  - Added `isinstance(payload, dict)` guard in `on_message`.
  - For one-shot scripts: injected a `console.log` shim that captures output into a `_console_output` array, returned directly in the `agent:result` payload. One-shot results are self-contained — no need to call `get_session_messages`.
  - For persistent scripts: injected a `console.log` shim that routes output via `send({ type: 'agent:log', message: msg })`, which `on_message` appends to `_script_messages[session_id]`. Agents retrieve these via `get_session_messages`.

Open issues
=================

OpenSSL on iOS
---------

* The OpenSSL build for iOS isn't fully correct — non-breaking for now since the MCP server doesn't depend on SSL directly, but worth revisiting if a future dependency needs a working SSL context on-device.

Future plans
=================

* **Agent scaffolding:** Pivot from manual A2A implementation to providing MCP server configs and agent definitions for [Gemini CLI](https://github.com/google-gemini/gemini-cli) and [Claude Code](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview). Both tools are building native multi-agent orchestration and MCP support, which aligns with the project's direction of composing multiple MCP servers into automated workflows. [5ire](https://github.com/nanbingxyz/5ire) was used extensively during development and remains a solid GUI option, but the multi-agent orchestration vision requires CLI-native tools. The framework doesn't need its own orchestration layer — existing AI agents handle that.

* **Jadx-MCP integration:** Pair FridaC2MCP (dynamic) with [Jadx-MCP](https://github.com/pari-27/jadx-mcp) (static analysis) so an agent can decompile, identify targets, and hook them live in a single workflow.

* **FastMCP v3 migration:** Pinned to FastMCP 2.x on Android and iOS. FastMCP 3.x moved constructor kwargs (`json_response`, `stateless_http`) to `run_http_async()` and more — migration is straightforward but hasn’t been prioritized vs. upstream release cadence.

* Need to implement session management

* Need to implement any level of security XD