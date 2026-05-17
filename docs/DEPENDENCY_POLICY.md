# Dependency Management & Vulnerability Policy

Find has both a Node.js frontend and Python backend — they update differently, so we need different rules for each. The goal: ship fast, stay secure, avoid the disasters that happen when you blindly adopt code written by strangers on the internet.

## Frontend: `pnpm` (The Wild West)

npm packages move *fast*. Someone publishes a new version every 5 minutes, and not all of them are legitimate. A malicious actor can slip a package with a supply-chain attack past initial review, especially if everyone adopts it immediately. **We don't.**

**The actual rules:**

- **Lockfile is law:** `pnpm-lock.yaml` is committed and exact. No floating versions, no "latest", nothing vague. This way we all run the same code in CI and locally.
- **7-day embargo on routine updates:** For minor/patch bumps of non-critical packages, we wait a week after publish. This lets the community find and flag sketchy code before we pull it in. Yes, 7 days feels conservative — it's supposed to. (Real story: a popular npm package was compromised in 2024, and it was caught within 48 hours because people noticed the new version doing weird stuff.)
- **Automated CI reporting:** `pnpm audit --audit-level=high` runs on every PR. We currently run this in report-only mode to prevent blocking unrelated PRs, but any new high/critical vulns must be resolved.

**The exception:**
If a high or critical CVE drops for something we actually use (e.g., `react`, `next`, `axios`), the embargo is *waived*. We patch immediately, get CI to pass, and merge it. There's no point in following the waiting period if the package is already known-dangerous.

## Backend: `uv` + Python (Reproducibility Matters)

Python's dependency resolver is deterministic when you lock it. Unlike npm's peer-dependency chaos, we can actually guarantee "if it works here, it works everywhere." But that also means we can't just yolo-update everything — we need to test.

**The actual rules:**

- **Strict lock via `uv.lock`:** Everything is resolved and frozen in `uv.lock`. No floating versions in `pyproject.toml` like `"fastapi>=0.100"`. When you run `uv sync`, you get *exactly* what's in the lock file, no surprises.
- **Test GPU & mock modes locally before bumping major libs:** If you're bumping FastAPI, CLIP, or any ML lib, run it locally in both GPU mode and mock-embedder mode (`docker-compose.light.yml`). We've had cases where an update breaks ONNX loading or pgvector compatibility. Better to catch it locally than in CI.
- **CI scans for known vulns:** `pip-audit` runs in CI and checks the locked deps against the Python Packaging Advisory DB. If it finds something, the build fails.

**The exception:**
Same as frontend — if a critical CVE lands, skip the local testing step, bump immediately, and get it merged fast.

## Update Cadence

- **Routine stuff** (lint config, utilities, non-core libs): monthly review pass. Batch minor/patch updates together to avoid CI fatigue.
- **Critical** (React, FastAPI, CLIP, our direct imports): quarterly or as needed. Test before deploying.
- **Transitive deps** (things we don't import directly): trust `pip-audit` and `pnpm audit`. If it's clean, it's fine.

## What Actually Breaks (Real Examples)

- **Prerelease versions in frontend:** alpha/beta npm packages sometimes break in subtle ways (tree-shake differences, ESM vs CJS mismatches). Don't use them unless you know why.
- **Abandoned Python packages:** If a package stops getting security updates, we need to notice and either patch ourselves or migrate. Check GitHub stars & last release date before adopting something new.
- **EMBEDDING_DIM misalignment:** If we bump the CLIP model, the embedding dimension changes. If `pyproject.toml` and the pgvector column definition don't match, queries silently fail. Check `backend/src/find_api/core/config.py` before and after updates.
- **GPU vs CPU mismatch:** CUDA versions, torch compatibility, ONNX runtime quirks. Always test both modes.

## How to Handle "This Package Looks Sus"

If a dependency update looks off (new maintainer, weird code change, drastically larger bundle size):
1. Do a GitHub search on the package. Is it being forked/mirrored? Are there open issues about it?
2. Check the npm/PyPI registry page. When was it published? By whom? How many weekly downloads?
3. If still unsure, flag it in a GitHub issue or Slack. We'd rather delay a week than adopt something sketchy.

## Summary

- **Frontend:** Wait 7 days, except for security patches. Let the community catch the grifters.
- **Backend:** Test major updates locally (both GPU & mock mode). Lock everything in `uv.lock`.
- **Security:** Always expedite. A known CVE beats a hypothetical stability issue every time.
- **Unsure?** Ask. Better paranoid than pwned.