# ECDAT — Enterprise Cryptographic Discovery & Analysis Tool

**Prototype demonstrating automated cryptographic discovery, inventory, CBOM generation, quantum-risk assessment and migration prioritisation.**
Smart India Hackathon 2026 — Problem Statement SIH26164.

`DISCOVER → INVENTORY → CBOM → QUANTUM RISK → PRIORITISE → MIGRATE`

Everything runs in the browser. No backend, no bundler, no third-party runtime dependency, and **no source code ever leaves the page**.

---

## Quick start

```bash
npm test      # 12 engine tests (detection, scoring, CBOM, PQC mapping, prioritisation, exports)
npm run dev   # serves on http://localhost:5173
npm run build # runs tests, stages ./dist for deployment
```

You can also just open `index.html` through any static server. (Opening it as a `file://` URL will not work — ES modules require `http://`.)

In the app: **Load Enterprise Demo** runs the whole pipeline over the bundled fictional repositories, or go to **Discovery** to scan your own files, a folder, a `.zip`, or a pasted snippet.

---

## Deploying to GitHub Pages

1. Push this repository to GitHub (default branch `main`).
2. **Settings → Pages → Build and deployment → Source: GitHub Actions.**
3. Push. `.github/workflows/deploy.yml` runs the tests, builds `dist/`, and deploys.

It will be live at `https://USERNAME.github.io/REPOSITORY/`.

**Why it works under a subdirectory:** every asset and module is referenced with a relative path (`./assets/...`, `./services/...`) and routing is hash-based (`#/inventory`). There is no base path to configure and no server rewrite needed. `.nojekyll` is emitted so paths beginning with `_` are never stripped, and `404.html` mirrors `index.html` so deep links resolve.

---

## Architecture

Analysis logic is fully separated from the UI. The engines are plain ES modules with no DOM access, which is why they can be unit-tested under Node and swapped for a server-side implementation later.

```
index.html
assets/styles.css
scripts/build.mjs
tests/run-tests.mjs
src/
  app.js                       router, state, all pages, event delegation
  data/demoDataset.js          fictional enterprise repositories (real parsed source)
  services/
    cryptoRules.js             detection rule set + library rules
    discoveryEngine.js         file → artefact records
    quantumRiskEngine.js       deterministic 0–100 scoring + Mosca inequality
    pqcRecommendationEngine.js use-case-dependent PQC / hybrid mapping
    migrationPriorityEngine.js weighted prioritisation and ranking
    cbomGenerator.js           CBOM assembly + CSV flattening
    reportGenerator.js         executive & technical reports, summaries, roadmap
    pipeline.js                orchestration
  ui/
    charts.js                  dependency-free SVG charts
    zip.js                     ZIP reader via the browser's DecompressionStream
```

**Backend integration seam:** replace `runPipeline()` in `services/pipeline.js` with a server call. Every engine takes plain objects in and returns plain objects out, so nothing in the UI changes.

### A note on the stack

The problem statement suggested React + TypeScript + Vite. This build deliberately uses vanilla ES modules instead, for three reasons: it deploys to GitHub Pages with zero configuration and zero build output, it has no supply-chain surface (relevant for a security tool), and the analysis engines run identically in Node and the browser. The module boundaries mirror the suggested `/services` layout exactly, so porting the UI to React later is mechanical.

---

## Risk methodology

Deterministic. The same artefact always produces the same score, and every point is attributable — the **Explain Risk** tab shows each factor with its rationale.

```
score = Σ (normalised factor × weight), clamped to 0–100

Quantum vulnerability   35   per-algorithm value; Shor-breakable public key = 1.0
Algorithm strength      20   key/curve size, mode, and whether already broken
Business criticality    20   critical 1.0 / high .75 / medium .5 / low .25
Data lifetime           12   min(1, years / 15)
Exposure                 8   internet-facing 1.0 / partner .7 / internal .4 / isolated .15
Migration complexity     5   primitive type + 0.06 per dependency (cap 0.3)

Bands:  LOW < 35 ≤ MEDIUM < 55 ≤ HIGH < 75 ≤ CRITICAL
```

**Not everything non-PQC is critical.** RSA, DH, ECDSA and ECDH sit at full quantum vulnerability because Shor's algorithm breaks them. AES and the SHA-2 family sit far lower because generic quantum search gives only a quadratic speed-up — and AES-256 scores below AES-128, SHA-384 below SHA-224. MD5, SHA-1, DES, 3DES, RC4 and SSL score high through *algorithm strength*, not quantum vulnerability, because they are classically broken; the tool reports those two concerns separately.

**Migration priority** is a second, separate blend: `0.40·risk + 0.20·criticality + 0.15·dataLifetime + 0.10·exposure + 0.10·dependencies + 0.05·cryptoImportance`. Migration complexity is deliberately *excluded* — complex work is slower, not less urgent, so it is surfaced on its own.

**Mosca-style assessment:** `dataLifetime + migrationTime > threatHorizon → start planning`. All three values are user-editable on the Risk Analysis and Methodology pages and feed every calculation and report. ECDAT makes **no prediction** about when a cryptographically relevant quantum computer will exist; the horizon is your assumption.

**PQC mapping** depends on the use case: key establishment → ML-KEM (FIPS 203); signatures → ML-DSA (FIPS 204); long-lived trust anchors → SLH-DSA (FIPS 205); AES-128 → AES-256 (no PQC algorithm involved); broken primitives → immediate classical replacement. Hybrid transitions (e.g. `ECDH + ML-KEM-768 → ML-KEM-768`) are presented as recommendations for human review, never automatic replacements.

---

## Demo data

`src/data/demoDataset.js` contains **fictional** source files for seven invented applications — Payment Gateway, Identity Service, Customer Portal, Legacy Banking API, Internal HR System, Cloud Data Service, IoT Management Service — with realistic metadata (environment, criticality, data lifetime, exposure, dependencies).

The dashboard numbers are **not hardcoded**. The discovery engine parses that source text at runtime, exactly as it would parse your uploads, and every metric, chart, score and report is computed from the result. It currently yields ~57 artefacts across a deliberate mix: RSA-1024/2048/4096, ECDSA/ECDH on P-256/P-384, AES-128 and AES-256, ChaCha20-Poly1305, 3DES, DES, MD5, SHA-1, SHA-256/384, PBKDF2, Argon2, TLS 1.2/1.3, SSLv3, X.509 references, and one ML-KEM pilot branch. The UI marks it as demo data on every page.

No real organisation, host, credential or key material appears anywhere in this repository.

---

## Data privacy

- Discovery, scoring, CBOM generation and report rendering all execute in your browser tab.
- Uploaded source is read with the File API and parsed in memory; ZIPs are decompressed with the browser's built-in `DecompressionStream`, not a third-party library.
- No network requests are made with your data. There is no server and no analytics.
- Scan results and planning assumptions are cached in `localStorage` on your device only; **Clear workspace** removes them.
- PDF export uses the browser print dialogue ("Save as PDF") rather than a bundled PDF library.

ECDAT is a **defensive** analysis and inventory tool. It contains no exploitation, credential-harvesting or attack functionality.

---

## Known prototype limitations

- Detection is **pattern-based** — regular expressions over source text. There is no dataflow, taint or call-graph analysis, so both false positives and false negatives are expected. Detection confidence reflects how unambiguous a pattern is, not correctness.
- No binary, runtime, network, TLS-handshake or certificate-store scanning. "Certificate" findings are *references in code and configuration*, not parsed X.509 objects.
- Business criticality, data lifetime, exposure and dependency links are supplied metadata, not measured facts.
- The CBOM structure is **inspired by** CycloneDX cryptographic-asset concepts. It is not validated against the CycloneDX schema and no standards-compliance claim is made.
- Key sizes are captured only where they appear literally in source or config; many findings will legitimately show an unknown key size.
- No authentication, multi-user support, RBAC or persistence beyond `localStorage`. The login screen is explicitly labelled as simulated.
- Not a production security assurance tool: no certification, no guaranteed compliance, no automatic safe migration.

---

## Tests

`npm test` covers RSA / ECC / AES / SHA detection, library-and-version detection, risk determinism and factor-sum consistency, the AES-256 < AES-128 < RSA-2048 ordering, CBOM generation and CSV export, use-case-dependent PQC mapping, weighted prioritisation ordering, Mosca outcomes in both directions, and export non-triviality. All 12 pass.

MIT licensed.
