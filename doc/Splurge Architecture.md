# Splurge: Architecture & Design Document

**Status:** Draft for review — captures all design decisions from the initial planning discussion
**Scope:** Architecture and design only. Work breakdown (epics, tasks, sequencing) is intentionally **out of scope** for this document and will be produced as a separate planning effort using this document as input.
**Constraint:** This document contains no code. It references Surge XT source locations in Appendix A as orientation for planners.

---

## 1. Vision & Goals

Splurge is a new software synthesizer derived from Surge XT. It is not an incremental fork: it lifts Surge XT's proven DSP modules into a new, semi-modular architecture with a rebuilt engine, patch format, and user interface.

### 1.1 What we preserve from Surge XT

These qualities motivated starting from Surge rather than from scratch:

- Free and open-source.
- Integration of high-quality third-party DSP, notably the Airwindows effect suite (which continues to evolve upstream).
- A large existing preset library (made available to Splurge via one-way import).
- Deep, interesting individual modules: a rich oscillator roster, an unusually large waveshaper library, MSEG and Lua-scriptable modulators, MTS-ESP microtuning support.
- Cross-platform operation (Windows, Linux, macOS) as VST3 and CLAP (and AU via the framework).

### 1.2 What changes

The four pillars of Splurge, each elaborated in later sections:

1. **Total visual feedback** (§14): every visible control always reflects its effective, modulated state — composite modulation ranges color-coded by modulator type, current-value cursors, modulation-aware displays — at no cost to GUI frame rate. Inspiration: Serum, Phase Plant, and their ancestor Massive — but taken further than any of them.
2. **Semi-modular lane architecture** (§5–§7): the fixed pipeline (fixed counts of oscillators, filters, effects, modulators, and the two-scene structure) is replaced by a single universal container — the **lane** — supporting any number of oscillators, effects, and modulators per patch, with serial/parallel routing, while defaulting to a sane subtractive signal flow that requires zero routing effort from the casual user. We deliberately push as close to fully modular as possible **without** becoming a pure graph editor — this is not Reaktor, and that boundary is intentional.
3. **Oscillators as modulation sources** (§9): oscillator audio output is routable to parameters, making FM and phase modulation general routing facts rather than fixed topology presets.
4. **Reorganized content** (§11–§12): effects unified into one credit-annotated taxonomy (no separate "multieffects" silo); presets unified under one categorized factory tree with inline author credit and first-class tags.

### 1.3 Non-goals

- **No Splurge → Surge backward compatibility.** Import is one-way (Surge XT presets into Splurge). Splurge presets are not loadable in Surge.
- **No pure graph editor.** Semi-modular, with sane defaults, always.
- **No mobile or WebAssembly version in the initial release.** Viability was assessed and is recorded in §4.3; the long-term vision (a full-fat synth playable away from the studio, with presets that copy back to the desktop plugin) remains on the wishlist.

---

## 2. Guiding Principles

These principles were established during design and should govern all downstream decisions:

1. **Splurge's data model serves Splurge.** Surge import is a one-way translator that converts Surge's specific structures into Splurge's general primitives. The translation may be lossy in *form* as long as it is faithful in *audio result*. "Surge users expect X" or "Surge does it this way" is never a valid justification for a Splurge design choice; "Splurge is a better synth if X" is the only valid form. The importer is allowed to be complex; the Splurge data model is not.
2. **Sane defaults over modular ceremony.** A user following the path of least resistance gets a conventional subtractive synth without touching routing. Modular power is opt-in.
3. **Primitives and packaged effects are both first-class.** Whenever a packaged effect (e.g., a distortion unit) is built around a reusable kernel (e.g., a waveshaping function), both the primitive and the package ship as separate modules (§10.5).
4. **Visual truth.** Anything that has a visible control reflects its current effective state in that control's visible state — always, not only when a modulator is selected.
5. **Performance optimizations must not shape user-facing architecture.** SIMD voice batching and threading are engine concerns hidden behind module context metadata (§6, §8); the user never arranges modules *for* the optimizer, though the UI may gently surface why a module fits certain contexts.
6. **Framework independence of the core.** The module library and engine core are plain C++ with no GUI-framework types in their public interfaces. The framework (JUCE) is confined to the GUI layer and host adapters (§4).
7. **Import compatibility by construction.** Module identity, parameter schemas, and behavior evolve under strict additive rules (§10.3) so that every Surge XT preset remains importable forever, and Splurge presets remain loadable across Splurge versions.
8. **Accessibility from the start.** Surge XT has a genuine accessibility story; Splurge designs accessibility in from day one rather than retrofitting (§14.8).

---

## 3. Strategic Approach: Lift the Modules, Rebuild the Rest

### 3.1 Assessment of the Surge XT codebase

A code review of Surge XT (~163k lines across ~362 source files) found a sharp split:

**Cleanly liftable** — well-defined polymorphic interfaces, self-contained files:
- All 12 oscillator types behind a common oscillator base class with a factory dispatcher.
- All ~31 effects behind a common effect base class with a factory dispatcher, including the Airwindows wrapper hosting ~70+ algorithms.
- Modulation source classes (LFO, envelope, controller/macro sources).
- MSEG (multi-segment envelope generator) storage and evaluator; Formula (Lua-scripted) modulator storage and evaluator; step sequencer.
- Wavetable data structures and I/O; sample loading.
- MTS-ESP and native tuning integration.
- DSP math infrastructure: filters, halfband/oversampling filters, sinc tables, parameter smoothers, the 40+-entry waveshaper function library.

**Tangled with the orchestrator** — and these are precisely the parts Splurge replaces:
- The synthesizer orchestrator (~5k lines): voice allocation, scene mixing, FX chain construction, send/return wiring.
- The voice class, which hard-codes 3 oscillators, 2 filter units, 1 waveshaper, 2 envelope generators per scene, and its coupling to the SIMD filter batching layer.
- The main GUI editor (~7k lines plus ~50 supporting files).
- The hardcoded count constants pervading storage and streaming (3 oscillators, 12 LFOs, 8 macros, 16 FX slots, 2 scenes, etc. — full list in Appendix A).

**Conclusion adopted:** Surge's intellectual value lives in the modules, not the orchestrator. The orchestrator and UI are exactly the parts this project is dissatisfied with. Therefore: **vendor the DSP modules as a static library and build everything above the module boundary fresh.** Doing the rewrite in place — incrementally refactoring the orchestrator while keeping compatibility with a schema we don't want — was evaluated and rejected as slower and dirtier than a fresh build consuming the lifted modules.

### 3.2 The module boundary and the parameter-bag interface

The lifted modules currently declare their parameters into per-module storage structs (oscillator storage, effect storage) using Surge's parameter class. **Decision:** keep those parameter-bag structs and the parameter class as part of the module library's public surface, rather than redesigning the modules' parameter declaration mechanism. Rationale:

- It is dramatically less work than decoupling parameter declaration from storage.
- It makes Surge import nearly mechanical: every parameter value in a Surge patch maps one-to-one onto a Splurge instance of the same lifted module, because the module reads the identical struct.
- The new engine is free to own, allocate, and arrange these parameter bags however it likes (dynamically, inside lanes) without touching module internals.

During the lift, modules must be cleaned so they compile without the Surge orchestrator, voice class, or GUI; any global/static state and reaches into the old patch hierarchy are removed or wrapped. A standalone test harness that instantiates a single module, feeds it input, and verifies output is the regression net for the entire project.

### 3.3 What is built fresh

- **Engine core:** lane runtime, voice management, modulation routing, parameter/effective-value system, signal-bus types, SIMD batching layer, optional thread scheduler.
- **Patch format** and persistence (§11).
- **Surge XT importer** (§12).
- **GUI**: editor, widget library, lane editor, patch browser (§14).
- **Host adapter shells** (thin, via JUCE).

### 3.4 Repository organization

A structure along these lines (names indicative):

- `modules/` — vendored Surge DSP: oscillators, effects, filters, waveshapers, modulation sources, MSEG/Formula evaluators, wavetable and sample I/O, tuning, DSP math. Evolves under the rules of §10.3.
- `engine/` — lane graph runtime, voice runtime, modulation routing, parameter system, patch model, persistence.
- `ui/` — editor, widgets, browser (JUCE permitted here).
- `adapters/` — plugin shells: VST3, CLAP, AU, standalone (JUCE permitted here).
- `importers/surge-xt/` — the one-way Surge preset translator; the only place Surge-specific knowledge lives outside `modules/`.

### 3.5 Known costs of the rewrite path (risk acknowledgment)

Rewriting the orchestrator forfeits years of accumulated fixes in areas that are easy to get 80% right and hard to finish:

- Host integration edge cases: automation, latency reporting, sample-rate changes, freeze/render, MPE corner cases.
- Voice stealing, glide/portamento, monophonic legato modes.
- Undo/redo and automation recording.
- Accessibility (mitigated by designing it in from the start).
- The long tail of importing every factory preset correctly.

These are accepted costs, tracked in the risk register (§16).

### 3.6 Upstream contribution stance

Where Splurge work product happens to be upstreamable to Surge XT (bug fixes in lifted modules, new module algorithms that don't depend on Splurge's engine), we keep it separable — by directory convention and/or build-time definition — so contributions back to Surge remain possible. With the fresh-build approach this is an aspiration rather than a structural driver; the mechanics are an open item (§17).

---

## 4. Technology & Framework Decisions

### 4.1 JUCE retained

Alternatives were evaluated (iPlug2; DPF; raw CLAP/VST3 SDK plus a UI library such as Slint, VIZIA, or Qt; the Rust ecosystem). **Decision: stay with JUCE.** Rationale: it is the path of least resistance to a *finished* product for a developer who doesn't already know an alternative framework — the most mature widget library, accessibility support, MIDI/MPE/automation glue, and plugin-format adapters in the audio space. The licensing concern (GPL/commercial dual license) and the immature mobile/WASM story are accepted trade-offs, revisitable later precisely because of §4.2.

### 4.2 The framework-independence rule (hard rule)

- JUCE types may appear **only** in `ui/` and `adapters/`.
- The module library and engine core expose plain-C++ interfaces with no framework types. Any JUCE leakage found in lifted modules is wrapped or removed during the lift.
- Consequence: a future framework change, a headless build, or a mobile build touches only the shell and UI layers, never the synth itself.

### 4.3 Mobile / WebAssembly viability (recorded, deferred)

- The DSP is portable C++ with SSE intrinsics; porting means an SSE-to-NEON shim or scalar fallback for the SIMD layer — bounded work.
- The UI is the real cost: a touch UI is effectively a full additional UI project, and JUCE's Android and WASM stories are rough.
- Decision: out of scope for the initial release; not a design constraint; the framework-independent core preserves the option.

### 4.4 Plugin targets

VST3 and CLAP as primary targets; AU and standalone via JUCE as the framework makes them cheap. Block-based processing with oscillator-stage oversampling is inherited from the lifted DSP conventions.

---

## 5. The Lane Model (Core Architecture)

### 5.1 The lane is the only structural container

A **lane** is the sole first-class container in a patch. A lane holds:

- **Oscillators** — zero or more.
- **Effects/processors** — zero or more, in a defined serial order.
- **Modulators** — zero or more, each scoped per-voice or per-lane (§9.2).
- **Voice configuration** — polyphony mode (poly/mono/legato/latch), voice count, key range, MIDI channel filter, glide/portamento settings.
- **An output target** — another lane, or master.
- **A user-assignable name**, and a read-only **input label** naming the lane's source (or the generic `<multi>` when more than one source feeds it).

The voice configuration is **dormant** when the lane contains no oscillators: such a lane allocates no voices and acts as a pure processing bus, and the voice-config UI simply does not apply. A lane with oscillators allocates voice instances and manages its own polyphony.

There is deliberately **no** "scene," "voice group," "instrument layer," or other second container concept. An earlier design iteration proposed a "voice group" abstraction; it was recognized as collapsing entirely into lanes (separate sets of oscillators in separate lanes, recombining at master) and was removed. Surge's Scene A/B mechanic is likewise not a Splurge concept — it is reconstituted *only* by the importer as two ordinary lanes (§12.3).

### 5.2 The patch

A patch is:

- An ordered list of **lanes**.
- **Macros** (patch-level, so they can modulate parameters in any lane; modulatable themselves — §9.2).
- **Global modulators** (mod wheel, aftertouch, MIDI CCs, pitch bend, keytrack-style sources — addressable from any lane).
- **Metadata**: name, author, license, comment, category, tags (§11.3).
- **Embedded or referenced resources**: samples, wavetables, impulses (§11.2).

### 5.3 Routing and topology rules

- **Cycle prevention by construction:** a lane may route only to a **higher-numbered lane** or to **master**. This numbered-lane rule was chosen over free-form routing with runtime cycle detection for its simplicity and predictability; it makes circular routes impossible rather than detected.
- **Serial vs. parallel is an emergent property of routing**, not a mode switch: lanes that feed other lanes form serial chains; lanes that feed master directly run in parallel; combinations of both are free.
- **Merging** is implicit: multiple lanes targeting the same destination sum there, and the destination's input label reads `<multi>`.
- **Sends collapse into routing:** there is no separate send/insert mechanism. A "send" is simply a lane that receives routed input (optionally at a routing weight) from one or more other lanes. Surge's send slots import as exactly this (§12.3).

### 5.4 Typed lane signal buses

The connection between lanes is a **typed signal bus**, not a hardcoded stereo pair:

- **Stereo mix** — the default type; a post-mix stereo stream.
- **Per-voice bundle** — a vector of per-voice signals, enabling downstream lanes to process per-note ("Poly" lanes, §7.4). Valid only between lanes sharing voice scope.
- **Multichannel** — the same abstraction generalizes to multichannel layouts (surround, ambisonics, or other formats). Agreed as a supported direction: once the bus is typed, adding a layout is a matter of defining the type and its connection rules, not re-architecting.

The initial release may implement only the stereo type, but the abstraction is designed in from the start so the others are features, not refactors.

### 5.5 Multiband splitting is a lane primitive

The requested "effect that splits a signal into bands routed to different destinations" is **not an effect** in Splurge; it is a routing primitive: a splitter lane (typically oscillator-free) takes one input and produces multiple band outputs, each routed to a different downstream lane. The bands recombine wherever those lanes converge. This falls out of the lane model rather than requiring a special effect API with multiple outputs.

### 5.6 The default experience

A new patch contains one lane with a conventional subtractive layout (oscillator → filter → output to master) and sensible default modulators. The casual user never sees routing; the lane mechanic earns its keep only when the user wants layers, splits, parallel processing, multiband chains, or unusual filter topologies.

### 5.7 Example topologies

| Scenario | Lane configuration |
|---|---|
| Basic patch | One lane: oscillators + voice config + FX → master |
| Five-layer stack | Five lanes, each with own oscillators and voice config → master |
| Imported Surge patch | Two lanes ("Scene A", "Scene B") → master; key/channel filters per Surge's split mode |
| FX bus / send | An oscillator-free lane receiving weighted input from other lanes |
| Multiband processing | Splitter lane → three band lanes (each with its own FX) → master |
| Mono lead over poly pad | Two lanes with different polyphony modes |
| Per-note FX | Upstream lane exposes per-voice bundle; downstream "Poly" lane processes per-note (§7.4) |

---

## 6. Lane Processing Phases

### 6.1 The real distinction: per-voice vs. post-mix

Investigation of Surge's quad-SIMD filter architecture established that the meaningful architectural boundary is **not** "filter block vs. FX section" — it is **per-voice processing context vs. post-mix processing context**. A lane containing oscillators implicitly has two phases:

- **Phase 1 — per-voice:** modules run inside the voice loop, once per sounding voice. Voice-batchable modules are SIMD-batched across voices (§8.1). Per-voice modulators (envelopes, voice-scoped LFOs) apply here. Eligible modules: oscillators, voice-batchable filters, voice-batchable waveshapers/saturators, and any future module written for this context.
- **Phase 2 — post-mix:** the lane's voices have been summed into a single stream. Modules run once per block on that stream. Anything can live here: Airwindows algorithms, reverbs, complex effects. Only lane-scope and patch-scope modulators apply.

A lane without oscillators is pure phase 2 from its first module.

### 6.2 The transition is automatic and visible

The phase 1 → phase 2 transition (the "voice mix") occurs **automatically at the first module in the chain that requires post-mix context**. The user does not manage it, but the UI renders it as a visible divider in the lane so the structure is legible. If the user inserts a post-mix-only module early in a chain, the divider simply moves; nothing breaks. Power users may force an earlier transition (e.g., by dragging the divider or inserting an explicit voice-mix point) — a rare need.

### 6.3 Module context capability

Every module type declares which contexts it supports: **per-voice**, **post-mix**, or **both** (with a separate implementation per context where applicable; the engine selects by placement). This resolves a Surge oddity the design review surfaced: Surge's per-voice filter menu differs from its FX-section filter menu (e.g., no Airwindows filters per-voice) not because those filters are fundamentally different, but because each was *implemented* for one context — per-voice filters are written branchless and vectorizable for voice batching; FX filters are written for a single stereo stream. In Splurge:

- Menus are **unified** (one filter list, one waveshaper list, etc.).
- Each entry carries an indicator of where it can live (per-voice / post-mix / both).
- Friendly, intuitive terminology for *why* a module fits certain contexts is a UI-design task, deferred (§17).

### 6.4 Cross-phase modulation constraint

Per-voice modulators cannot modulate phase 2 parameters *per voice* — after the voice mix there is no voice scope. They may contribute only as an aggregated (summed) control signal. The UI must surface this clearly when a routing edge crosses the phase boundary.

### 6.5 Filters are individual modules

An earlier design iteration proposed lifting Surge's two-filter-plus-routing structure as a single "filter block" module. This was **rejected** (it was justified by Surge's mental model, violating Principle 1). Final decision: **filters are individual, first-class modules** in the lane chain. Multi-filter topologies (parallel, dual, ring combinations) are expressed through Splurge's general mechanisms — module ordering within a lane, or sub-lane splits that recombine — and the importer translates Surge's fixed filter topologies into these forms (§12.4). The SIMD optimization that motivated the filter block moves into the engine as generalized voice batching (§8.1).

---

## 7. Voices & Polyphony

### 7.1 Per-lane voice configuration

Each oscillator-bearing lane owns: polyphony mode (poly, mono, legato variants, latch), voice count limit, key range, MIDI channel filter, glide/portamento. Layers and keyboard splits are just lanes with overlapping or disjoint ranges/channels — there is no separate split/layer feature.

### 7.2 Voice runtime (built fresh)

The engine's voice runtime handles: note on/off and voice allocation per lane; voice stealing; retrigger behavior; glide/portamento; MPE (per-note pitch bend, timbre, pressure); MTS-ESP and native tuning (lifted from Surge's tuning integration). This is the highest-concentration area of audio-domain edge cases being rewritten rather than inherited; it is flagged accordingly in the risk register (§16).

### 7.3 Per-voice state sizing

Surge allocates oscillator state into fixed-size placement buffers. Splurge must **not** carry a fixed per-voice state cap: future modules (notably the granular oscillator, with many grains in flight) have much larger and variable per-voice footprints. Voice state allocation must be dynamic or generously tunable from the first engine design.

### 7.4 Per-note lane processing ("Poly" lanes)

A downstream lane may opt in to consuming an upstream lane's **per-voice bundle** (§5.4) instead of its stereo mix, running its modules once per voice — the equivalent of Phase Plant's "Poly" button. Constraints established:

- Valid only between lanes sharing voice scope (same triggering, same polyphony pool).
- All modules in a Poly lane must support per-voice context (§6.3).
- Opt-in per connection, because carrying N voices' buffers between lanes is memory- and CPU-intensive.

This is a planned post-v1 feature; the typed-bus abstraction it requires is baked into v1 (§13).

---

## 8. Performance Architecture

### 8.1 SIMD voice batching, generalized

Surge's core DSP optimization processes 4 voices simultaneously through the same filter algorithm, with per-voice state packed into adjacent 128-bit SIMD lanes; more than 4 voices are handled as sequential 4-wide batches. Splurge **keeps this optimization and generalizes it**: in a lane's per-voice phase, the engine batches *any* voice-batchable module across voices — filters, waveshapers, and whatever else qualifies — not just a privileged filter pair. A module qualifies for batching if its algorithm is branchless (or branches in lockstep), vectorizable, and its per-voice state packs into SIMD lanes; this is exactly the per-voice context capability of §6.3.

### 8.2 Thread-level parallelism at the lane grain

The design conclusion on multicore use:

- **Lane-grain parallelism is the right grain.** Lanes that don't depend on each other are independent until they merge; the lane graph is a DAG (guaranteed acyclic by the numbered-lane rule), so a topological scheduler can dispatch independent lanes to a worker pool and synchronize only at merge points, once per block.
- Mechanism: a **persistent worker pool** (never spawn threads in the audio callback) fed by a lock-free task queue; **per-worker accumulation buffers** with a final reduction to avoid write contention; platform real-time scheduling cooperation where available (e.g., macOS audio workgroups).
- **Voice-batch-grain threading was considered and rejected**: the work units are microseconds at typical block sizes, so synchronization overhead rivals the work; hosts already parallelize across plugin instances.
- **SIMD widening to 8 lanes (AVX) was considered and rejected as a strategic move**: it is x86-only (NEON is 128-bit), forcing dual implementations for a non-portable ~2× in one pipeline stage. It remains a possible later optimization, not an architecture driver.
- The worker pool is **deferable**: v1 may ship single-threaded, but the lane graph must be designed as a parallelizable DAG from the start so threading is an addition, not a refactor. A user-facing "use up to N cores" control is a possible later refinement.

### 8.3 Acceptable cost envelope

A modest increase in per-voice CPU overhead is acceptable in exchange for architectural generality and better multi-core utilization; what matters is wall-clock time per block and the maximum sustainable voice/lane count.

---

## 9. Modulation System

### 9.1 Identity: dynamic IDs and multi-output sources

- Modulation sources are identified by **stable string/hash IDs**, not enum integers (Surge's `ms_lfo1`-style enum does not carry over). Counts are not fixed: arbitrary numbers of modulators per lane and per patch.
- **Multi-output modulators are first-class.** A routing source is the tuple *(source ID, output name)*, and each modulator declares its outputs with semantic names. This preserves and generalizes Surge's indexed-output convention (MSEG and Formula modulators already emit up to 8 outputs) and is required for planned modulators such as the spiral LFO with separate X and Y outputs (§13).

### 9.2 Scopes

- **Voice scope** — one modulator instance per sounding voice (envelopes, voice LFOs); state lives on the voice; instantiated at note-on.
- **Lane scope** — one instance shared by all voices in the lane (Surge's "scene LFOs" map here); runs once per block.
- **Patch scope** — macros and global sources (mod wheel, aftertouch, MIDI CC, pitch bend, key-tracking sources), addressable from any lane.
- **Macros are modulatable** (meta-modulation): a macro's output is its base value plus a modulation underlayer, so LFOs/envelopes/other macros can modulate macros. This capability exists in Surge and is preserved.
- Scope is a property on the modulator; the cross-phase constraint of §6.4 applies at the voice→post-mix boundary.

### 9.3 Modules expose internal signals as routing sources

Beyond declaring parameters that *receive* modulation, modules declare signals they *emit*. Oscillators expose at minimum: **audio output**, **effective pitch**, **effective frequency (Hz)**, **gate**, and **retrigger**. LFOs expose their value output(s). This single principle makes the following fall out as ordinary routing rather than special-case features: oscillator-as-modulator, ratio/harmonic pitch modes (§13), key-tracked modulation, and cross-module pitch effects.

### 9.4 Audio-rate modulation: the hybrid approach

Three options were evaluated for letting oscillators modulate parameters:

1. Downsample oscillator output to control rate — rejected (cannot actually do FM/PM).
2. Make every parameter an audio-rate buffer — rejected (a massive rewrite of every consumer).
3. **Hybrid (chosen):** routings whose source is audio-rate and whose destination is on an **allowlist of audio-rate-capable destinations** get a dedicated fast path that hands the audio buffer directly to the consumer; all other routings remain control-rate (evaluated once per block). The initial allowlist: FM depth, phase modulation input, filter cutoff — growable over time.

### 9.5 Oscillator-as-modulator

- **Same-lane (v1):** oscillators in the same lane share voice scope; routing oscillator B's audio into oscillator A's FM/phase input is a per-voice, audio-rate routing edge resolved at voice instantiation, mechanically built on the lifted modules' existing audio-rate FM input facility. This is the supported, primary case.
- **Cross-lane (post-v1):** lanes don't share voices, so the only coherent semantic is the upstream lane's *summed* output feeding a downstream parameter as a single shared (non-per-voice) signal — closer to sidechain modulation. Deferred, and to be labeled distinctly ("global") in the UI when built.

### 9.6 FM topologies are routing edges

Splurge has no FM-topology enum. Any oscillator may feed any other same-lane oscillator's FM/phase input via routing edges. Surge's four fixed FM configurations (off; 2→1; 3→2→1; 3→1) exist only inside the importer, which expands them into edges (§12.5).

### 9.7 Modulator catalogue and required behaviors

Lifted: the Surge LFO (full shape set), MSEG, Formula (Lua), DAHDSR-style envelopes, step sequencer, random/alternate sources, MIDI-derived sources. Required behaviors for LFO-class modulators (mostly module-level work; listed here because they are commitments):

- **Trigger modes:** free-running (tempo-synced to host BPM when sync is enabled) or key-triggered.
- **Loop modes:** normal, ping-pong, reverse, and no-loop (one-shot/envelope mode).
- **Freely controllable phase** (start phase/offset as a parameter).

### 9.8 Extending the modulator roster

Two patterns, chosen by fit: a variation that fits the existing LFO's parameter schema becomes a **new shape** in the lifted LFO; fundamentally new behavior with its own state model and parameters becomes a **new module type** under a new ID (§10.3). The spiral LFO and chaotic-attractor modulators are the latter.

### 9.9 Effective values and range data (the UI contract)

The modulation system maintains, per parameter:

- The **current effective value** (base value plus all modulation contributions, as actually applied this block).
- The **modulation range list**: each routing's contribution interval and its source type/identity — sufficient for the UI to draw a composite, color-coded range and a value cursor.

This data is produced on the audio side where modulation is applied, cached, and read by the UI; parameters carry **dirty flags** so the UI repaints only what changed (§14.2). This is new infrastructure — Surge has no "what is my total modulation range" query; the design review confirmed it must be added, and that recomputing it per UI frame by walking routing lists would be too costly.

---

## 10. Module Library

### 10.1 Inventory of lifted modules

- **Oscillators (12):** Classic, Sine, Wavetable, S&H Noise, Audio Input, FM3, FM2, Window, Modern, String, Twist, Alias.
- **Effects (~31):** delays (Delay, Floaty Delay), EQs (3-band parametric, 11-band graphic), modulation effects (Phaser, Flanger, Chorus, Ring Modulator), spectral/spatial (Frequency Shifter, Rotary Speaker, Mid-Side Tool), tone/dynamics (Distortion, Waveshaper, Conditioner, Resonator), reverbs (Reverb 1, Reverb 2, Spring Reverb), character/experimental (Vocoder, Combulator, Treemonster, Ensemble, Tape, Bonsai, Nimbus, CHOW, Neuron, Exciter), Convolution, Audio Input, and the **Airwindows wrapper** (~70+ algorithms behind one selector).
- **Filters:** Surge's per-voice filter algorithm library (the SIMD-vectorized implementations), lifted as individual filter modules.
- **Waveshapers:** the 40+-entry transfer-function library (§10.5).
- **Modulators:** per §9.7.
- **Infrastructure:** wavetable structures and I/O, sample loading, MTS-ESP/native tuning, DSP math, oversampling filters, smoothers.

### 10.2 Module identity and namespaces

Every module type has a **stable string ID**, namespaced by origin: lifted modules as `surge.*` (e.g., `surge.sine`, `surge.wavetable`), Splurge originals as `splurge.*` (e.g., `splurge.sampler`, `splurge.granular`, `splurge.spiral-lfo`). Splurge patches reference modules by these IDs. Loading a patch that references an unknown ID fails cleanly and informatively.

### 10.3 Evolution and compatibility rules

These rules guarantee that import compatibility survives indefinite library growth:

| Change | Import-safe? | Rule |
|---|---|---|
| Add a new module type | Always | New IDs live in their own namespace; old presets never reference them |
| Add a new modulation source or destination | Always | Same reasoning |
| Add a parameter to an existing lifted module | Yes | Must ship a sensible default for old presets |
| Add a shape/mode/sub-algorithm to an existing module | Yes | Old presets simply don't select it |
| Rename a parameter's display name | Yes | Display names are not identity |
| Rename a parameter's ID | No | Forbidden without a migration shim mapping old → new |
| Change a parameter's range or units | Only with migration | A converter for stored values is mandatory |
| Remove a lifted module | Never | Deprecate and hide from menus instead; the registry keeps it forever |
| Change a module's sound for the same parameter values | Only with versioning | Bump that module's streaming version and keep old behavior reachable; when in doubt, **fork under a new ID** rather than mutate |

Additionally: the **Surge-enum → string-ID translation table is frozen forever** once shipped (§12.3), and lifted-module parameter schemas are **append-only**.

### 10.4 Per-module streaming versions

Surge's pattern of per-module streaming-version fix-ups (each oscillator/effect reconciles old patch data against current behavior at load) is **kept intact in the lifted modules** — inheriting Surge's accumulated migration logic for free — and **adopted for Splurge's own modules** going forward, so Splurge presets stay loadable across Splurge versions by the same mechanism.

### 10.5 Primitives and packaged effects (the waveshaper/distortion ruling)

A waveshaper is a memoryless per-sample transfer function; a distortion effect is that kernel **plus context** (drive, pre-filter, post-filter, output gain, wet/dry, optionally internal oversampling). Functionally a distortion at neutral settings *can* act as a bare waveshaper, but Splurge ships **both as separate modules**:

- **Waveshaper (primitive):** the transfer function with drive/output gain; carries Surge's full 40+-shape library; cheap enough to drop anywhere, including the per-voice phase.
- **Distortion (packaged):** pre-gain → pre-filter → the same shape library → post-filter → mix/output, with optional oversampling; the successor to Surge's Distortion effect.

The general principle (Principle 3) extends across the library — filter primitive vs. auto-filter package, delay-line vs. chorus/flanger packages — whenever a kernel is independently useful.

### 10.6 Effects taxonomy and crediting

- Surge's "multieffects"-style silos are **dissolved**: all effects live in one unified, categorized menu (filtering, delay, reverb, distortion, dynamics, modulation, spectral, utility, …).
- Effects derived from external projects are **credited inline** in their menu entries — e.g., "ZLowpass [AW]" or "ZLowpass [Airwindows]" under the Filtering category, or under an Airwindows subheading within their functional category. Individual Airwindows algorithms are **hoisted into the master menu** by function rather than buried behind a single "Airwindows" effect entry (the single-wrapper dispatch implementation may remain; the *organization* changes).

---

## 11. Patch Format & Persistence

### 11.1 Format principles

- Human-readable serialized body (XML or similar), following the precedent that made Surge patches debuggable and importable.
- **Versioned from day one**, with Splurge's stream revisions starting **above** Surge's range (e.g., at 100, with Surge XT currently at 28): any patch whose revision is below the Splurge floor is routed to the Surge importer; at or above, it is a native Splurge patch.
- Splurge presets are explicitly **not** backward-compatible with Surge (§1.3).

### 11.2 Resources

Embedded binary resources are a **generic** facility (a typed blob with metadata), not a wavetable-specific one — required so samples (sampler oscillator), wavetables, and impulses all embed uniformly. By-name references to library resources remain supported, as in Surge.

### 11.3 Metadata, categories, and tags

- Fields: name, author, license, comment, category, **tags** (multiple).
- **Tags are first-class.** (Design review finding: Surge already serializes tags in patch metadata and indexes them in its SQLite patch database — the gap is purely UI. Splurge surfaces them fully: tag editing, tag filtering, tag-based browsing.)
- **Open investigation:** collapsing categories into tags entirely (category as a distinguished tag) — see §17.

### 11.4 Library organization

- **One unified factory tree, organized by sound category** (Basses, Leads, Pads, …). The Surge-style split — factory content by category but third-party content grouped by author folder — is eliminated: every preset, including third-party content, lives under a category, with the **author credited inline** in the browser (read from patch metadata, which Surge patches already carry).
- User content remains in a user location, also categorized and tagged.

---

## 12. Surge XT Import

### 12.1 Philosophy

One-way translation, Surge XT → Splurge. Success criterion: **perceptual audio equivalence** (not bit-identity) for imported presets. The importer is the *only* component (outside the lifted modules themselves) allowed to contain Surge-specific knowledge, and it is allowed to be elaborate; restructuring (splitting, merging, reshaping topologies) is fine as long as the sound survives.

### 12.2 Why this is tractable

1. Surge patches are a binary wrapper around readable XML — hand-debuggable.
2. Because the lifted modules keep their parameter-bag structs (§3.2), every Surge parameter value populates the identical structure the Splurge module reads — no per-parameter reinterpretation.
3. Surge's fixed topology is a strict subset of what lanes can express, so import is a well-defined projection.
4. Keeping the lifted modules' streaming-mismatch logic (§10.4) means patches from *any* Surge era are first normalized by Surge's own accumulated migration code.

### 12.3 The structural mapping

| Surge concept | Splurge mapping |
|---|---|
| Scene A / Scene B | Two lanes (named "Scene A"/"Scene B") → master; key-range/channel filters configured from Surge's split/dual/channel scene modes; MPE channel conventions mapped onto the two lanes by default |
| 3 oscillators per scene | 3 oscillator module instances in that scene's lane |
| Oscillator FM config | Routing edges (§12.5) |
| Filter section (2 units + waveshaper + routing mode) | Module ordering or sub-lane splits (§12.4) |
| Scene insert FX (4 per scene) | The lane's post-mix (phase 2) effects, in order |
| Send slots (4) and per-scene send levels | Oscillator-free lanes receiving weighted input from the scene lanes |
| Global FX (4) | The master lane's effect chain |
| 8 macros | 8 patch-level macros (with meta-modulation preserved) |
| 6 voice LFOs + 6 scene LFOs per scene | 12 modulators in that lane: voice-scoped and lane-scoped respectively |
| Modulation routings (voice / scene / global lists) | Routing entries with translated source identities, via a **static, frozen lookup table** (~40 entries) from Surge's source enum to Splurge IDs, including indexed outputs (MSEG/Formula output indices → output names) |
| MSEG, Formula, step-sequencer data | Lifted storage structures, unchanged |
| Wavetables (embedded or by name) | Same mechanisms, via the generic resource facility |
| Tags, author, comment, license, category | Carried into Splurge metadata; category re-homed into the unified tree |
| Streaming revision (≤ 28) | Triggers the importer (§11.1) |

### 12.4 Filter-topology mapping

Surge's filter-block routing modes decompose into Splurge primitives:

| Surge filter mode | Splurge import shape |
|---|---|
| Serial 1 / 2 / 3 (waveshaper at differing positions) | Filter and waveshaper modules ordered in the lane chain per mode |
| Parallel | Sub-lane split: one filter per sub-lane, recombined |
| Dual | Oscillator outputs routed to separate sub-lanes, one filter each |
| Stereo | Filter modules with stereo channel assignment |
| Ring | Sub-lane split recombined through a ring-modulation combine module |
| Wide | Stereo-width module following the filter chain |

This requires two small Splurge-native utility modules — a **ring-mod combine** and a **stereo width** module — which are independently useful, not import-only artifacts.

### 12.5 FM-configuration mapping

| Surge FM config | Imported routing edges |
|---|---|
| Off | none |
| 2 → 1 | osc 2 audio → osc 1 FM input |
| 3 → 2 → 1 | osc 3 → osc 2 FM input; osc 2 → osc 1 FM input |
| 3 → 1 | osc 3 → osc 1 FM input |

### 12.6 What survives, what's hard, what's lost

- **Preserved:** all parameter values, oscillator/effect type selections, modulation routings and depths, macros (incl. meta-modulation), wavetables, MSEG/Formula/step data, tags and metadata.
- **Hard parts (bounded):** the per-mode filter translations above; the ~40-entry source translation table (mechanical); MPE/channel-mode mapping; macro meta-modulation (free if the modulator model allows modulators to target modulators, which it does).
- **Acceptable losses:** patches exploiting un-migrated ancient-Surge quirks or topology edge cases that don't project cleanly (expected to be vanishingly rare); users are warned when an import is imperfect.

### 12.7 Importer build strategy

Built incrementally, each stage a regression checkpoint: (1) smoke-test import — right oscillators at right pitch, most routing ignored; (2) full parameter coverage; (3) modulation routings; (4) FX chains and lane mapping; (5) the full factory corpus loads and sounds correct. The corpus then remains a permanent regression suite.

---

## 13. Planned Future Features and Their v1 Foresight

These are agreed *nice-to-haves* — **not** initial-release commitments — evaluated for what the v1 architecture must anticipate. Four foresight items are baked into the v1 design; everything else is purely additive later.

| Feature (inspiration) | Description | Architectural prerequisite | Status of prerequisite |
|---|---|---|---|
| Sampler oscillator (Serum 2, Phase Plant) | Sample playback as an oscillator: pitch resampling, start/end, loop points | Generic embedded binary resources in the patch format | **Baked into v1** (§11.2) |
| Granular oscillator (Phase Plant) | Grain scheduling/density/position over sample data | No fixed per-voice state cap | **Baked into v1** (§7.3) |
| Spiral LFO; chaotic modulators (Serum 2; Lorenz attractors, etc.) | Modulators emitting separate named outputs (X and Y, etc.) | Multi-output routing sources | **Baked into v1** (§9.1) |
| Pitch modes: octave/semi/fine; pitch ratios (Serum 2); harmonic ratios (Phase Plant, Serum 2); pitch shift (Phase Plant) | Ratio/harmonic modes follow another oscillator's pitch | Oscillators expose effective pitch as a routing source; the mode UI is a shortcut that creates the edge | **Baked into v1** (§9.3) |
| Per-note lane processing (Phase Plant "Poly") | Downstream lane processes each voice separately | Typed lane buses incl. per-voice bundle | **Baked into v1** (§5.4, §7.4) |
| Multichannel lane output | Surround/ambisonic/other layouts between lanes | Same typed-bus abstraction | **Baked into v1** (§5.4) |
| Phase offset & randomness (Phase Plant) | Oscillator start-phase control and per-note randomization | None — oscillator parameters | Pure addition |
| Noise oscillator | Standard and exotic generated noise colors; sampled noise expected via the sampler instead | None — new module | Pure addition |
| LFO behaviors | Free-run w/ BPM sync or key-trigger; normal/ping-pong/reverse/no-loop (envelope); free phase | None — module-level (§9.7); cheap enough that these may land in v1 | Pure addition |
| Cross-lane oscillator modulation | Upstream lane's summed audio as a "global" modulation source | Audio-rate hybrid path (§9.4) plus a labeled global-source concept | Post-v1 (§9.5) |

---

## 14. User Interface Design

### 14.1 The visual feedback doctrine

The defining UI commitment, stated as requirements:

1. **Every control always displays its composite modulation range** — the union of all modulation contributions affecting it — **color-coded by modulator type**, visible at all times (not only when a modulator is selected, which is Surge's behavior). The displayed range must be the *actual reachable range* of each routing (e.g., an LFO modulating a 10-second portamento by a small depth shows only that sliver, never the whole bar).
2. **Every control carries a current-value cursor** showing the effective (modulated) value within the control's range, live.
3. **Modulation-aware displays:** oscillator displays, LFO displays, and similar visualizations reflect the *effective, currently-modulated* state — fed by the running engine's effective-value snapshots — not static parameter values. (Surge's oscillator preview renders an unmodulated instance; Splurge's must not.)
4. **Universality:** anything that can possibly have a visible control reflects its state in that control's visible state. No invisible modulation.
5. **Framerate is sacred:** none of the above may degrade GUI frame rate.

### 14.2 The data plumbing behind it

Requirement 5 is met by architecture, not hope: the engine maintains the per-parameter effective-value and modulation-range cache (§9.9), populated where modulation is applied; the UI is a read-only consumer; per-parameter **dirty flags** confine repaints to changed widgets. (Design review finding: Surge's editor refreshes modulation state for *all* parameters on a global flag at 60 Hz — the always-on composite display would multiply that cost unsustainably; per-widget invalidation is mandatory, and rendering must stay cheap per widget.)

### 14.3 Knob-first control design

Splurge's UI favors **knobs over sliders**. Surge's slider-heavy design wastes screen real estate and reads as visually intimidating; most continuous parameters become knobs (with the composite ring/arc range display of §14.1), with sliders reserved for cases where they genuinely communicate better. (In Surge this distinction is skin-orientation-driven rather than widget-driven; in Splurge's fresh UI it is a deliberate design default.)

### 14.4 The lane editor

The central new UI surface: lanes with visible module chains; the automatic phase divider rendered in-lane (§6.2); editable lane names; read-only input labels (`<multi>` for merged inputs); output-target selection per lane (which is also how serial vs. parallel is expressed); creation/reordering within the numbered-lane constraint. The overall visual metaphor (rack-style strips vs. node-graph vs. paneled) is an **open question** (§17) — with the constraint that it must keep the path-of-least-resistance patch looking like a normal synth, not a patching environment.

### 14.5 Module browsing

Unified, categorized module menus (§10.6) with origin crediting and per-entry context indicators (per-voice / post-mix / both), using approachable language for *why* (terminology TBD, §17).

### 14.6 Patch browser

One factory tree by sound category; inline author display; first-class tag filtering and editing; fast search/type-ahead over an indexed patch database (Surge's SQLite-backed patch DB concept carries forward).

### 14.7 Skinning

Whether Splurge ships a skin engine (community skinnability as in Surge) or a fixed, polished theme in v1 is an **open question** (§17). The widget architecture should keep theming (colors/assets) separable regardless.

### 14.8 Accessibility

Screen-reader labeling, keyboard navigability, and parameter accessibility are designed in from the first widget. JUCE's accessibility support is part of why it was retained.

### 14.9 Editing infrastructure

Undo/redo across parameter, routing, and structural (lane/module) edits, and host automation recording, are required product features and must be considered in the engine's edit model (structural edits are harder to undo than value edits if bolted on later).

---

## 15. Quality & Testing Strategy

- **Module harness:** standalone instantiation and golden-output tests for every lifted module — built during the lift, kept forever (§3.2).
- **Import corpus:** every Surge XT factory and bundled third-party patch must load and produce sound; staged importer milestones per §12.7; perceptual-equivalence spot checks; the corpus runs as a permanent regression suite.
- **Engine tests:** lane-graph construction/ordering, phase-boundary placement, modulation application, voice lifecycle (allocation, stealing, legato/glide), typed-bus connection rules.
- **Performance:** regression benchmarks against Surge XT on comparable patches (voices/CPU); GUI frame-time budget tests with heavy modulation visualization; thread-scheduler correctness (when enabled) under sanitizers.
- **Host matrix:** the major DAWs across Windows/macOS/Linux, exercising automation, render/freeze, sample-rate changes, MPE.

---

## 16. Risk Register

| Risk | Notes / mitigation |
|---|---|
| Voice-runtime edge cases (stealing, legato, glide, MPE) | Highest concentration of rewritten domain knowledge; mitigate with Surge behavior as reference, early host testing, targeted tests |
| Host-integration papercuts | Accumulates late; mitigate by standing up the plugin shell early and testing in real DAWs throughout |
| Module-lift surprises (hidden globals, storage reach-ins) | Budget explicitly; the test harness catches behavioral drift |
| Importer long tail | Staged strategy (§12.7) and corpus regression keep it incremental |
| GUI performance under always-on visualization | Per-parameter dirty flags and audio-side caching are load-bearing (§14.2); benchmark early with worst-case patches |
| Threading correctness | Deferable feature (§8.2); design the DAG now, ship threads only when provable |
| Scope creep in the lane editor | The open metaphor question (§17) should be settled with prototypes before deep investment |
| Accessibility debt | Avoided by doing it from the start (§14.8) |

---

## 17. Open Questions

Decisions deliberately left open, with context:

1. **Lane editor visual metaphor** — rack-style vertical strips vs. node-graph vs. paneled layout (§14.4). Constraint: simple patches must look like a normal synth.
2. **Skin engine in v1** — community skinnability vs. fixed theme (§14.7).
3. **Module-library evolution policy** — track upstream Surge module improvements (periodic re-sync) vs. diverge freely after the lift (§3.6 mechanics included).
4. **Context-capability terminology** — the friendly wording/iconography explaining per-voice vs. post-mix module availability (§6.3).
5. **Categories collapsing into tags** — investigate making category a distinguished tag rather than a separate axis (§11.3).
6. **Worker pool in v1 or deferred** — DAG design is committed either way (§8.2).
7. **Upstream contribution mechanics** — directory/build conventions for Surge-contributable work (§3.6).
8. **Per-module wet/dry convention** — a uniform mix-control policy across packaged effects (Surge mixes per-effect and per-slot inconsistently).

---

## 18. Decision Index

Quick reference for planners; each decision is elaborated at the cited section.

| # | Decision | Where |
|---|---|---|
| 1 | Lift Surge DSP modules as a vendored library; rebuild engine, patch format, UI, adapters fresh | §3 |
| 2 | Keep module parameter-bag structs as the module API surface | §3.2 |
| 3 | Stay on JUCE; confine it to UI and adapters; core stays framework-independent | §4 |
| 4 | Mobile/WASM deferred; viability recorded | §4.3 |
| 5 | The lane is the only container; no scenes, no voice groups | §5.1 |
| 6 | Macros and global modulators live at patch level | §5.2, §9.2 |
| 7 | Cycle prevention via the higher-numbered-lane-or-master rule | §5.3 |
| 8 | Sends are just lanes; serial/parallel emerges from routing | §5.3 |
| 9 | Typed lane buses: stereo, per-voice bundle, multichannel | §5.4 |
| 10 | Multiband splitting is a lane primitive, not an effect | §5.5 |
| 11 | Default patch is a conventional subtractive lane; routing is opt-in | §5.6 |
| 12 | Lane phases: per-voice → automatic, visible voice-mix transition → post-mix | §6 |
| 13 | Modules declare context capability; unified menus with indicators | §6.3 |
| 14 | No "filter block": filters are individual modules; topologies via ordering/sub-lanes | §6.5 |
| 15 | Quad SIMD voice batching kept and generalized to all voice-batchable modules | §8.1 |
| 16 | Threading at lane grain (persistent pool, DAG scheduler); per-voice threading and AVX widening rejected | §8.2 |
| 17 | Dynamic string IDs; multi-output sources as (source, output) tuples; no fixed modulator counts | §9.1 |
| 18 | Modulator scopes: voice / lane / patch; macros modulatable | §9.2 |
| 19 | Modules expose internal signals (pitch, gate, audio out…) as routing sources | §9.3 |
| 20 | Audio-rate modulation via hybrid allowlist (FM depth, phase, cutoff first) | §9.4 |
| 21 | Oscillator-as-modulator: same-lane in v1; cross-lane (summed, "global") post-v1 | §9.5 |
| 22 | FM topologies are routing edges; no topology enum | §9.6 |
| 23 | LFO commitments: trigger modes, BPM sync, loop modes (normal/ping-pong/reverse/one-shot), free phase | §9.7 |
| 24 | Per-parameter effective-value + modulation-range cache with dirty flags | §9.9, §14.2 |
| 25 | Namespaced stable module IDs (`surge.*`, `splurge.*`) | §10.2 |
| 26 | Additive-only evolution rules; frozen translation table; fork-don't-mutate; per-module streaming versions | §10.3–§10.4 |
| 27 | Primitives and packaged effects both ship (waveshaper + distortion) | §10.5 |
| 28 | Multieffects dissolved; unified taxonomy; inline source crediting (e.g., "ZLowpass [AW]"); Airwindows hoisted | §10.6 |
| 29 | Human-readable versioned patch format; Splurge revisions start above Surge's; one-way import only | §11.1 |
| 30 | Generic embedded-resource facility | §11.2 |
| 31 | Tags first-class; unified factory tree by category; inline author credit | §11.3–§11.4 |
| 32 | Import: perceptual equivalence; importer owns all Surge-specific knowledge; staged build with corpus regression | §12 |
| 33 | Always-on composite color-coded modulation ranges + value cursors; modulation-aware displays; framerate-safe | §14.1–§14.2 |
| 34 | Knob-first UI | §14.3 |
| 35 | Accessibility, undo/redo, automation recording designed in from the start | §14.8–§14.9 |
| 36 | Four v1 foresight items for future features (resources, voice-state sizing, multi-output routing, typed buses) | §13 |

---

## Appendix A: Surge XT Codebase Reference

Orientation points for downstream planners, from the design-phase code review. Paths relative to the Surge XT repository root.

**Orchestration & voice (replaced by Splurge engine):**
- Per-block processing entry: `SurgeSynthesizer::process()` — `src/common/SurgeSynthesizer.cpp` (≈ line 4557). Flow: input upsampling → per-scene voice rendering → quad filter batches → downsampling → scene insert FX → scene summing → send FX → global FX → master gain/clip.
- Per-voice processing: `src/common/dsp/SurgeVoice.cpp` (`process_block` ≈ line 1032); oscillators placement-new'd into fixed buffers (≈ lines 525–544); modulation applied additively into a local parameter copy (≈ lines 1264–1283).
- SIMD voice batching: `src/common/dsp/QuadFilterChain.h`; voices processed in 4-wide batches (`SurgeSynthesizer.cpp` ≈ line 4775). Voice pool: 2 × 64 pre-allocated voices.

**Hardcoded counts (the constraints Splurge removes)** — mostly `src/common/SurgeStorage.h` (≈ lines 72–83, 157): `n_oscs = 3`, `n_egs = 2`, `n_lfos_voice = 6`, `n_lfos_scene = 6`, `n_lfos = 12`, `n_filterunits_per_scene = 2`, `n_fx_params = 12`, `n_fx_slots = 16`, `n_send_slots = 4`, `n_scenes = 2`; `n_customcontrollers = 8` (`ModulationSource.h` ≈ line 130); `MAX_VOICES = 64`, `BLOCK_SIZE = 32` with 2× oscillator oversampling (`globals.h`).

**Modules (the lift):**
- Oscillator base & factory: `src/common/dsp/oscillators/OscillatorBase.h`; `spawn_osc` in `Oscillator.cpp`; 12 types enumerated in `SurgeStorage.h` (≈ lines 283–300); one .h/.cpp pair per type under `src/common/dsp/oscillators/`. Audio-rate FM input via the oscillator's FM-assignment facility.
- Effect base & factory: `src/common/dsp/Effect.h` (≈ lines 41–134); `spawn_effect` in `Effect.cpp`; ~31 effects under `src/common/dsp/effects/`.
- Airwindows wrapper: `src/common/dsp/effects/airwindows/AirWindowsEffect.{h,cpp}` — parameter 0 selects among ~70+ registered algorithms; remaining parameters re-typed dynamically per algorithm.
- Waveshaper function library: `QuadFilterWaveshapers` (40+ transfer functions).
- Modulation: source enum (~42 entries) and routing struct (source, destination, depth, mute, source index, scene) in `src/common/ModulationSource.h` (≈ lines 40–84, 253–266); three routing lists (voice, scene, global) in `SurgeStorage.h`; MSEG (≤128 segments, multiple curve types), Formula (Lua, ≤8 outputs), step sequencer storages in `SurgeStorage.h`.
- Parameter system: `src/common/Parameter.h` — `modulateable` flag, normalized-value conversion; **no existing "total modulation range" query** (the gap §9.9 fills).
- Wavetable data/IO: `src/common/Wavetable.h`; mipmapped, shared across voices.

**Patch system:**
- Format: FXP wrapper (`sub3` header) around XML plus embedded wavetable blobs; load/save in `src/common/SurgePatch.cpp` (load ≈ line 1126, XML save ≈ line 3764). Streaming revision `ff_revision = 28` (`SurgeStorage.h` ≈ line 151); per-module streaming-mismatch handlers throughout oscillators/effects.
- Metadata already includes name, category, author, license, comment, and **tags** (serialized ≈ lines 3789–3798).
- Categories derive from **folder paths** (factory by sound category; third-party by author folder — the layout §11.4 replaces); the in-file category attribute is ignored on preset load. SQLite-backed patch DB (`src/common/PatchDB.*`) indexes tags as searchable features.

**GUI (replaced, with findings that shaped §14):**
- Editor: `src/surge-xt/gui/SurgeGUIEditor.{h,cpp}` (~8k lines combined) + ~50 supporting files; 60 Hz idle timer; modulation refresh iterates all parameters on a single flag.
- Controls: `ModulatableSlider` — knob vs. slider is orientation in skin XML, not a widget type; modulation display state is three-valued (unmodulated / modulated-by-active / modulated-by-other) with positive/negative colors only — no composite or per-source rendering.
- Oscillator preview (`OscillatorWaveformDisplay`) renders an unmodulated oscillator instance.
- Skin engine: XML-driven `.surge-skin` packages under `resources/data/skins/`.
