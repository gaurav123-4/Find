# Desktop Framework ADR: Tauri vs Electron

**Status:** Decided (Tauri Primary, Electron Evaluated as Fallback)

**Date:** 2026-05-14

**Context:** Evaluate Electron as a fallback option to Tauri for Find's desktop client. Goal is not to switch immediately, but to document tradeoffs and define conditions that would trigger a switch.

---

## 1. Background

Find is currently a web application with a Docker-based stack (Next.js frontend, FastAPI backend, ML pipeline). A desktop client is planned to enable local-first processing without Docker dependency.

Tauri was selected as the primary framework for its smaller footprint and native performance. However, three scenarios could favor Electron:

1. **Node-based orchestration** — The FastAPI backend and ML worker require Python + Redis + RQ. Electron can bundle Node tooling more naturally than Tauri for sidecar process management.
2. **Auto-update tooling maturity** — Tauri 2.x updater is newer. Electron's electron-updater is battle-tested with GitHub Releases and S3.
3. **Static export constraints** — Tauri requires Rust compilation per target. Electron works with standard Next.js builds without build target complexity.

---

## 2. Quantitative Tradeoffs

### App Size

| Framework | Packaged Size | Notes |
|-----------|--------------|-------|
| Tauri 2.x | 10-15 MB | WebView2 included on Windows (30-50MB if bundling WebView2 runtime) |
| Electron 33+ | 80-150 MB | Chromium + Node.js bundled; no external runtime deps |

**Real benchmarks:**

- Tauri hello world: ~8 MB Windows installer (~3 MB compressed)
- Electron hello world: ~100 MB Windows installer (~50 MB compressed)
- Find's Next.js frontend build: ~2-5 MB (assets only)

**Startup Time**

| Framework | Cold Start | Warm Reload |
|-----------|-----------|------------|
| Tauri | 50-150 ms | < 20 ms |
| Electron | 200-500 ms | 100-300 ms |

**Why the gap:**

- Tauri uses OS-native WebView2 (Windows) / WebKit (macOS/Linux)
- Electron bundles full Chromium + V8 engine
- Tauri runtime is ~1 MB; Electron runtime is ~50-70 MB

### Memory Usage

| Framework | Idle Memory | Under Load |
|-----------|-------------|------------|
| Tauri | 30-60 MB | 80-150 MB |
| Electron | 100-200 MB | 200-400 MB |

For Find, this matters: ML processing is memory-heavy already (YOLOv10 ~150MB, Florence-2 ~300MB). Saving 100-200 MB at the framework level leaves more headroom for model inference.

---

## 3. Comparison Matrix

### 3.1 Installer Support

| Criterion | Tauri 2.x | Electron |
|-----------|-----------|----------|
| Windows | MSI, NSIS, WiX | NSIS, MSI (via electron-builder), Squirrel.Windows |
| macOS | DMG, App Bundle | DMG, PKG, App Store |
| Linux | AppImage, Deb, RPM | AppImage, Deb, RPM, Flatpak |
| Signing | Requires Apple Developer account + notarization | Same requirements + notarization |
| Distribution | Self-hosted or Tauri Updates server | Self-hosted, GitHub Releases, S3, generic server |

**Analysis:**
Both frameworks support all three major platforms. Tauri uses platform-native tools (WiX for Windows, DMG for macOS, etc.) which produce more idiomatic installers. Electron's electron-builder normalizes the experience across platforms but produces larger installers.

**Verdict:** Tie, with slight edge to Tauri for smaller installer sizes.

### 3.2 Auto-Update Maturity

| Criterion | Tauri 2.x | Electron |
|-----------|-----------|----------|
| Update mechanism | Tauri Updater plugin (built-in) | electron-updater (electron-builder) |
| Update sources | GitHub Releases, S3, custom endpoint | GitHub Releases, S3, generic HTTP server |
| Delta updates | Supported (built-in) | Supported via electron-updater |
| Rollback | Manual via version pinning | Built-in with electron-updater |
| Maturity | ~2 years in production (2.x newer, < 1 year) | ~8 years in production |
| Signed updates | Required | Required |
| Self-update UX | Built-in UI components | Built-in API (autoUpdater) |

**Analysis:**
Electron's auto-update is battle-tested at scale (VS Code, Slack, Discord, etc.). Tauri 2.x updater works but has fewer production case studies. Edge cases like network interruption during update, corrupted downloads, and rollback scenarios are better documented for Electron.

**Gaps for Find with Tauri:**

- No built-in update notification UI (must build custom)
- Update server configuration requires self-hosting or Tauri Cloud
- Less community knowledge for troubleshooting update failures

**Verdict:** Electron. Auto-update is a production-critical feature for desktop apps; Electron's tooling has orders of magnitude more production hours.

### 3.3 Process Management for Sidecars

**Context:** Find requires FastAPI backend + ML worker running as sidecar processes. These communicate via Redis and RQ.

| Criterion | Tauri 2.x | Electron |
|-----------|-----------|----------|
| Child process spawning | Rust `Command` API | Node.js `child_process` |
| Process lifecycle management | Built-in Rust APIs | Node.js APIs |
| IPC with sidecars | Rust channels → JSON-RPC via Tauri commands | Node.js IPC + native messaging |
| Process monitoring | Manual + Rust async | Node.js built-in monitoring |
| Exit handling | Rust signal handlers | Node.js process events |
| Cross-platform consistency | Good (Rust stdlib) | Good (Node.js cross-platform) |
| Sidecar bundling | Python binary must be bundled separately | Python sidecar works same as any child process |

**Analysis:**
Both frameworks can manage sidecar processes. Key differences:

- Tauri sidecar API (Tauri 1.x) was limited; Tauri 2.x improved this with `tauri-plugin-shell`
- Electron's Node.js heritage makes it more natural for managing processes that use Node tooling
- Python sidecars work fine on both, but bundling/distributing Python with Tauri requires additional tooling

**For Find's specific architecture:**

1. FastAPI runs as sidecar → serves API on `localhost:8000`
2. ML worker runs as sidecar → connects to Redis, processes RQ jobs
3. Desktop app (Tauri/Electron) → makes HTTP calls to FastAPI sidecar

This pattern is straightforward on both frameworks. Electron has slightly better tooling for managing multiple processes and their interdependencies.

**Verdict:** Slight edge to Electron for process management maturity, but Tauri 2.x is sufficient for Find's needs.

### 3.4 Security Hardening

| Criterion | Tauri 2.x | Electron |
|-----------|-----------|----------|
| CSP headers | Configurable | Configurable |
| Context isolation | Default (browser-side) | Default |
| Node integration | N/A (no Node in renderer) | Disabled by default in Electron 12+ |
| Remote module | N/A | Removed in Electron 12+ |
| Sandbox | Enabled by default | Enabled by default |
| Preload scripts | Typed IPC between renderer and main | Context bridge API |
| Security audit tooling | Limited | electron-security-auditor, npm audit |
| Attack surface | Smaller (no bundled Chromium/Node in renderer) | Larger (full browser runtime exposed if misconfigured) |

**Analysis:**
Security hardening is where Tauri's architecture provides inherent advantages:

- No Node.js in renderer → fewer attack vectors
- No Chromium in separate process → smaller attack surface
- Rust backend → memory safety by default

Electron has improved significantly (Electron 12+ disabled Node integration by default, added context isolation), but misconfiguration risks remain. For apps handling user data (like Find's image library), Tauri's default-deny architecture is preferable.

**Verdict:** Tauri. Better default security posture with minimal configuration.

### 3.5 Fit with Current Next.js App

| Criterion | Tauri 2.x | Electron |
|-----------|-----------|----------|
| Next.js integration | Works with `next export` or custom build | Works with standard Next.js build |
| Static export constraint | Requires `output: 'export'` or Tauri build pipeline | No export constraints |
| Next.js features supported | All except server-side features (API routes, SSR) | All features work (SSR disabled in packaged app) |
| Build pipeline complexity | Higher (must configure Tauri build separately) | Lower (standard Next.js + electron-builder) |
| Hot reload in development | Tauri CLI + Next.js dev server | Electron with custom reload tooling |

**Analysis:**
Find's frontend uses Next.js with API routes for upload/search functionality. In the desktop context:

- API routes would move to the FastAPI sidecar
- Client-side rendering is already the primary mode
- No SSR dependency in core features

Tauri requires either:

1. `output: 'export'` for static export (loses some Next.js features)
2. Custom build pipeline that runs Next.js build, then Tauri build

Electron can use standard Next.js build output with minimal configuration.

**Verdict:** Slight edge to Electron for Next.js integration simplicity.

---

## 4. Decision Criteria

Use Electron over Tauri when **any** of these conditions are met:

### Hard Triggers (should switch immediately)

1. **Auto-update is mandatory before first stable release** and Tauri updater cannot meet the timeline
   - If Find needs auto-update working within 2 weeks and Tauri updater setup has blocking issues
2. **Python bundling for sidecars becomes a blocker**
   - If packaging Python + FastAPI for Tauri proves significantly harder than Electron
3. **Team lacks Rust expertise** and critical Tauri-specific bugs arise
   - Rust debugging is non-negotiable for time-critical fixes

### Soft Triggers (evaluate and decide)

1. **Static export limitation blocks a critical Next.js feature**
   - e.g., `next/image` with external domains, dynamic routes, etc.
2. **Auto-update production issues exceed threshold**
   - If Tauri updater causes >5% of users to have update failures in beta
3. **Sidecar process management complexity exceeds estimate**
   - If Tauri-sidecar IPC causes significant engineering overhead

### Tauri Stays

1. **App size matters** — if size impact on user adoption is significant
2. **Security is prioritized** — if users store sensitive images locally
3. **Performance under ML load matters** — saving 100-200 MB framework overhead helps

---

## 5. Architecture Implications

### For Tauri

```
Desktop App (Tauri)
├── WebView2 (Windows) / WebKit (macOS/Linux)
├── Frontend: Next.js static build
├── Sidecar: FastAPI + Python (bundled Python executable)
│   └── RQ + Redis embedded or lightweight
└── IPC: Tauri commands ↔ Rust ↔ FastAPI
```

### For Electron

```
Desktop App (Electron)
├── Chromium + Node.js
├── Frontend: Next.js build (standard, no export required)
├── Sidecar: FastAPI + Python (child process)
│   └── RQ + Redis embedded
└── IPC: Context bridge ↔ preload ↔ main process ↔ FastAPI
```

**Key difference:** Tauri requires bundling Python; Electron treats FastAPI as any other child process.

---

## 6. Outstanding Questions

1. **Auto-update server**: Will Find self-host an update server, use Tauri Cloud, or use GitHub Releases?
   - Tauri updater needs a server; Electron's GitHub Releases integration requires only a token
2. **Python bundling**: Can we package FastAPI + ML worker as a single executable for Tauri?
   - Tools: PyInstaller, Nuitka, or embedded Python in Rust
3. **ML model distribution**: Will ML models be bundled in the installer or downloaded on first run?
   - Bundling: larger installer, offline-first
   - Download: smaller installer, requires first-run network
4. **Sidecar architecture finalization**: Is embedded Redis + RQ realistic for desktop, or should the sidecar connect to a hosted Redis?
   - Embedded: no external deps, but adds complexity
   - Hosted: simpler app, but requires external service

---

## 7. Recommendation

**Start with Tauri.** Reasons:

1. Size and performance advantages are significant for a local-first app
2. Security posture is inherently better without browser runtime in renderer
3. Tauri 2.x has matured to production readiness
4. Find's architecture (FastAPI sidecar) works equally well on both

**Switch to Electron if:**

- Auto-update setup exceeds 1 week of engineering effort on Tauri
- Python bundling causes blocker-level issues
- Critical Tauri-specific bugs cannot be resolved in reasonable time

**Do not implement Electron concurrently.** This ADR documents the decision criteria. If conditions trigger a switch, implement Electron as a replacement, not a parallel track.

---

## 8. Implementation Notes (if switching to Electron)

- Use `electron-builder` for packaging (mature, well-documented)
- Use `electron-updater` for auto-update (GitHub Releases source recommended)
- Next.js build: package the standard `next build` output from `.next/` (or `.next/standalone` when using `output: "standalone"`)
- FastAPI sidecar: spawn as child process, wait for health check before exposing to renderer
- IPC: use context bridge pattern (Electron 12+ default secure pattern)
- Security: enable `contextIsolation`, `sandbox`, `nodeIntegration: false`

---

## 9. Related

- [Discussion #37](https://github.com/Abhash-Chakraborty/Find/discussions/37)
- Docker compose (existing backend architecture): `docker-compose.yml`
- Light contributor mode for UI-only work: `docker-compose.light.yml`
