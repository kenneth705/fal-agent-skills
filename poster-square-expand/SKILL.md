name: poster-square-expand
description: >-
Expand movie/marketing posters (or any image) into a perfect square by
outpainting the existing scene outward, WITHOUT inventing new subjects. Fully
autonomous: the user just drops one or more images and sends them, with no
further instructions. Trigger whenever a user attaches one or more
posters/images and asks to make them square, expand/extend them, outpaint,
fill to square, "square these up", pad to square, or resize-to-square for a
set of posters. Built for licensed/copyrighted characters (studios who own
their IP): uses only IP-safe models that will not refuse the content.
Poster Square Expand
Turn any poster into a clean 3000x3000 square by extending its existing
environment outward on the short sides. This is a production pipeline hardened
against two real failure modes: (1) IP moderation refusing copyrighted
characters, and (2) a faint seam line where the original meets the fill.
Zero-instruction contract
The user will typically attach one or more images and say something like "make
these square" or nothing at all. Do not ask clarifying questions. Every
decision below is already made. Run the whole pipeline autonomously for every
attached image, then hand back the finished squares.
Defaults (never ask, never deviate unless the user explicitly overrides):
Target canvas: 3000 x 3000 px, centered.
Output format: PNG.
Batch ALL attached images (one square each), in parallel where possible.
Extend the environment ONLY. Never add new people/characters/creatures.
THE IP WALL — read this first
The characters in these posters are copyrighted. The client owns them, but the
models do not know that. Two popular outpaint models hard-refuse and must
NOT be used:
fal-ai/flux-2-pro/outpaint — BLOCKED by provider moderation on copyrighted
characters. enable_safety_checker: false does NOT override it. Dead end.
fal-ai/ideogram/v3/reframe — BLOCKED by moderation, no bypass. Dead end.
Use these IP-safe models only:
fal-ai/flux-pro/v1/fill — PRIMARY. Mask-driven fill, never refuses. Winner
in side-by-side tests. Runs at a reduced canvas (~1264px), so it MUST be
upscaled afterward.
fal-ai/bria/expand — FALLBACK. Licensed training data, never refuses, runs
a full 3000x3000 in one shot from canvas geometry (no mask). Use if Flux Pro
Fill ever misbehaves on a given poster.
Always set safety_tolerance to its maximum on Flux Pro Fill.
The no-new-subjects rule (critical)
The fill prompt must describe environment only — sky, clouds, buildings,
mountains, ground, haze, embers. NEVER describe or name any character, person,
creature, or the subject. Naming a subject makes the model paint a duplicate of
it into the new edge. If duplicate limbs/figures appear, the prompt is too rich
— shorten it to the barest environment description and rerun.
It is fine to let the model finish a partial element already crossing the seam
(a leg, a wall, a cloud running off-frame). It is NOT fine to introduce a new
one.
Seam-elimination technique (v2 — the important refinement)
On lighter posters (bright skies), a naive pad-and-fill leaves a subtle
vertical line exactly where original meets fill. Three sandbox tweaks kill it:
Inward BITE = 28px. The white (fill) region eats 28px INTO the original
on each padded side. Flux then repaints a thin strip of the real poster edge
and blends it with the new area, so the transition happens inside
repainted territory instead of on a hard boundary.
FEATHER = 40px gaussian on the mask boundary. The keep-to-fill handoff
becomes a soft gradient, not a crisp edge — this is what removes the line on
bright/low-texture areas.
Edge-replicated padding. Fill the padded strips with the poster's own
edge pixels (replicate), NOT black. Even the seed the model paints over is
already color-matched to the adjacent image.
Pipeline
For each attached image:
1. Pad to square + build mask (sandbox, run_python)
Download the source. Determine the long side; target square = max(3000, long
side) but standard is 3000.
Center the original on a 3000x3000 canvas. Pad the short sides with
edge-replicated pixels (PIL: crop the 1px edge column/row and tile it, or
ImageOps/numpy edge replicate).
Build a matching 3000x3000 mask: black = keep, white = fill. Start with
white over the padded strips, then extend each white strip 28px inward
over the original (the BITE). Gaussian-blur the mask by 40px (the
FEATHER).
If the source is already landscape/short in height, pad top/bottom instead —
same logic on whichever axis is short.
save_output both the padded image and the mask.
2. Derive an environment-only prompt (automatic)
Look at the source edges and write a terse (<=12 word) environment description
of what continues outward. Examples from production:
clear blue sky, city skyline buildings, continuous background
bright cloudy sky, rugged rocky mountains, drifting embers, continuous background
fiery temple interior, glowing red stone steps, drifting embers, continuous background
dramatic sunset sky, distant fantasy city skyline, drifting embers, continuous background
stormy dramatic sky, dark clouds, drifting embers, continuous background
misty castle courtyard, stone walls, hazy sunlight, continuous background
Never name a character. If duplicates appear on rerun, cut the prompt down.
3. Flux Pro Fill (fal-ai/flux-pro/v1/fill)
image_url = padded image, mask_url = feathered mask, prompt =
environment-only string, safety_tolerance = max.
Lands ~1264x1264. That is expected — upscale next.
4. Topaz upscale (fal-ai/topaz/upscale/image)
model = High Fidelity V2, face_enhancement = true,
output_format = png.
upscale_factor chosen to reach >=3000px (e.g. 2.0 from ~1600px squares,
2.5x from ~1264px). Overshoot is fine.
5. Optional exact crop (sandbox)
If the client needs pixel-exact 3000x3000, center-crop the upscaled result to
3000x3000 in run_python and save_output.
Batch behavior
Multiple attached posters = do all of them. Sandbox pad+mask can loop in one
run_python script; the Flux Pro Fill runs go in one submit_batch_model_run;
the Topaz upscales go in one submit_batch_model_run. Chain: pad/mask ->
fill batch -> upscale batch.
Fallback
If Flux Pro Fill produces artifacts or a stubborn seam on a specific poster,
rerun that one through fal-ai/bria/expand (full 3000x3000, no mask) and
upscale only if needed.
