---
name: poster-square-expand
description: >
  Expand movie/marketing posters (or any image) into a perfect square by
  outpainting the existing scene outward, WITHOUT inventing new subjects.
  Fully autonomous: the user just drops one or more images and sends them,
  with no further instructions. Trigger whenever a user attaches one or more
  posters/images and asks to make them square, expand/extend them, outpaint,
  fill to square, "square these up", pad to square, or resize-to-square for
  a set of posters. Built for licensed/copyrighted characters (studios who
  own their IP): uses only IP-safe models that will not refuse the content.
---

# Poster Square Expand

Turn any poster into a clean square by extending the scene that is ALREADY
there (sky, ground, walls, atmosphere) out to the new edges. Never add new
people, characters, or subjects. If a subject's limb runs off-frame, finish
that limb naturally, nothing more.

This skill is **zero-instruction**. When the user drops images and sends
them, run the entire pipeline end-to-end for every image without asking
questions. Only stop to ask if an input is not an image or is already square.

## Defaults (do not ask the user)

- **Target size:** 3000 x 3000 px, square.
- **Padding placement:** centered. A portrait poster gets equal width added
  left + right; a landscape poster gets equal height added top + bottom.
- **Prompt:** environment-only, auto-derived from the image (see below).
  Never describe subjects.
- **Output format:** PNG.
- **Batch:** if the user drops multiple images, process them all in parallel.

## Why these models (IP-safe rule)

Studio posters contain copyrighted characters the client owns but the model
cannot verify. **Flux 2 Pro Outpaint and Ideogram Reframe both refuse
copyrighted characters at the provider level (422) and cannot be bypassed.**
Do not use them for this. The two routes that work:

- `fal-ai/flux-pro/v1/fill` — **primary.** Paints into a white mask region on
  a padded canvas. Best edge continuation in testing. Set `safety_tolerance`
  to its max so licensed IP does not trip a refusal.
- `bria/expand` / `fal-ai/bria/expand` — **fallback.** Licensed-data model,
  never refuses, and expands by canvas geometry in one shot at full res.

## Pipeline (run per image, all images in one batch where possible)

### Step 1 — Pad to square canvas + build mask (sandbox, `run_python`)

For each input image:

1. Download it, read width `W` and height `H`.
2. If already square, skip expansion (just upscale to 3000 if needed).
3. Compute the square side `S = max(W, H)`. Center the original on an
   `S x S` canvas:
   - portrait: `pad_x = (S - W) // 2` on left and right
   - landscape: `pad_y = (S - H) // 2` on top and bottom
4. Export two files at `S x S`:
   - **padded**: original centered on the square canvas. Fill the new area
     with **edge-replicate** (stretch the poster's own edge pixels outward),
     so the seed the model paints over is already color-matched. This alone
     removes most of the density mismatch that shows up as a seam on lighter
     posters.
   - **mask**: black where the original sits, WHITE over the padding to fill.
     Then apply a **moderate gaussian feather (radius ~8-12 px) to the mask
     boundary only**. This softens the hard black/white edge into a short
     gradient so the handoff between kept and filled pixels blends instead
     of showing a crisp line. This is the ONE change from the plain-mask
     version — keep it moderate; do NOT bite inward into the original and do
     NOT over-blur (heavy feathering added complexity for little gain).
5. `save_output` both, keep their URLs.

Reference example files from a real run:
- padded: https://v3b.fal.media/files/b/0aa503b1/U0NJKrF0NXaqI18MPFThC_motu_castle_padded.png
- mask:   https://v3b.fal.media/files/b/0aa503b1/EesowIJOsjXBC4mjTgj0K_motu_castle_mask.png

### Step 2 — Auto-derive an environment-only prompt

Look at the image and describe ONLY the surroundings to continue into the
padding: sky, ground, walls, weather, atmospheric particles. End with
"continuous background". NEVER name or describe any character/subject, or the
model may invent duplicates.

Good real examples used in production:
- graveyard poster: `dark night sky, faint moonlight glow, misty graveyard ground, drifting particles, continuous background`
- temple duel: `fiery temple interior, glowing red stone steps, drifting embers, continuous background`
- hero portrait: `dramatic sunset sky, distant fantasy city skyline, drifting embers, continuous background`
- group on rock: `stormy dramatic sky, dark clouds, drifting embers, continuous background`
- castle courtyard: `misty castle courtyard, stone walls, hazy sunlight, continuous background`
- riding mount: `bright cloudy sky, rugged rocky mountains, drifting embers, continuous background`
- dark villain: `dark green smoky haze, murky atmosphere, drifting mist, continuous background`

Keep it under ~15 words. Match the existing color palette and lighting.

### Step 3 — Flux Pro Fill (`fal-ai/flux-pro/v1/fill`)

Fetch the schema, then submit one run per image in ONE
`submit_batch_model_run` (slug `poster-square-fill`). Per-run input:

- `image_url`: the padded canvas URL from step 1
- `mask_url`: the feathered mask URL from step 1
- `prompt`: the environment-only prompt from step 2
- `safety_tolerance`: max allowed by schema (licensed IP)
- `output_format`: `"png"`

Chain to step 4 with `continueAfter`.

Note: Fill returns at reduced resolution (approx 1264 px for ~1600 canvases).
That is expected; step 4 restores full size.

### Step 4 — Topaz upscale to 3000 (`fal-ai/topaz/upscale/image`)

On the continuation, fetch the schema, then one run per fill result in ONE
`submit_batch_model_run` (slug `poster-square-upscale`):

- `image_url`: the fill result URL
- model: `"High Fidelity V2"`
- `upscale_factor`: `3000 / current_width`, clamped to Topaz max (4.0). A
  safe default of `2.5` overshoots 3000 from a 1264 fill.
- `face_enhancement`: `true` (sharpens any faces in the poster)
- `output_format`: `"png"`

### Step 5 — Optional exact crop to 3000 x 3000 (sandbox)

If the overshoot needs to be exact, downscale/center-crop each upscale result
to exactly 3000 x 3000 in one `run_python` pass and `save_output`. Do this
automatically only if results overshoot; otherwise deliver the Topaz output.

## Consistency & guardrails

- Environment-only prompts + a mask that protects the original are what stop
  new characters from appearing. If duplicates ever show up, shorten the
  prompt further (drop everything but sky/ground + "continuous background").
- The seam fix is just two cheap things: edge-replicate padding + a moderate
  feathered mask boundary. That is deliberately the WHOLE change from the
  plain version. Do not add inward biting into the original, heavy blur, or
  other steps — they add complexity without improving results.
- Keep placement centered unless the user explicitly asks otherwise.
- If Flux Fill refuses or degrades on a specific image, fall back to
  `bria/expand` for that one (it never refuses and runs at full res).
- 402 / payment errors are account credit/billing issues from the provider,
  NOT a pipeline or masking problem. Nothing in mask geometry can cause a
  402. If they appear, it is a credits/quota matter to resolve on the account.
- Do not ask clarifying questions for the standard case. Just run it.
