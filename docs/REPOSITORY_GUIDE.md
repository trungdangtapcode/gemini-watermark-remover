# Gemini Watermark Remover – Comprehensive Technical Documentation

This document is a deep technical guide to the repository architecture, runtime behavior, processing pipeline, configuration surfaces, and “model” logic used in `gemini-watermark-remover`.

---

## 1) What this repository is

`gemini-watermark-remover` is a pure JavaScript project that removes Gemini image watermarks with a deterministic inverse-compositing approach, not AI inpainting.

The repository ships **three runtime surfaces**:

1. **Web app** (`src/app.js` + `public/`): upload images, process client-side, download results.
2. **Userscript** (`src/userscript/index.js`): patches Gemini page workflows (preview replacement + copy/download hook).
3. **SDK** (`src/sdk/*`): reusable browser / image-data / node APIs for third-party integration.

---

## 2) High-level architecture

## Core engine (`src/core/*`)

The core is split into specialized modules:

- **Alpha templates**
  - `embeddedAlphaMaps.js`: embedded base64-encoded alpha maps for 48 and 96 sizes.
  - `adaptiveDetector.js`: interpolation/warping and correlation scoring.
- **Position and config priors**
  - `watermarkConfig.js`: default and official-size-based config resolution.
  - `geminiSizeCatalog.js`: catalog of official Gemini image dimensions and search seeds.
- **Processing logic**
  - `blendModes.js`: reverse alpha blending math.
  - `watermarkProcessor.js`: full decision + trial + refine + multipass pipeline.
  - `multiPassRemoval.js`: repeated removal passes and stopping criteria.
  - `candidateSelector.js`: candidate scoring/selection.
  - `watermarkDecisionPolicy.js`: normalization of detector/attribution confidence tiers.
- **Metadata/display helpers**
  - `watermarkDisplay.js`, `selectionDebug.js`, `restorationMetrics.js`.

## Platform adapters

- `src/workers/watermarkWorker.js`: worker-side process entrypoint.
- `src/core/workerClient.js`: worker client for web app.
- `src/userscript/processingRuntime.js`: userscript runtime that attempts inline worker and safely falls back.

## Shared integration helpers

- `src/shared/pageImageReplacement.js`: in-page image replacement flow.
- `src/userscript/downloadHook.js`: intercept Gemini native download/copy flows.
- `src/userscript/processBridge.js`: page/userscript `postMessage` bridge.

---

## 3) The processing pipeline (image -> watermark decision -> output)

The main pipeline is implemented in `processWatermarkImageData()` (`src/core/watermarkProcessor.js`).

### Step A: Inputs and defaults

The processor requires preloaded alpha maps:

- `alpha48` and `alpha96` (required).
- optional `getAlphaMap(size)` to support interpolated/custom sizes.
- optional tuning: `adaptiveMode`, `maxPasses`.

If maps are missing, it throws immediately.

### Step B: Initial config prior (where watermark should be)

1. Resolve default config from image dimensions:
   - Prefer official catalog mapping when exact dimension matches (`geminiSizeCatalog.js`).
   - Otherwise fallback heuristic:
     - `>1024 x >1024` -> `96/64/64`
     - else `48/32/32`
2. Run `resolveInitialStandardConfig()` to compare 48-vs-96 correlations at candidate anchors and switch when alternate score is significantly better.

### Step C: Candidate selection and decision tiering

`selectInitialCandidate()` evaluates standard/adaptive candidates, computes trial removals, and picks a best trial.

Internally, scoring includes:

- spatial correlation
- gradient correlation
- near-black guardrails
- adaptive confidence (for non-default offsets/sizes)

If no suitable candidate exists, processing is skipped with metadata:

- `applied: false`
- `skipReason: "no-watermark-detected"`
- decision tier recorded as `insufficient` (or the best available tier).

### Step D: Pass 1 removal

For a selected candidate, the processor applies reverse alpha blend once and records “before/after” metrics.

### Step E: Optional multi-pass cleanup

If residual is still high after pass 1, it calls `removeRepeatedWatermarkLayers()` with remaining pass budget (`maxPasses`, default 4).

### Step F: Alpha-gain recalibration (conditional)

If residual is still problematic, the processor tries candidate `alphaGain` values, bounded by near-black ratio checks, and accepts recalibration only when score improvement is meaningful.

### Step G: Subpixel outline refinement (conditional)

For edge/outline leftovers, it tries subpixel shifts + tiny scale offsets + gain variants (`warpAlphaMap`) and accepts the best refinement when gradient/spatial constraints are improved safely.

### Step H: Output contract

Returns:

- `imageData`: processed pixels.
- `meta`: structured processing metadata containing:
  - `applied`, `skipReason`
  - `size`, `position`, `config`
  - detection metrics and suppression gain
  - template warp / subpixel shift
  - `alphaGain`
  - pass summary (`passCount`, `attemptedPassCount`, `passStopReason`, `passes`)
  - `source` and normalized `decisionTier`
  - selection debug summary

---

## 4) Detection and “model” strategy

Important: this project does **not** use an ML model.

The “model” in this repository is a deterministic, rule-and-template system:

1. **Template prior**: embedded alpha maps (48 and 96).
2. **Dimension prior**: official Gemini size catalog by model family / tier.
3. **Signal model**: spatial + gradient normalized cross-correlation.
4. **Decision policy**: threshold tiers (`direct-match`, `needs-validation`, `safe-removal`, etc.).

### Official size catalog as a structured prior

`geminiSizeCatalog.js` includes official size tables (e.g. `gemini-3.x-image`, `gemini-2.5-flash-image`) and maps resolution tiers to watermark config:

- `0.5k` -> 48 logo, 32 margins
- `1k/2k/4k` -> 96 logo, 64 margins

For near-official exports (scaled variants), it projects candidate watermark anchors into current dimensions and feeds these as adaptive search seeds.

### Adaptive detector behavior

`adaptiveDetector.js` performs coarse-to-fine candidate scoring around bottom-right priors using:

- grayscale conversion
- Sobel gradient magnitude
- normalized cross-correlation against alpha map and alpha gradients
- variance-based penalty/bonus

The resulting confidence and scores are consumed by selector/decision policy instead of being trusted blindly.

---

## 5) Configuration surfaces

## Core processing options

Common options accepted in SDK and runtime wrappers:

- `adaptiveMode`
  - `'auto'` (typical default path)
  - `'always'` (userscript runtime forces this by default)
  - `'off'` / `'never'` (disable adaptive fallback/search)
- `maxPasses` (default pipeline max is 4).
- `alpha48`, `alpha96` (inject precomputed maps).
- `getAlphaMap(size)` (custom alpha source for non-standard sizes).
- `engine` (reuse `WatermarkEngine` instance to cache maps).

## Userscript runtime flags

`src/userscript/runtimeFlags.js`:

- compile-time default: `__US_INLINE_WORKER_ENABLED__` (currently defined as `false` in build config).
- force flag (runtime):
  - `localStorage['__gwr_force_inline_worker__'] = '1'` (page or `unsafeWindow` scope)
  - or `__GWR_FORCE_INLINE_WORKER__ = true`

Only when force/default allows and worker APIs are available will inline worker be attempted.

## Build-time constants

`build.js` injects:

- `__US_WORKER_CODE__`: bundled inline worker string.
- `__US_INLINE_WORKER_ENABLED__`: inline worker default enable flag.

---

## 6) Runtime pipeline by product surface

## A) Web app runtime

1. User uploads image(s) in `src/app.js`.
2. App may initialize module worker client (`WatermarkWorkerClient`) if supported.
3. If worker path fails, app falls back to main-thread engine.
4. For each item:
   - decode image
   - process via engine (worker or main thread)
   - compute/attach metadata
   - render status, preview, and downloadable result

## B) Userscript runtime

Initialization flow (`src/userscript/index.js`):

1. Build fetch path capable of cross-origin blob retrieval (`GM_xmlhttpRequest` + fallback fetch).
2. Initialize processing runtime (`createUserscriptProcessingRuntime`):
   - try inline worker handshake (`ping`) if enabled/forced
   - fallback to main-thread engine on failure
3. Install `postMessage` bridge handlers (`gwr:userscript-process-request/response`).
4. Install Gemini download/copy hook.
5. Install page preview replacement pipeline.

This gives two integration channels:

- DOM image replacement for visible previews.
- native Gemini actions (download/copy) returning processed blobs.

## C) SDK runtime

Entry points:

- `gemini-watermark-remover` (browser + image-data exports)
- `gemini-watermark-remover/browser`
- `gemini-watermark-remover/image-data`
- `gemini-watermark-remover/node`

Node API requires caller-provided decode/encode functions; repository intentionally does not hardcode a single image codec backend.

---

## 7) Build and release pipeline

Build script is `build.js` using esbuild contexts.

### Outputs

- Web app bundle: `dist/app.js` (entry `src/app.js`).
- Worker module: `dist/workers/watermark-worker.js`.
- Userscript bundle: `dist/userscript/gemini-watermark-remover.user.js`.
- Static assets copied to `dist/`:
  - `public/*`
  - `src/i18n/*`

### Development mode (`pnpm dev`)

- watch rebuild for app + worker + userscript
- file watchers sync `public` and `src/i18n`
- internal static server starts (default 4173, auto-increment if occupied)

### Production mode (`pnpm build`)

- clean and rebuild dist once
- minify app/worker (userscript remains `minify: false` for operability/debuggability)

### Script matrix (`package.json`)

- `pnpm dev`, `pnpm build`, `pnpm serve`
- `pnpm test`
- userscript/probe utilities: `probe:tm`, `probe:tm:profile`, `probe:tm:setup`
- benchmarks: `benchmark:samples`, `benchmark:userscript`
- sample export: `export:samples`

---

## 8) Testing strategy

The repository has broad test coverage across:

- **core**: detector, processor, blend modes, policy, config, metrics
- **shared/userscript**: bridge, hook, runtime flags, URL handling, trusted types
- **worker**: worker contract and message path
- **sdk**: API surface, type declarations, package exports, TS consumer
- **project/scripts**: build/config/layout/automation invariants
- **regression**: sample asset consistency and i18n checks

Test command:

```bash
pnpm test
```

---

## 9) Key extension points for contributors

If you want to evolve the algorithm safely:

1. **Catalog updates**
   - Add/adjust official size entries in `geminiSizeCatalog.js`.
2. **Decision policy tuning**
   - Adjust threshold constants in `watermarkDecisionPolicy.js`.
3. **Adaptive search tuning**
   - Tune correlation and confidence weighting in `adaptiveDetector.js`.
4. **Pass/refinement behavior**
   - Adjust pass limits, gain candidates, and acceptance rules in `watermarkProcessor.js`.
5. **Runtime behavior**
   - Worker/main-thread strategy in `workerClient.js` and userscript `processingRuntime.js`.

When changing any of these, run the full test suite and inspect regression sample behavior.

---

## 10) Practical integration examples

## Browser image-data (pure)

```js
import { removeWatermarkFromImageData } from 'gemini-watermark-remover';

const result = await removeWatermarkFromImageData(imageData, {
  adaptiveMode: 'auto',
  maxPasses: 4
});

console.log(result.meta.decisionTier, result.meta.applied);
```

## Engine reuse for batch jobs

```js
import { createWatermarkEngine, removeWatermarkFromImageData } from 'gemini-watermark-remover';

const engine = await createWatermarkEngine();
for (const imageData of batch) {
  const { imageData: out, meta } = await removeWatermarkFromImageData(imageData, { engine });
}
```

## Node integration (codec injected)

```js
import { removeWatermarkFromBuffer } from 'gemini-watermark-remover/node';

const { buffer, meta } = await removeWatermarkFromBuffer(inputBuffer, {
  mimeType: 'image/png',
  decodeImageData,
  encodeImageData,
  adaptiveMode: 'auto'
});
```

---

## 11) Important operational notes

- Userscript inline worker is **not** the default production path; safe fallback to main thread is expected in restrictive Gemini CSP environments.
- A successful `new Worker(...)` constructor alone is not considered success; handshake (`ping`) must complete.
- Runtime flags may need cross-scope reading (`unsafeWindow`) to be effective in userscript context.

---

If you are onboarding to this repository, start from:

1. `src/core/watermarkProcessor.js` (pipeline core)
2. `src/core/geminiSizeCatalog.js` (dimension/model prior)
3. `build.js` (artifact pipeline)
4. `src/userscript/index.js` + `src/userscript/processingRuntime.js` (real-world Gemini integration)
