# gemini-vectorize

Batch-convert photos into print-ready SVG line art. AI photo-to-vector
pipelines leave you doing the same tedious hand-offs every time — redraw the
photo, upscale it, vectorize it, then clean up the SVG colors before it's fit
to print. This CLI chains all of that into one idempotent batch job:

```
photo
  │  Gemini "nano banana" image model, redrawn per your prompt   ~$0.067
  ▼
redrawn/*.png
  │  Recraft crisp-upscale                                        4 units (~$0.004)
  ▼
upscaled/*.jpg
  │  Recraft vectorize                                           10 units (~$0.01)
  ▼
vectorized/*.svg
  │  svg-color-rinse --optimize (color cleanup + svgo)
  ▼
vectorized/*.svg  (print-ready)
```

Rough cost per photo through the full pipeline: **~8¢** (Gemini image
generation + 14 Recraft API units).

## Install

```sh
# run without installing
npx gemini-vectorize

# or install globally
npm install -g gemini-vectorize
gemini-vectorize
```

Requires Node 20+. Zero npm dependencies — uses the global `fetch`,
`FormData`, and `Blob` APIs.

## Setup

You need two API keys:

- **Gemini** — create one at [aistudio.google.com/apikey](https://aistudio.google.com/apikey),
  then `export GEMINI_API_KEY=...`
- **Recraft** — create one at [recraft.ai/profile/api](https://recraft.ai/profile/api),
  then `export RECRAFT_API_KEY=...`

**Privacy note:** on Google's **paid** Gemini API tier, inputs and outputs are
not used for model training. On the **free** tier, they *are* used for
training. If you're sending private photos through this pipeline, enable
billing on your Google AI Studio project before running it.

## Quick start

```sh
mkdir photos
cp your-photos/*.jpg photos/
gemini-vectorize init      # writes a starter prompts.json — edit it to taste
gemini-vectorize           # redraw + upscale + vectorize + rinse, end to end
```

Results land in `vectorized/*.svg`.

## Intake folders: one per prompt

Every key in `prompts.json` gets its own intake folder:

- the `default` key reads from `photos/`
- every other key `K` reads from `photos-<kebab-case(K)>/` — `keepText`
  reads `photos-keep-text/`, `noBackground` reads `photos-no-background/`

To add a new intake variant, just add a key to `prompts.json` and create the
matching folder — no code changes. All intake folders feed the same
`redrawn/` → `upscaled/` → `vectorized/` pipeline. Empty or missing folders
are simply skipped.

The starter `prompts.json` (from `gemini-vectorize init`) ships three
variants:

- `default` (`photos/`) — high-contrast ink illustration, no text preserved
  or added
- `keepText` (`photos-keep-text/`) — same style, but retains text already in
  the photo (signage, captions, labels) without inventing new text
- `noBackground` (`photos-no-background/`) — same style, with the background
  removed and replaced by one of three randomly chosen treatments (halftone
  dots, comic starburst, angular geometric panels) — see prompt variants below

All redraws land in the single `redrawn/` folder under the source filename,
so **use unique filenames across intake folders** — a `photo1.jpg` present in
two intake folders both maps to `redrawn/photo1.png`, and whichever is
processed second gets skipped as already done.

## Prompts are config, not code

Prompts live in `prompts.json` in your working directory:

```json
{
  "default": "redraw this photo as a stark high-contrast black-and-white comic ink illustration, heavy ink lines, blocky black shadows, bold negative space, flat binary line art. Pure illustrative artwork: no text, no titles, no logos.",
  "keepText": "redraw this photo as a stark high-contrast black-and-white comic ink illustration, heavy ink lines, blocky black shadows, bold negative space, flat binary line art. Keep the text that is there, do not add any extra text. No added titles, mastheads, trade dress, barcodes, or logos.",
  "noBackground": [
    "…flat binary line art. Remove the background entirely and replace it with a flat ben-day halftone dot pattern…",
    "…flat binary line art. Remove the background entirely and replace it with a dramatic 1960s comic starburst with bold radiating rays…",
    "…flat binary line art. Remove the background entirely and replace it with bold flat angular geometric shapes and diagonal panels…"
  ]
}
```

Run `gemini-vectorize init` to write this starter file if you don't have one
yet (the real starter file spells out each `noBackground` variant in full).
Edit it freely — this is where your house style lives (ink density, line
weight, era references, whatever), and each key you add is a new intake
variant with its own `photos-<kebab-case(key)>/` folder. `--prompt "..."`
overrides the `default` prompt for a single run without touching the file.

### Prompt variants: string or array

A prompt value can be a plain string (used for every image in that folder) or
an **array of strings** — then each image gets one variant chosen at random,
and the log shows which one was used:

```
photo1.jpg [variant 2/3] ... redrawn
```

The starter `noBackground` key showcases this: three complete standalone
prompts that replace the removed background with halftone dots, a comic
starburst, or angular geometric panels. The pick is random per image per
run, so the re-roll workflow doubles as a variant re-roll — delete the
output from `redrawn/`, re-run, and the image may draw a different variant.

## Review and re-roll workflow

The pipeline is idempotent: any output file that already exists is skipped.
To get a different result for one photo:

```sh
rm redrawn/some-photo.png vectorized/some-photo.svg
gemini-vectorize redraw     # or just `gemini-vectorize` to also re-vectorize
```

If a redraw keeps coming back weak or muddy on the default (flash) model, tell it
to try harder:

```sh
gemini-vectorize redraw --pro
```

## Prompt troubleshooting

A few fixes that came out of running this pipeline against real (messy)
source photos:

- **Color creeping in** — a red or CMYK tint bleeding into what should be
  pure black-and-white line art. Add a strict-black-ink clause: "STRICTLY
  BLACK INK ON WHITE PAPER ONLY: pure black and pure white, absolutely no
  color anywhere." The starter `prompts.json` already carries this in every
  variant.
- **The model copies logos or trademarks off the source photo** onto
  redrawn clothing. Add: "Replace any brand logos, trademarks, or insignia
  on clothing with plain fabric."
- **Multi-frame sources collapse to one panel** — a photo-booth strip or
  contact sheet redraws as a single image, losing the frames. Drop any
  "single panel" style language from your prompt and replace it with an
  explicit layout-preservation instruction instead, e.g. "Preserve the exact
  panel/frame layout of the source image — do not merge multiple frames
  into one."
- **One detail is wrong but the rest is good** — don't re-roll the whole
  image from the source photo. Take the generated raster from `redrawn/`
  and send it back through the model with a narrow edit instruction
  ("remove the wristwatch, change absolutely nothing else"). The model
  preserves existing linework far better when editing its own output than
  when redrawing from the source photo. Once you're happy with the edit,
  delete the downstream `upscaled/`/`vectorized/` files for that image and
  re-run `vectorize`.
- **Flash-tier misses** — likeness drift, an invented prop or costume piece
  that isn't in the source photo. Usually clears up with one retry: delete
  the output and re-run `redraw`. Still off on the second attempt? Escalate
  that image to `--pro`.

## Running under AI agents

Every stage is a serial loop of API calls — a batch of a few dozen photos
can take minutes, not seconds. Run it to completion; don't orphan a
background process expecting to check back on it later. If a run does get
killed partway through, no cleanup is needed: everything here is
idempotent, so re-running the same command picks up where it left off
(already-written outputs are skipped).

The review loop is delete-outputs-and-rerun: reject a redraw by deleting it
from `redrawn/` (and its SVG from `vectorized/`, if it got that far) and
running the stage again. Review the redraws in `redrawn/` *before* running
`vectorize` — Recraft spends real money per image, so there's no reason to
upscale and vectorize a redraw you're about to reject.

For unattended runs, pass `--yes` to skip the interactive cost-estimate
prompt (a run refuses to proceed on a non-interactive stdin without it) and
`--max-images N` to cap how many new redraws a single invocation generates,
so a misconfigured intake folder can't blow through a budget in one shot.

## Subcommands, flags, and env vars

| subcommand | does |
|---|---|
| `run` (default) | redraw + vectorize, end to end |
| `redraw` | `photos/` + `photos-<variant>/` → `redrawn/` (Gemini only) |
| `vectorize` | `redrawn/` → `upscaled/` → `vectorized/` (Recraft + svg-color-rinse) |
| `models` | list Gemini image models available to your API key |
| `init` | write a starter `prompts.json` in the current directory |
| `balance` | print your Recraft credit balance and exit |

| flag | meaning |
|---|---|
| `--pro` | use the "pro" Gemini image model instead of the default flash model |
| `--no-upscale` | skip the Recraft crisp-upscale step before vectorizing |
| `--prompt "..."` | override the `default` (no-text) redraw prompt for this run |
| `--yes` | skip the pre-redraw cost-estimate confirmation prompt (required if stdin isn't a TTY) |
| `--max-images N` | cap how many NEW redraws this run will generate; excess pending images are deferred to a later run |
| `-h`, `--help` | print usage |

| env var | meaning |
|---|---|
| `GEMINI_API_KEY` | required for `redraw` / `models` |
| `RECRAFT_API_KEY` | required for `vectorize` / `balance` |
| `GEMINI_MODEL` | override the default model id (default: `gemini-3.1-flash-image`) |
| `GEMINI_MODEL_PRO` | override the `--pro` model id (default: `gemini-3-pro-image`) |
| `SVG_COLOR_RINSE` | path to a local svg-color-rinse executable/script |

## Cost guardrails

Before the redraw stage sends a single image to Gemini, it prints a cost
estimate covering the idempotency-filtered pending images — the Gemini
redraw cost for the active model plus the Recraft upscale + vectorize cost
those same images will incur later — and asks `Proceed? [y/N]` unless
`--yes` is passed. If stdin isn't interactive and `--yes` is absent, the run
aborts with a message telling you to pass `--yes`, rather than hanging
forever waiting on a prompt nobody can answer.

`--max-images N` caps how many *new* redraws a single invocation will
generate; anything past the cap is deferred with a summary line, and simply
re-running the command continues where it left off.

The vectorize stage prints your Recraft credit balance (via
`GET /v1/users/me`) at its start and end, best-effort — if the call fails
it's skipped silently rather than interrupting the run. Run
`gemini-vectorize balance` any time to check it on its own.

## Companion: svg-color-rinse

After each SVG is written, this tool runs it through
[svg-color-rinse](https://github.com/dgaidula/svg-color-rinse) `--optimize
--in-place` — collapsing the dozen near-blacks and near-whites that Recraft's
vectorizer tends to leave behind into clean `#000000`/`#ffffff` anchors, then
minifying with svgo. If `SVG_COLOR_RINSE` isn't set, it's run via `npx --yes
svg-color-rinse`. If that's not available either (offline, no npm), the tool
prints a note and moves on — the SVG has already been written either way,
just not rinsed yet.

## macOS note

Image resizing/reformatting (shrinking oversized files, capping dimensions at
4096px, converting TIFF/WebP to PNG/JPEG before sending to either API) uses
macOS's built-in `sips` command. On other platforms `sips` isn't available,
so files are sent to the APIs as-is — this works fine as long as your source
photos are already under 5MB, 4096px, and in PNG/JPEG/WebP format.

## License

MIT
