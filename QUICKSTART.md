# Photon Quickstart

Get up and running in under 5 minutes. No programming knowledge required.

---

## Step 1: Install Photon

Open your terminal app and paste this command:

```bash
pip install photon-imager
```

> **Don't have `pip`?** If you see "command not found", install Python first from [python.org/downloads](https://www.python.org/downloads/). Python includes `pip` automatically.

To verify it worked:

```bash
photon --version
```

You should see something like `photon 0.7.10`.

---

## Step 2: Download the AI models

Photon uses AI models that run entirely on your computer (nothing is sent to the cloud). You only need to download them once:

```bash
photon models download
```

This downloads ~350 MB of model files. It may take a few minutes depending on your internet connection.

---

## Step 3: Process your first image

### Option A: Interactive mode (recommended for beginners)

Just type:

```bash
photon
```

This opens a guided menu that walks you through everything step by step.

### Option B: Process a single image

```bash
photon process photo.jpg
```

Replace `photo.jpg` with the path to your image. **Tip:** you can drag an image file from Finder (macOS) or your file manager (Linux) directly into the terminal window, and it will paste the file path for you.

### Option C: Process a whole folder of images

```bash
photon process ./my-photos/
```

Replace `./my-photos/` with the path to your folder. **Tip:** drag the folder from Finder into the terminal, same as above. Photon automatically finds all images inside the folder, including subfolders.

---

## What you get back

Photon outputs a JSON result for each image containing:

| Field | What it means |
|-------|---------------|
| **tags** | What's in the image (e.g. "dog", "beach", "sunset") |
| **embedding** | A 768-number fingerprint for finding similar images |
| **metadata** | Camera info, GPS coordinates, date taken, etc. |
| **content_hash** | A unique ID for the exact file (for finding duplicates) |
| **perceptual_hash** | A fuzzy ID for nearly-identical images |
| **thumbnail** | A small preview image (base64-encoded) |

### Save results to a file

```bash
photon process ./my-photos/ -o results.json
```

For large batches, JSONL format writes one result per line (better for big datasets):

```bash
photon process ./my-photos/ -o results.jsonl -f jsonl
```

---

## Optional: Add AI descriptions

Photon can also generate natural-language descriptions of your images using an LLM. This is optional and requires an API key from one of these providers:

- **Ollama** (free, runs locally) -- [ollama.com](https://ollama.com)
- **Anthropic** (paid API) -- requires `ANTHROPIC_API_KEY`
- **OpenAI** (paid API) -- requires `OPENAI_API_KEY`

Example with Ollama (free, local):

```bash
photon process photo.jpg --llm ollama --llm-model llama3.2-vision
```

Example with Anthropic:

```bash
export ANTHROPIC_API_KEY="your-key-here"
photon process photo.jpg --llm anthropic
```

---

## Common options

| Option | What it does |
|--------|--------------|
| `-o results.json` | Save output to a file instead of printing it |
| `-f jsonl` | Use JSONL format (one result per line) |
| `-p 8` | Use 8 parallel workers (default: 4, higher = faster) |
| `--quality high` | Use the higher-detail 384px model (slower but more accurate) |
| `--skip-existing` | Skip images that were already processed (useful for re-runs) |
| `--no-thumbnail` | Skip thumbnail generation |
| `--no-embedding` | Skip embedding generation |
| `--no-tagging` | Skip tag generation |
| `-v` | Show detailed logs (helpful for troubleshooting) |

---

## Troubleshooting

**"command not found: photon"**
Make sure `pip install photon-imager` completed without errors. You may need to restart your terminal, or use `pip3` instead of `pip`.

**"No supported image files found"**
Photon supports JPEG, PNG, WebP, GIF, TIFF, and HEIC/AVIF. Check that your file is one of these formats.

**"Required models not installed"**
Run `photon models download` first. The models are required for embedding and tagging.

**Processing is slow**
Try `--no-tagging` to skip the most expensive step, or use `-p 8` to increase parallelism. The first run on a new image set is slower because Photon builds a vocabulary cache.

---

## Next steps

- Run `photon config init` to create a config file you can customize
- Run `photon config show` to see all current settings
- See [`docs/usage.md`](docs/usage.md) for the full reference
