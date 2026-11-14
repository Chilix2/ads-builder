HANDOFF — Social Video Studio → rebuild into our platform
Audience: Claude Code (terminal). Read this whole file before touching code.
Source of truth is the code in this repo; this file explains intent, architecture, and what is left.
---
1. Goal
A browser-based, CapCut-style editor for short-form social content (Reels, TikTok, Shorts, Stories, Carousels).
The user composes a post from footage, text, stickers, comment cards, UI "forms", effects and audio, then gets:
Exports: PNG frame, WebM video, per-layer transparent PNG overlays, project JSON.
A native rebuild plan for Instagram (Edits app) or TikTok (in-app editor / Studio / CapCut Pro):
step-by-step instructions that recreate the post with the platform's own tools (native captions,
native stickers, native music) so the post gets native reach instead of being an uploaded, flattened video.
Anything the platform cannot do natively is baked into a "plate" video or shipped as an overlay PNG.
Your job: port this editor into our platform (our auth, our storage, our design system, our routing),
keeping the engine (`lib/editor/*`) behavior identical, then finish the open items in section 6.
---
2. Stack
Next.js 16 App Router, React 19, TypeScript, Tailwind v4 (tokens in `app/globals.css`), shadcn (`components/ui`), lucide icons.
AI SDK v7 (`ai`) through Vercel AI Gateway — plain model ids like `google/gemini-3-flash`. No provider keys in code.
No database, no auth yet. State is an in-memory store (see 4.1).
Run: `pnpm install && pnpm dev` → http://localhost:3000. Typecheck: `npx tsc --noEmit` (currently clean).
---
3. File map
```
app/
  page.tsx                  → renders the editor (client-only via components/editor/editor-client.tsx)
  layout.tsx, globals.css   → fonts + design tokens
  api/transcribe/route.ts   → POST audio → word-timed transcript (AI SDK transcription) → auto captions
  api/speech/route.ts       → POST text+voice → TTS audio (AI SDK speech)
  api/extract/route.ts      → POST image → JSON list of reusable elements (forms/stickers + bbox), Gemini vision

components/editor/
  editor.tsx        Shell: TopBar / LibraryPanel / Stage / Inspector / SlidesBar / Timeline + keyboard shortcuts
  top-bar.tsx       Format, post type, target platform, undo/redo, export menu, opens RebuildGuide
  library-panel.tsx Left tabs: Templates, Media, Text, Elements, Extract, Audio, Effects, Filters, Motion (drag & drop to stage)
  extract-panel.tsx ExtractTab (image → elements), AutoCutTemplates, SavedElements
  audio-panel.tsx   Transcribe → captions, TTS voice, voiceover recording, music
  stage.tsx         Canvas preview + selection handles, drag/resize/rotate, safe-zone overlay
  inspector.tsx     Per-layer property editor (switches on layer.kind)
  timeline.tsx      Tracks, scrub, trim, split, drag clips
  slides-bar.tsx    Story / carousel slides
  rebuild-guide.tsx Native rebuild plan UI (checklist, tiers, copy Markdown / download manifest)
  controls.tsx, layer-thumb.tsx  small shared UI

lib/editor/            ← THE ENGINE. Framework-agnostic except store.ts/playback.ts (tiny external stores)
  types.ts          Data model (Project, Layer union, Asset, Slide, formats, post types, limits)
  store.ts          useSyncExternalStore store: project, selection, undo/redo (80 steps), slides, layer ops
  playback.ts       time / playing store
  factory.ts        create* layer factories, demo project, SAMPLE_ASSETS (public/samples/*.png), FORM_DEFAULTS
  render.ts         Canvas 2D renderer for every layer kind (1237 lines) — used by stage AND export
  render-forms.ts   Renderer for UI "forms" (iOS alert, AirDrop, Notes, Google search, context menu, ID badge, iMessage, Safari tabs)
  animation.ts      In/out/loop animation math (computeAnim), preview keyframes
  presets.ts        ANIM_IN, ANIM_OUT, LOOPS, FILTERS, EFFECTS catalogs
  library.ts        Library items (stickers, comments, forms, frames, shapes, templates) + toolIdForItem
  actions.ts        insertItemById, applyTemplate, insertAsset, importFiles, saved elements, DRAG_MIME
  assets.ts         Image/video/audio element cache, media sync during playback, audio capture for export
  captions.ts       Transcript → caption TextLayers (5 styles incl. karaoke)
  autocut.ts        CapCut-style AutoCut: beat detection (Web Audio) + cut templates
  export.ts         PNG, WebM (MediaRecorder on canvas + audio), per-layer PNGs, project JSON import/export
  safe-zones.ts     Platform UI safe zones (TikTok, Reels, Shorts, X) drawn on stage
  platform-tools.ts Native tool catalog: menu paths per platform, tier (free / edits-plus / capcut-pro / studio-web), verify flag, fallbacks
  rebuild.ts        buildRebuildPlan / summarizePlan / planToMarkdown / planToManifest (schema studio.rebuild/v2)
```
---
4. Architecture you must preserve
4.1 State
`editor` (store.ts) is a module singleton read with `useSyncExternalStore`. `editor.get().project` is the whole document.
`project.layers` is the live layer list of the active slide. `project.slides[activeSlide]` is stale — always read slides via `getSlides(project)`.
Undo: call `editor.checkpoint()` once before a live gesture (drag/resize), then `setProject(p, false)` per frame.
When porting, you may swap this for Zustand/Redux, but keep the same semantics (checkpoint + live updates, 80-step history).
4.2 Coordinates & sizing
Layer `x,y` = center in project pixels; `width,height` in project pixels; `rotation` in degrees.
Project sizes come from `FORMATS` (1080×1920, 1080×1350, 1080×1080, 1920×1080). Scale sizes with `unit(p) = min(w,h)/1080`.
4.3 Rendering
One renderer (`render.ts`) draws a frame at time `t` onto a 2D canvas. Stage preview and every export call the same code — never fork it.
Layer visible when `start <= t < start + duration`. Animations via `computeAnim(layer, t)`. Effects in `CAMERA_EFFECTS` transform the whole scene.
Form layers: wrap words in `*asterisks*` in title/body to color them with `accent`.
4.4 Rebuild plan logic (rebuild.ts)
For each layer `strategyOf(layer, platform)` returns:
`native` — rebuild with the platform tool (text, auto-captions, music, polls, mentions…)
`fallback` — platform lacks it, step names the substitute
`plate` — baked into the base video (media, effects, keyframes, filters)
`overlay` — exported transparent PNG, added as a photo sticker (shapes, forms, foreign comment cards)
Auto-caption lines are grouped into one step. Comment cards are native only if the variant matches the platform's reply sticker.
4.5 AI routes
All server-side, AI Gateway, zero-config auth on Vercel. Validate input size/type (already done: 400/413). Keep `maxDuration`.
---
5. Status — what is done
Editor shell, canvas stage with transform handles, timeline (trim/split/move), inspector for all 8 layer kinds.
Layer kinds: media, audio, text, shape, sticker (29 types incl. polls/quiz/frames), comment (7 variants), effect (12), form (8).
Animations in/out/loop, filters, adjustments, warp.
Formats 9:16 / 4:5 / 1:1 / 16:9; post types reel / story / carousel with per-platform slide limits.
Safe zones for TikTok, Reels, Shorts, X.
Audio: transcription → captions (5 styles), TTS, voiceover recording, music, fades/volume.
AutoCut templates with beat detection.
Extract: upload a screenshot → AI returns editable forms/stickers → insert or save to "Saved elements".
Exports: PNG, WebM with audio, per-layer PNG overlay pack, project JSON (assets inlined), JSON import.
Native rebuild guide for Instagram & TikTok with tiers, verify flags, Markdown + JSON manifest.
Keyboard: Space play, Delete, S split, Esc, arrows nudge (Shift = 10px), Cmd/Ctrl+Z / Shift+Z.
Standalone HTML prototypes (not part of the app, included in the zip under `artifacts/`):
`artifacts/social-overlays/index.html` — overlay gallery, images inlined as base64, fully self-contained.
`artifacts/comment-card/` — comment card prototype.
---
6. Remaining work (do in this order)
Port into our platform
Move `lib/editor/*` as-is into our repo (e.g. `packages/editor` or `src/features/editor`). Keep tests of behavior identical.
Rebuild `components/editor/*` with our design system; keep props/store usage. Use `components/ui` equivalents we already have.
Mount at our route (e.g. `/studio/[projectId]`) behind our auth. The editor must stay client-only (`dynamic(..., { ssr:false })` pattern in editor-client.tsx).
Move the 3 API routes to our backend; keep request/response shapes (see section 7).
Persistence (not implemented)
Projects live only in memory; refresh loses work. Save `Project` JSON per user to our DB (debounced autosave + explicit save).
Assets are data/blob URLs inside the project. Upload files to our object storage, store URLs in `Asset.src`, never inline base64 in the DB.
Scope every query by the session user id.
Video export to MP4
Current export is WebM via MediaRecorder (realtime, browser-dependent). Platforms prefer MP4/H.264.
Option A: WebCodecs + mp4 muxer client-side. Option B: server render job (ffmpeg) from project JSON + `render.ts` in a headless worker. Plates (`base-plate`, `plate-N`) must exclude `native`/`overlay` layers — filter with `strategyOf(...) === 'plate'`.
Export pack
Zip plates + overlay PNGs + `plan.md` + `manifest.json` in one download (currently separate downloads).
Extract improvements
`sticker` results are bbox crops only; add background removal for real cutouts.
Verify bbox accuracy on real screenshots; add a manual crop adjust step.
Responsive / mobile
Library hidden below `md`, Inspector hidden below `lg`. Add bottom-sheet versions for mobile.
Verify native menu paths
Every path in `platform-tools.ts` with `verify: true` must be checked against the current Edits / TikTok / CapCut app versions. Update paths, not logic.
Hardening
Rate-limit the AI routes per user. Add security headers (nosniff, referrer policy, HSTS, X-Frame-Options SAMEORIGIN).
Large projects: render.ts redraws everything per frame — add an offscreen cache for static layers if perf drops.
Optional: Playwright agent that consumes `manifest.json` (schema `studio.rebuild/v2`) to drive the rebuild automatically.
---
7. API contracts
```
POST /api/transcribe   multipart: audio=File        → { words: [{ text, start, end }], text, ... } (see audio-panel.tsx)
POST /api/speech       json: { text, voice }        → audio/mpeg body
POST /api/extract      multipart: image=File (≤8MB) → { elements: ExtractedElement[] }  (type in route.ts; bbox = 0–1 fractions, x,y = top-left)
Errors: { error: string } with 400 / 413 / 500
```
---
8. Acceptance checklist
[ ] `npx tsc --noEmit` clean, app boots, demo project renders identical to this repo.
[ ] Drag/resize/rotate, timeline trim/split, undo/redo work.
[ ] Transcribe → captions, TTS, Extract all work through our backend.
[ ] Project survives reload (DB) and assets load from storage.
[ ] Rebuild plan for the demo project matches this repo's output for both platforms (compare `planToMarkdown`).
[ ] MP4 export plays on iOS and uploads to TikTok/Instagram.
