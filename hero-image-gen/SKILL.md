---
name: hero-image-generation
description: Generate brand-aligned hero images for work.flowers blog posts and social content using Google Gemini via the Zapier MCP connector. Use this skill whenever Dennis asks for a hero image, blog image, social image, post visual, or any generated artwork for work.flowers content — even if he doesn't explicitly say "hero image". Trigger when the user mentions creating, drafting, or generating an image for a blog post, newsletter (Flow Statements), LinkedIn post, or any other piece of content. Also trigger when the user asks to "make an image for…", "illustrate this post", "generate a visual for…", or refers to image generation alongside Gemini, Nano Banana, Imagen, or Zapier. Walks through concept generation, prompt drafting, user approval, then calls Nano Banana Pro via Zapier and saves the result to disk.
---

# Hero Image Generation

Translate a work.flowers content piece into a brand-aligned hero image. The skill produces three outputs in order: concept directions for the user to choose from, a detailed image generation prompt for them to approve, and finally a generated image saved to disk and shown inline.

The skill is opinionated about brand because the failure mode is "generic AI vector stock" — flat, sterile, cliché-ridden imagery that undermines the editorial tone of work.flowers content. Texture, restraint, and considered composition are what make the visual identity recognisable.

**Always pass the relevant reference image to Gemini.** The reference images in this skill's `references/` folder do more work than any prompt description can. The `Zapier:google_ai_studio_gemini_generate_image` tool accepts a `files` parameter — use it. Describing the aesthetic in words is always lossier than showing it.

---

## Workflow

The workflow has three checkpoints. Do not skip ahead — each step depends on context from the previous one.

1. **Read the post → suggest 3–4 concept directions, each tagged Style A or Style B**
2. **User picks a direction → draft the detailed prompt and pause for approval**
3. **User approves → call Gemini via Zapier MCP (with reference image attached) and save the image**

### Step 1 — Read the source content first

Before suggesting anything, read the post the image is for. The image should reflect the post's **core argument or tension**, not just its topic.

- If the user pastes the post or links to it, read it.
- If the user references a Notion page (e.g. blog post, Flow Statements draft, LinkedIn post in the Social Content DB), fetch it via the Notion MCP connector before continuing.
- If neither is available, ask the user for the post or its core thesis. Do not guess.

The reason this matters: a post about automation should feel **human**, not robotic. A post about AI tools should feel **considered**, not hype-y. Without reading the actual argument, you'll default to surface-level topic illustrations (the "robot at a keyboard" failure mode).

### Step 2 — Suggest 3–4 concept directions

Present concepts as labelled options. Each concept should include:

- A short name (2–4 words)
- A one-paragraph description of the visual idea
- The emotional tone or message it conveys
- **Which house style it uses** (Style A or Style B — see below)

Keep the set varied. Mix both styles across the options — don't put all four in the same style. A typical good mix: two Style A options (one literal, one with figures), two Style B options (one with metaphor objects, one purely abstract).

#### The two house styles

work.flowers has two named visual styles. Every concept must map to one of them. See `references/style-a-editorial-illustration.png` and `references/style-b-paper-craft.png` for the canonical examples.

**Style A — Editorial illustration (2.5D)**

The "human at work" mode. Best for posts about practice, craft, daily work, workflows, or when a human subject grounds the post.

- **Medium:** Vector-led illustration with painterly soft shading. Forms have volume and dimension — never pure flat vector.
- **Composition:** Grounded foreground subject (a desk, a person at work, a workspace) sitting in front of a decorative abstract backdrop.
- **Backdrop:** Flowing ribbon and wave forms in indigo/violet/blue, occupying ~30–50% of the canvas. Halftone dot patterns overlaid on flat colour areas.
- **Subject treatment:** Figures (when present) feel natural and expressive — not stiff. Glasses, sweaters, considered details. Faces in 3/4 view.
- **Lighting:** Warm directional light from above (often a hanging lamp or window), creating soft volumetric glow.

**Style B — Paper craft / physical metaphor**

The "abstract concept" mode. Best for posts that argue something conceptual, where there's no obvious human subject, or where the thesis is best captured as a metaphor.

- **Medium:** Photographic paper sculpture — looks like a physical set built from cut, folded, and crumpled paper, photographed under directional light. (Gemini renders this convincingly; you don't actually need physical paper.)
- **Composition:** Constructed objects in the foreground (paper stacks, origami forms, geometric solids), abstract paper-wave backdrop behind. Strong directional light beam often cutting through the scene.
- **Backdrop:** Layered cut-paper waves in tonal blues and violets, with halftone dot patterns and subtle geometric paper textures (diamond patterns, wave lines).
- **Subject treatment:** No people, no screens, no UI. Pure object metaphor. Peach/ochre paper objects appear as warm focal points.
- **Lighting:** Strong directional warm light creating real shadow geometry. The light beam is often the compositional spine.
- **Frame:** Deckled / torn paper edge around the image (visible in `style-b-paper-craft.png` as a soft white torn border).

#### Concept directions to draw from

These are starting points, not a fixed menu. Mix, adapt, or invent based on the post's argument.

- **The desk / workspace** *(Style A)* — Overhead or 3/4 flat-lay of a tidy workspace with objects reflecting the post's themes. Warm and human. Good when the post is about practice, craft, or daily work.
- **Person at work** *(Style A)* — Side or 3/4 view of a figure engaged with their work, abstract backdrop ribbons behind. Good for posts where a human subject grounds the argument.
- **Signal in the noise** *(Style B)* — Light beam cutting through clutter, ordered stacks of paper next to crumpled rejects. Good for focus, systems, cutting through overwhelm.
- **Constructed metaphor** *(Style B)* — Specific paper objects representing the post's thesis (e.g. paper bridges, towers, machines, paths). Good for posts arguing a conceptual frame.
- **The pipeline / diagram** *(Style A)* — Isometric or illustrated flow showing inputs → process → output, hand-sketched feel. Good for workflow and automation posts.
- **The edit** *(Style A)* — Hands interacting with physical media (pen on paper, marked-up printout) with subtle digital element behind. Good for posts about human judgment or craft.

Once concepts are presented, wait for the user to pick one (or ask for a recommendation).

### Step 3 — Draft the detailed prompt and pause for approval

Once a direction is chosen, write a **detailed image generation prompt** as a single continuous paragraph (not a bulleted list). Use the appropriate scaffolding below depending on which style was chosen.

#### Style A prompt scaffolding (defaults to include)

Weave these elements into the prompt unless the concept explicitly overrides one:

- **Medium:** "2.5D editorial illustration, vector forms with painterly soft shading and subtle gradients"
- **Backdrop:** "decorative abstract backdrop of flowing ribbon and wave forms in Persian Indigo `#2E1B88`, Russian Violet `#4E1B61`, and Azure `#1479E1`, with halftone dot patterns overlaid on flat colour areas"
- **Foreground subject:** described per concept
- **Lighting:** "warm directional light from above creating soft volumetric glow"
- **Warm accent:** "single Ochre `#E17A14` or Peach `#F6C696` focal element — one warm note only, not scattered"
- **Base:** "white `#FFFFFF` breathing space framing the composition"
- **Mood:** one or two words (e.g. "considered", "human", "focused")
- **Aspect ratio:** `16:9` for blog hero, `1:1` for social square, `4:5` for LinkedIn portrait

#### Style B prompt scaffolding (defaults to include)

- **Medium:** "photographic paper sculpture scene, looks like a physical set built from cut and folded paper, photographed under studio lighting"
- **Backdrop:** "layered cut-paper wave forms in tonal Persian Indigo `#2E1B88`, Russian Violet `#4E1B61`, and Non-Photo Blue `#9CE1FC`, with halftone dot patterns and subtle geometric paper textures (diamond and wave patterns)"
- **Foreground objects:** described per concept (paper stacks, origami forms, geometric solids, etc.)
- **Lighting:** "strong directional warm light creating a visible light beam cutting diagonally through the scene, with real cast shadows"
- **Warm accent:** "Ochre `#E17A14` and Peach `#F6C696` paper objects as warm focal points, sparingly placed"
- **Frame:** "soft deckled / torn paper edge around the image, off-white border"
- **Mood:** one or two words (e.g. "deliberate", "quiet", "contemplative")
- **Aspect ratio:** as above

#### Universal exclusions (always add)

Append these to every prompt unless the concept explicitly calls for one:

- No text, typography, or labels embedded in the image
- No stiff or generic figure poses; no stock-looking people
- No faceless human silhouettes — these consistently render as uncanny or creepy, especially for AI/tech themes. If a person is needed, give them more detail; if abstraction is needed, drop the human form entirely
- No "robot hands on keyboard" / "glowing brain" / "abstract neural network" cliché
- No pure flat vector with no texture
- No drop shadows that read as Microsoft Office clip art
- No hyper-saturated or neon colour palettes
- No disembodied limbs — translucent or floating hands, arms reaching from off-frame, etc. The model renders these literally and they read as ghostly or threatening
- No watermarks or signatures

After drafting, **stop and show the prompt to the user**. Do not call Gemini yet. Ask explicitly whether to proceed with generation, or if they want to revise. This checkpoint exists because image generation is irreversible per call — if the prompt is wrong, the output will be wrong, and credits/time are wasted.

### Step 4 — Generate via Zapier MCP (with reference image)

Once the user approves, call the Zapier MCP tool `Zapier:google_ai_studio_gemini_generate_image`. Use these parameters:

- `model`: `gemini-3-pro-image-preview` (Nano Banana Pro — the current default for this skill)
- `prompt`: the approved prompt from Step 3
- `instructions`: **must include the phrase** `"PROCEED IMMEDIATELY ... Do not ask any further questions."` alongside a brief description of what's being generated (e.g. "Generate a hero image for a work.flowers blog post about deterministic vs agentic AI, in Style B paper-craft aesthetic. PROCEED IMMEDIATELY ... Do not ask any further questions."). Without this, the Zapier resolver tends to stop and ask follow-up questions about optional sampling parameters (temperature, top P, reference images) before running.
- `output_hint`: `"the generated image file URL or base64 data"`
- **`files`: pass the matching reference image** — `references/style-a-editorial-illustration.png` for Style A concepts, `references/style-b-paper-craft.png` for Style B concepts. This is the single highest-leverage parameter — Gemini will match the reference's texture, palette, and finish far more reliably than from prompt text alone.

If the user has provided their own reference image (e.g. a previous post's hero they want to match), use that instead of or alongside the canonical reference.

The tool may return the image as a URL, base64-encoded data, or both, depending on Zapier's response shape. Handle both cases.

#### Verifying the model actually ran

The Zapier resolver can silently substitute the `model` value with one of its own — it will return a successful response but the image will have been generated by a different model than requested. **Always check `resolvedParams.model` in the response.** If `status: "guessed"` or `reason: "llm-guess"`, the resolver overrode your value. Look for `status: "locked"` and `reason: "exact-prefill-choices-match"` to confirm the right model ran. If the wrong model ran, stop and tell the user — don't pretend the generation worked.

#### When the resolver refuses to honour the model

If the resolver insists only Imagen models are available despite the schema advertising Gemini support, the Zapier action's pre-configured options are likely interfering. The fix is on the user's side: they need to go to https://mcp.zapier.com, edit the "Google AI Studio (Gemini): Generate Image" action, and either remove the pre-configured `model` value or hard-code only the `apiVersion` field. Tell the user this rather than silently falling back to Imagen.

### Step 5 — Save to disk and show inline

Save the image to `/mnt/user-data/outputs/` with a descriptive filename based on the post title or theme (e.g. `hero-deterministic-vs-agentic-ai.png`). Then call `present_files` to make it visible.

If the response gives a URL: download the bytes (e.g. `curl -L -o <path> <url>`) and save.
If the response gives base64: decode and write to disk.

After saving, briefly note (one sentence) what was generated and offer two follow-ups: regenerate with a tweaked prompt, or proceed to publish/upload.

---

## Brand reference

All concepts and prompts must align with the work.flowers visual identity. Hex codes are not negotiable — use them by hex, not by approximate name, so Gemini has no ambiguity.

### Colour palette

| Name | Hex | Role |
|---|---|---|
| Persian Indigo | `#2E1B88` | Primary anchor — headers, hero backgrounds, strong accents |
| Azure | `#1479E1` | Primary interactive / highlight — UI elements, glows, links |
| Russian Violet | `#4E1B61` | Deep accent — depth, sophistication, paired with blues |
| Non-Photo Blue | `#9CE1FC` | Light backgrounds, soft fills, secondary highlights |
| Ochre | `#E17A14` | Warm accent — CTAs, focal details, single warm highlight |
| Peach | `#F6C696` | Soft secondary — shadows, hover states, subtle warmth |
| Eerie Black | `#1F1F1F` | Body text, icons, dividers |
| White | `#FFFFFF` | Page background, negative space, cards |

### Signature elements (encode by default)

These elements appear consistently across both house styles and are what make a work.flowers image identifiable. Include them by default:

- **Halftone dot overlays** on background regions — not generic "grain", specifically dotted texture as seen in both reference images.
- **Layered, sculptural backdrops** — flowing abstract shapes (ribbons in Style A, paper waves in Style B).
- **Strict palette discipline** — indigo/violet dominant, blues for highlights, peach/ochre as a *single* warm note, white as breathing room. Never deviate.
- **Warm directional light** creating real shadow geometry — never even ambient lighting.
- **Editorial composition** — generous negative space, clear hierarchy, every element looks intentional.

### Visual style principles (universal)

- **Contemporary and layered** — never sterile or generically AI-generated.
- **Texture is the medium**, not decoration. Halftone dots, paper grain, painterly shading — these prevent the flat digital feel that gives away AI generation.
- **Bold yet approachable** — generous negative space, clear hierarchy.
- **Ochre and Peach are accent colours only** — use sparingly for emphasis, never as a dominant fill. One warm focal point per image.
- **Figures, if any, should feel natural and expressive** — not generic or stiff. Often better to omit figures entirely (use Style B) than to use stock-looking ones.
- The finish should feel **editorial and crafted**, not AI vector stock. If the prompt could plausibly have been written for an Adobe Stock listing, it needs more specificity.

---

## Output format

### When suggesting concepts

Present each concept as a labelled, numbered option with: short title, one-paragraph description, *and the style tag (Style A or Style B)*. End with a question asking which direction to develop further.

### When drafting the final prompt

Output the prompt as a single continuous paragraph (not bullets), ready to paste into any image generation tool. Then add a short follow-up: *"Want me to generate this with the [Style A/B] reference attached, or tweak anything first?"*

### When saving the image

Save to `/mnt/user-data/outputs/<descriptive-filename>.png` and call `present_files`. Keep the surrounding message short — one sentence describing what was generated, then offer regenerate or proceed-to-publish.

---

## Edge cases and notes

- **Reference image is mandatory by default.** Always pass the matching style reference to Gemini's `files` parameter unless there's a specific reason not to. This is the single biggest lever for brand consistency.
- **User-supplied reference.** If the user provides their own reference (e.g. "match the style of last week's post"), pass that one. You can also pass *both* the canonical reference and the user's reference — Gemini accepts multiple files.
- **Multiple aspect ratios needed.** If the user wants both a blog hero (16:9) and a LinkedIn version (4:5 or 1:1), generate them as separate calls with adjusted aspect ratio in each prompt. Don't try to do it in one call.
- **Zapier MCP not connected.** If the Zapier connector isn't available or the tool call fails, fall back to outputting the finished prompt as plain text so Dennis can paste it into Gemini directly. Don't pretend the generation succeeded.
- **Notion "Prompt (Optional)" field.** If the post being illustrated lives in a Notion database that has a `Prompt (Optional)` property, save the final approved prompt there via the Notion connector. This mirrors the existing Notion AI workflow and makes prompts reusable.
- **Iteration vs. concept rework.** If the first generation isn't right, ask which element is off (composition, palette, mood, texture) before regenerating — don't just rerun the same prompt and hope. Common fixes: swap which style reference is attached, reduce the warm accent count, increase halftone visibility, or shift the dominant ribbon colour. But if a third or fourth regeneration still reads wrong, the underlying concept is probably the issue, not the prompt. Stop tweaking and reopen Step 2: suggest fresh concept directions instead. Visual metaphors can be structurally wrong (e.g. trying to depict an AI as a translucent partner figure) in ways that no amount of prompt tuning fixes.
- **Style mixing.** Don't try to combine Style A and Style B in the same image. They're distinct house styles; mixing reads as confused. If a concept seems to want both, choose the one that better serves the post's core argument.
- **IP-adjacent metaphors.** When a user references a copyrighted visual as inspiration (Marvel Infinity Stones, Batman's utility belt, specific film/game aesthetics, branded characters), flag the IP issue explicitly before drafting the prompt. Then translate the structural idea into a brand-safe rendering — e.g. "a curated set of distinct capabilities held together" can be rendered as a generic gauntlet with brand-coloured stones, deliberately steering away from the canonical look. Add the specific copyrighted elements to the exclusions list (e.g. "Avoid: Thanos, gold/bronze metallic finish, rivets, knuckle housings").
