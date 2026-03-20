---
name: paper
description: "Process academic papers into structured Obsidian notes. Use this skill whenever the user mentions /paper, wants to add a paper to their Obsidian vault, references an arxiv link/ID they want summarized, asks to enrich or update existing paper notes, wants to find backlinks between papers, or wants to extract and organize academic paper information. Triggers on: arxiv links, PDF paper paths, requests to summarize/add/process papers, requests to enrich or batch-update paper notes, requests to find/add backlinks."
argument-hint: <path or URL to paper PDF>
user-invocable: true
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, WebFetch, Agent
---

# Paper Skill

Process academic papers into structured Obsidian notes with figures, tags, backlinks, and BibTeX citations.

## Modes

### Mode 1: Single Paper (default)
**Usage:** `/paper <pdf_path_or_arxiv_id_or_url>`

Accepts:
- ArXiv ID: `2209.14988`
- ArXiv URL: `https://arxiv.org/abs/2209.14988`
- Local PDF path: `/path/to/paper.pdf`

### Mode 2: Batch Enrichment
**Usage:** `/paper --enrich`

Enriches all existing papers in `Papers/` with more detail, missing tags, backlinks, and citations.

### Mode 3: Find Backlinks
**Usage:** `/paper --backlinks`

Scans all papers in the vault and discovers missing cross-references between them. Adds `[[WikiLink]]` backlinks where papers reference each other but the links are missing.

---

## Single Paper Workflow

### Step 1: Obtain the PDF

If the input is an arxiv ID or URL, extract the numeric ID and download:
```bash
mkdir -p /tmp/paper_extract
curl -L -o /tmp/paper_extract/paper.pdf "https://arxiv.org/pdf/<ARXIV_ID>"
```

If it's a local PDF path, use it directly.

### Step 2: Extract text and figures

Run the extraction script bundled with this skill. Dependencies are declared in the workspace-root `pyproject.toml` and managed by `uv`. The `nougat` optional extra enables Nougat (neural PDF→markdown with LaTeX equations); without it, plain-text extraction via PyMuPDF is used as fallback.

**IMPORTANT:** `uv run` cannot handle spaces in absolute script paths (the Google Drive path contains `My Drive`). Always `cd` into the project root first and use the relative path:

```bash
cd "<project_root>" && uv run --extra nougat .claude/skills/paper/extract.py "<pdf_path>" /tmp/paper_extract/images
```

To skip Nougat (faster, but no equation support):
```bash
cd "<project_root>" && uv run .claude/skills/paper/extract.py "<pdf_path>" /tmp/paper_extract/images --no-nougat
```

Where `<project_root>` is the working directory (the vault root containing `pyproject.toml`).

This produces:
- `/tmp/paper_extract/extracted_text.txt` — full text by page
- `/tmp/paper_extract/figures_meta.json` — metadata for extracted figures
- `/tmp/paper_extract/images/fig*.png` — extracted figure images

Read the extracted text file and figures metadata to understand the paper.

### Step 3: Select overview figure and copy images

Look at `figures_meta.json` to find the best overview/pipeline figure (usually the first large figure, or one with a caption mentioning "overview", "pipeline", "method", or "framework").

Copy the selected overview image to the Obsidian vault:
```bash
cp /tmp/paper_extract/images/<best_fig>.png "<vault_images_path>/<PaperName>.png"
```

The vault images path can be found by listing `Papers/images/` via `mcp__obsidian__list_directory`. Copy images using bash to the actual filesystem path of the vault. The vault root is wherever the Obsidian vault lives on disk — find it by checking where existing images are stored.

To find the vault's filesystem path, check:
```bash
ls ~/Library/Mobile\ Documents/iCloud~md~obsidian/Documents/
```
Or look for the vault by checking where Obsidian stores data. The mcp__obsidian tools handle read/write of notes, but for binary files (images), copy directly to the filesystem.

### Step 4: Find BibTeX citation

Search for the paper's BibTeX citation. Try these sources in order:

1. **ArXiv API** (if arxiv paper): The extracted text often contains the citation or you can construct one from the metadata
2. **Semantic Scholar API**:
   ```bash
   curl -s "https://api.semanticscholar.org/graph/v1/paper/ArXiv:<ARXIV_ID>?fields=title,authors,year,venue,externalIds,citationStyles"
   ```
3. **Construct from extracted text**: Use the title, authors, year from the paper itself

The cite key should follow the pattern: `firstauthorlastname + year + firstwordoftitle` (e.g., `poole2022dreamfusion`).

### Step 5: Analyze the paper

From the extracted text, identify:

1. **Title**: The full paper title
2. **Links**: ArXiv URL, project page (often in the first page or abstract)
3. **One-line summary**: What does the paper do, in one concise line
4. **Key contributions**: The novel ideas this paper introduces (2-4 bullet points, with some detail)
5. **Key deltas from prior work**: What's specifically new compared to predecessors — be explicit about what changed and why it matters (2-3 bullet points)
6. **Method details**: Core technical approach, key equations if relevant
7. **Tags**: Match existing tags in the vault (see Tag Reference below), add new ones only if truly needed
8. **Backlinks**: Which papers already in the vault are referenced or related
9. **Implementation notes**: Training details, hyperparameters, practical insights, limitations
10. **Missing critical references**: Papers referenced heavily that aren't in the vault yet

### Step 6: Write the Obsidian note

Use `mcp__obsidian__write_note` to create the paper note at `Papers/<PaperName>.md`.

The note filename should be a short recognizable name (e.g., `DreamFusion`, `ReconFusion`, `NVDiffRec`) — no spaces, PascalCase.

**IMPORTANT — Writing Style:**

The Idea section is the heart of the note. Be more expressive and detailed than a bare-bones listing. The reader should understand:
- What problem the paper solves and why it matters
- The core technical insight (the "aha!" moment)
- How it improves on prior work specifically (not just "improves on X" but HOW and by how much)
- Key equations or formulations that capture the method

Think of it as writing notes for a researcher who wants to quickly recall what makes this paper important and how it works, without re-reading the paper. Use clear, direct language. Bullet points are fine but make them substantive.

**Format — follow this exactly (no YAML frontmatter):**

```
# <Full Paper Title>
[ArXiv](https://arxiv.org/abs/<ID>) - [Project Page](<url>)

![Overview](images/<PaperName>.png)
## Idea

- <One-line summary of what the paper does> #Tag1 #Tag2
- <Key contribution 1 — explain the core technical insight with enough detail to understand it>
- <Key contribution 2 — what's novel about the approach>
- <Key delta from prior work — e.g., "Unlike [[DreamFusion]] which uses SDS loss, this paper proposes VSD which reduces over-saturation by...">
- <Key delta 2 — be specific about improvements>
- <Core equation if applicable>: $$\mathcal{L} = ...$$
- Uses [[BacklinkedPaper]] for <what purpose>

## Notes

- <Implementation detail 1>
- <Training specifics — batch size, learning rate, training time if mentioned>
- <Limitations or failure cases>
- <Practical insights that would help someone reimplementing this>

## Cite

` ` `
@article{citekey,
  author = {...},
  title = {...},
  journal = {...},
  year = {...}
}
` ` `
```

(The triple backticks in the Cite section should be actual triple backticks, not spaced out as shown above — that's just to avoid markdown escaping issues in this skill file.)

### Step 7: Handle missing closely related methods

After writing the note, check for papers that are **closely referenced** in the Idea or Notes sections but don't yet exist in the vault. This includes:

1. **Direct comparisons**: Methods the paper explicitly benchmarks against or contrasts with (e.g., "Unlike Gen3R which...", "outperforms FlashWorld by...")
2. **Building blocks**: Methods the paper directly builds on or extends (e.g., "built on π3", "uses the decoder from X")
3. **Concurrent/sibling methods**: Papers solving the same problem mentioned as concurrent or closely related work

For each such missing paper:
- Search for it on arxiv (by title or known arxiv ID from the references section)
- Download the PDF, extract text/figures
- Create a full note following the same workflow (Steps 1-6)
- Backlink from the original paper using `[[WikiLink]]` syntax

**Scope**: Process any paper that appears by name in the Idea section of the note you just wrote, or is directly compared against in results tables/text. Do NOT recursively chase every citation — only methods that a reader would need indexed to understand the relationships described in the note.

**Example**: If OneWorld's note says "Unlike Gen3R which compresses 3D features..." and "outperforms FlashWorld", both Gen3R and FlashWorld should get their own notes if they don't already exist in the vault.

### Step 8: Verify

After writing, read the note back with `mcp__obsidian__read_note` to confirm it looks correct.

---

## Batch Enrichment Workflow (`--enrich`)

### Step 1: List all papers

Use `mcp__obsidian__list_directory` for `Papers/` and `Papers/GeneralTechniques/` to get all paper files.

### Step 2: Read and assess each paper

For each paper, use `mcp__obsidian__read_note` to read its current content. Assess completeness:
- Is the title just "Full name" (template placeholder)?
- Is the Idea section empty or very sparse (fewer than 3 substantive bullet points)?
- Is the Cite section empty?
- Are there no tags?
- Are there obvious missing backlinks?
- Is the overview image missing?

### Step 3: Enrich sparse papers

For papers that need enrichment:
1. Find the arxiv link from the note (if present)
2. If arxiv link exists, download the PDF and extract text/figures (same as single paper workflow)
3. If no arxiv link, search for the paper by title using web search
4. Enhance the Idea section with more detail — key contributions, deltas, equations
5. Add missing tags from the tag reference
6. Add backlinks to other papers in the vault
7. Find and add BibTeX citation if missing
8. Use `mcp__obsidian__write_note` to update the note (preserve any existing good content)

### Step 4: Report

After processing all papers, summarize what was enriched and what was left unchanged.

---

## Tag Reference

Use these existing tags consistently. Only create new tags if the paper covers a genuinely new concept not captured by these:

| Tag | Use for |
|-----|---------|
| #TextTo3D | Methods generating 3D from text prompts |
| #ImageTo3D | Methods generating 3D from single/few images |
| #Diffusion | Papers using diffusion models |
| #NovelViewSynthesis | Novel view synthesis methods |
| #NeRF | Neural radiance field methods |
| #Mesh | Methods that output or optimize meshes |
| #BRDF | Papers modeling BRDFs/materials |
| #Relightable | Methods producing relightable outputs |
| #TexturedPBRMesh | Methods outputting textured PBR meshes |
| #EnvironmentMap | Papers using environment map lighting |
| #SparseInput | Methods working from sparse input views |
| #SDS | Score Distillation Sampling and variants |
| #MaterialEstimation | Material/texture estimation methods |
| #3DReconstruction | General 3D reconstruction |
| #GaussianSplatting | 3D Gaussian Splatting methods |
| #Transformer | Transformer-based architectures |
| #LargeReconstruction | Large-scale reconstruction models (LRM-style) |

## Papers Currently in the Vault (for backlinks)

**Papers/**: ControlMat, DMTet, DiffusionPosteriorIllumination, DreamFusion, Fantasia3D, GeNVS, HexaGen3D, Hifa, IntrinsicImageDiffusion, IntrinsicImageDiffusionforMaterialEstimation, LGM, LRM, LucidDreamer, Magic3D, Matlaber, NVDiffRec, Nuvo, PixelNeRF, ProlificDreamer, ReconFusion, RichDreamer, SparseFusion, TextureDreamer, UniDream, Zero-1-to-3, Zero1to3, ZeroShape

**Papers/GeneralTechniques/**: PreFilteredImportanceSampling, PreIntegratedLighting, SurfaceGradientBasedBumpMapping

When mentioning any of these papers in a note, always use `[[WikiLink]]` syntax (e.g., `[[DreamFusion]]`, `[[NVDiffRec]]`).

Before writing a new note, re-check the current list of papers using `mcp__obsidian__list_directory` on `Papers/` — new papers may have been added since this skill was written.

## Find Backlinks Workflow (`--backlinks`)

### Step 1: List all papers

Use `mcp__obsidian__list_directory` for `Papers/` and `Papers/GeneralTechniques/` to build a complete list of paper filenames (without `.md` extension). These are the potential backlink targets.

### Step 2: Read each paper and identify missing links

For each paper note, read its content with `mcp__obsidian__read_note`. Then:

1. **Check existing backlinks**: Find all `[[WikiLink]]` references already present
2. **Search for unlinked mentions**: Look for paper names mentioned in plain text that should be `[[WikiLinked]]`. Check for:
   - Exact paper name matches (e.g., "DreamFusion" without `[[]]`)
   - Common variations (e.g., "TRELLIS 2" for `[[TRELLIS2]]`, "Zero-1-to-3" for `[[Zero1to3]]`)
   - Method names referenced in the Idea section that correspond to vault papers
3. **Identify thematic connections**: Papers that clearly relate to each other but lack cross-references:
   - Papers using the same technique (e.g., both use SDS → should link to [[DreamFusion]])
   - Papers building on the same base method
   - Papers compared against each other in their respective results sections

### Step 3: Add missing backlinks

For each paper with missing links, use `mcp__obsidian__patch_note` to:
- Replace plain-text paper name mentions with `[[WikiLink]]` syntax
- Add "Related: [[Paper]]" lines in the Idea section where a thematic connection exists but the paper wasn't explicitly mentioned

**Rules:**
- Only add backlinks where there's a genuine relationship (shared method, comparison, building-on)
- Don't add spurious links just because two papers exist in the same domain
- Prefer `patch_note` over `write_note` to minimize changes
- Process papers in batches using background agents for speed

### Step 4: Report

Summarize all backlinks added, grouped by paper.

---

## Cleanup

Cleanup of `/tmp/paper_*` directories is ALWAYS allowed and should be done automatically at the end of every workflow. These are temporary extraction artifacts with no user data. Use:
```bash
rm -rf /tmp/paper_extract /tmp/paper_*
```
