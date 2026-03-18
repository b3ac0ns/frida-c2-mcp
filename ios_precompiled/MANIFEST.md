# iOS Precompiled Packages

Precompiled Python 3.12 runtime and native wheels for rootless jailbroken iOS (arm64).

These were cross-compiled from macOS targeting `aarch64-apple-ios` with a rootless jailbreak layout (`/var/jb/`). See `setup.md` for full installation instructions.

## Contents

Place the following files in this directory:

| File | Source |
|------|--------|
| `python3.12_3.12.5-1_iphoneos-arm64.deb` | Python 3.12.5 runtime (based on [k1tty-xz/python3.12-ios-arm64](https://github.com/k1tty-xz/python3.12-ios-arm64), modified for rootless) |
| `frida-17.6.0-*.whl` | Frida Python bindings |
| `pydantic_core-2.41.5-*.whl` | Required by FastMCP / pydantic |
| `cryptography-47.0.0.dev1-*.whl` | Dependency chain; includes SIGILL fix for A11 |
| `cffi-2.0.1.dev0-*.whl` | Required by cryptography |
| `PyYAML-7.0.0.dev0-*.whl` | Required by FastMCP deps |
| `rpds_py-0.30.0-*.whl` | Required by FastMCP deps |
| `lupa-2.6-*.whl` | Lua embedding |

## After installing wheels

All native extensions (`.so` files) must be signed with `ldid` after installation:

```bash
find /var/jb/var/mobile/.local/lib/python3.12/site-packages -name "*.so" -type f -exec ldid -S {} \;
```

## License

The Python 3.12 `.deb` is based on k1tty-xz's work under the MIT license. Individual wheel licenses follow their upstream projects.
