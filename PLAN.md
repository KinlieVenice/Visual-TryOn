# Visual-TryOn: Project Plan

## 1. Problem

Online clothes shopping relies on a size chart and a guess. We want users to see, and be told numerically, how a specific garment in a specific fabric would fit their specific body, without handing the decision to a generative image model.

## 2. Product summary

1. The user enters body measurements (guided) and picks or refines a body type.
2. We build a 3D avatar of them. Optionally they add their face (selfie), hair and skin tone.
3. They enter or upload a garment size chart and the fabric (composition, weight, knit or woven).
4. We simulate the garment on the avatar and produce:
   - a **fit report**: ease per body region (chest, waist, hip, thigh, length, ...), labelled tight / good / loose, with a strain heatmap;
   - a **3D view** of the garment on the avatar, rotatable.
5. *Later:* photo mode puts the simulated garment on the user's real photo.

**Guiding rule: AI perceives, physics decides.** Models turn messy input (a selfie, a chart image) into numbers. The fit verdict comes only from deterministic physics and arithmetic, never from a neural network.

## 3. Principles

- **Accuracy over spectacle.** The fit report is the product; the picture illustrates it.
- **Deterministic core.** Same input gives same output (fixed solver time step and iteration count, no random seeds). Test with tolerances, not exact float equality.
- **Offline first.** Everything except photo mode works with no network.
- **Show uncertainty.** Display a confidence level instead of implying false precision. Typical expected error is about 1-2 cm on the body, more on drape.
- **Privacy by design.** Body and face data are sensitive biometric data. Process on-device by default, with explicit consent, encryption, and delete-on-request.

## 4. Pipeline

```
measurements + body-type refinement ─► SMPL-X avatar ─┐
(optional selfie ─► FLAME face, hair, skin tone)       │
                                                       ├─► XPBD cloth sim ─► fit report + 3D view
size chart + fabric ─► scaled base pattern ─► 3D garment┘

[Phase 3, cloud] simulated render + user photo ─► diffusion pass (visual only)
```

| Stage | Method | AI? | Offline? |
|---|---|---|---|
| Measurements to body shape | Fit SMPL-X shape parameters (regression / optimization) | No | Yes |
| Size chart entry | Manual form; OCR plus rules parser | Small OCR, optional | Yes |
| Messy chart / fabric label | Rules fallback; optional ~1B quantized SLM | Optional SLM | Yes |
| Fabric properties | Preset library plus formula from fiber content and GSM | No | Yes |
| Garment pattern | Scale a base pattern per garment type to the chart | No | Yes |
| Cloth simulation | XPBD (or similar) solver | No | Yes |
| Fit report and heatmap | Garment minus body per region; strain | No | Yes |
| 3D rendering | WebGPU / three.js | No | Yes |
| Face from selfie | Landmark model (MediaPipe-class) plus FLAME fit | Small CNN | Yes |
| Hair and skin tone | Asset library plus color sampling | No | Yes |
| Photo mode | Diffusion conditioned on sim render | Yes, heavy | Cloud |

The core pipeline must work with **zero language models**. An SLM is only a convenience for parsing charts and fabric labels, and for the measuring guide.

## 5. Offline-first architecture

- **Core engine** (body fit, pattern scaling, cloth sim, fit analysis) written once in **Rust or C++**, compiled to **WASM** (browser, PWA) and native (mobile, tests). One code path, so results are identical everywhere.
- **Client:** installable PWA (React/Next.js or similar) with guided measuring, 3D viewer, fit report UI.
- **Models:** small specialized models in **ONNX / TFLite**, bundled or downloaded once on demand (body model, face model, OCR, fabric presets, hair library). Decide install-time vs on-demand download based on size.
- **Storage:** local-only by default (IndexedDB / on-device encrypted store). No account needed for core use.
- **Cloud service (photo mode only):** FastAPI plus a GPU job queue running the diffusion pass. Stateless, with images deleted after processing, and opt-in.
- **Determinism controls:** pinned model versions, fixed solver parameters, versioned fabric presets and base patterns; golden-file tests for the engine.

## 6. Avatar details

- Built from entered measurements (height, weight, chest, waist, hip, inseam, shoulder width, thigh, etc.).
- Measurements alone are ambiguous (same waist, different shape), so add a **body-type refinement** step: reference silhouettes or sliders (shoulder slope, belly, glute/thigh fullness, posture). Optional front/side photos can refine the *shape only*, not be displayed.
- Always show the avatar's computed measurements beside the user's tape measurements; flag mismatches and ask for a re-measure.
- **Face and hair (cosmetic only, decoupled from fit):** selfie to FLAME face texture; hairstyle/color library, optional match-from-photo; skin tone sampled and applied to the body. Decide realistic vs stylized look early, because a mediocre realistic face is worse than a good stylized one.
- **Misuse guard:** the face must be the user's own (live selfie check); no storing or sharing face data beyond the session without opt-in.

## 7. Known limits

- Size charts are inconsistent between brands. Ask whether the chart is flat-garment or body measurements.
- A chart gives only 4-8 numbers; the rest of the pattern comes from base templates (needs real design work per garment type).
- Stretch and soft fabrics are hardest; rigid ones (denim, structured shirts) are easiest. Elastane content strongly changes how a chart should be read.
- Photo-only body estimation is weak, so measurements stay mandatory.
- Per-try-on compute cost in photo mode must be budgeted early.

## 8. Phases

### Phase 0: Validation spike (2-4 weeks) *(the gate for everything else)*
Prove the simulation matches reality before building anything else.

- [ ] Choose cloth solver for the spike (Blender / NVIDIA Warp / Houdini Vellum) and set up an offline script.
- [ ] Parametric body: fit SMPL-X to a set of measurement vectors; check that output measurements match inputs within tolerance.
- [ ] One base pattern (t-shirt), scaled by chart measurements, stitched and draped.
- [ ] Fabric presets: 5-10 fabrics (cotton jersey, cotton poplin, denim, polyester, elastane blends) with params: stretch (warp/weft/bias), bending stiffness, GSM, friction.
- [ ] Collect ground truth: 5-10 real people, a few real garments in known sizes; record body measurements, garment flat measurements, photos, and subjective plus measured fit.
- [ ] Compare simulated per-region ease against measured ease; record error per region and fabric.
- [ ] Benchmark XPBD cloth speed on a mid-range phone/laptop (feasibility of on-device).
- [ ] Define go / no-go thresholds (for example, ease within +/-1.5 cm on rigid fabrics, fit label agrees with reality in >=80% of cases).

### Phase 1: MVP (offline)
- [ ] Core engine in Rust/C++ to WASM, with golden-file determinism tests.
- [ ] Guided measurement flow (illustrations, validation, unit handling).
- [ ] Measurement-based avatar plus body-type refinement.
- [ ] Manual size-chart entry form (flat vs body measurements toggle).
- [ ] Fabric picker (presets) plus composition/weight entry.
- [ ] Tops and straight-leg pants only.
- [ ] Fit report (per-region ease, labels, strain heatmap, confidence).
- [ ] 3D viewer (rotate, compare sizes side by side).

### Phase 2: Personalization and parsing
- [ ] Selfie to FLAME face, skin tone, hairstyle library.
- [ ] On-device OCR for chart images plus rules parser, with a mandatory user confirmation of the parsed table.
- [ ] Fabric label reader; optional small SLM behind a feature flag.
- [ ] More garment types, bigger fabric library.
- [ ] Stylized vs realistic look decision, implemented.

### Phase 3: Photo mode (cloud) and ecosystem
- [ ] Diffusion pass conditioned on sim render (depth, normals, garment mask); shape locked to the simulation.
- [ ] Opt-in cloud upload, deletion guarantees.
- [ ] "Did it fit?" feedback loop to calibrate fabric presets and patterns.
- [ ] Brand size-chart presets; B2B plugin for online shops.

## 9. Risks

| Risk | Mitigation |
|---|---|
| Sim does not match reality | Phase 0 gate; do not proceed without passing thresholds |
| Body ambiguity from measurements | Body-type refinement; show computed vs entered measurements |
| Uncanny face/hair | Stylized option; keep cosmetics decoupled from fit |
| Privacy / biometric law (GDPR, BIPA) | On-device by default, explicit consent, delete-on-request, encryption |
| On-device performance | Benchmark in Phase 0; adjustable solver quality |
| Face-swap abuse | Own-face check; no retention without opt-in |
| Pattern coverage effort | Start with two garment types; expand by demand |

## 10. Open questions

1. Consumer app (users upload their own charts) or embedded in shops' sites (B2B)?
2. Web/PWA, native mobile, or both first?
3. Which garment types first (tops and pants assumed)?
4. Realistic or stylized avatar look?
5. Team and budget: build the cloth engine ourselves, or wrap existing tools for Phase 1?
6. Target markets (affects privacy law and size standards: US/EU/Asia sizing)?
