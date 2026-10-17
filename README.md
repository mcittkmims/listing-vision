# Apartment Lens · Vision service

The photo and video side of [Apartment Lens](https://github.com/mcittkmims/apartment-lens). It is a Python worker with no API. It takes jobs from the `ops.job` table and writes results back. It collects photo evidence but never decides whether two ads are the same flat; the core app does that.

> **Status:** design stage. This README describes the plan; there is no code yet.

## Two copies, same code

| Old laptop (always on) | MacBook (once a day, started by hand) |
|---|---|
| Fingerprints new photos within seconds | Vectors for the day's new photos |
| Photo search (~1 s per photo, CPU) | Photo types (interior, exterior, render…) |
| Video search (visual part) | LightGlue checks on look-alike pairs |
| | Re-indexing or fine-tuning, when needed |

On the Mac, `python -m worker --drain` takes every waiting job, processes it, and exits. On the laptop, the worker runs continuously and wakes up instantly through Postgres `LISTEN/NOTIFY`.

**Both machines must use the exact same DINOv2 model file.** It runs in PyTorch on the Mac and in ONNX Runtime on the laptop. Otherwise the laptop's searches won't line up with the vectors the Mac built.

## What each job does

| Job | Where | What happens |
|---|---|---|
| Fingerprint | Laptop | PDQ hash, then near-identical photos are grouped. A group shared by ads with different floors or sizes is marked `building` (facades, renders, show-flat photos); the rest are `unit`. |
| Vectors | Mac | DINOv2 ViT-S/14 turns each photo without a vector into 384 numbers, stored in pgvector. No job needed: the batch picks up every photo that lacks one. |
| Photo types | Mac | SigLIP 2 labels each photo interior, exterior, render, floor plan, map or logo, from text prompts, with no training. |
| LightGlue check | Mac | Only for new photos that look similar to an existing one but aren't copies. It counts how many points line up: many means the same room, few means a different room. The score is saved, so each pair is checked once. |
| Photo search | Laptop | Vector of the upload, 50 nearest photos, grouped by flat, top 5 with confidence and matching photos. |
| Video search | Laptop | Download a link with yt-dlp, keep one sharp frame per shot (ffmpeg, PySceneDetect), search each frame, then frames vote per flat. Below a minimum number of agreeing frames, the answer is "no confident match". |

## Not in version 1

- LightGlue checks: added once tests show vectors alone make too many mistakes.
- Fine-tuning DINOv2 on your own pairs.
- Prices from videos (caption, then OCR, then Whisper).

## Database

- **Reads:** `ops.job`, `media`, `ingest` (to group results by flat), and the photo store.
- **Writes:** hashes, types, vectors and scores in `media`; search results in `app`; jobs in `ops`.

## Runs on

The laptop (always on) and the MacBook (daily batch). The MacBook reaches the database over the home network.

## Tech

| Purpose | Tool |
|---|---|
| Language | Python 3.12 |
| Photo vectors | DINOv2 ViT-S/14 (PyTorch on the Mac, ONNX Runtime on the laptop) |
| Near-copy hash | `pdqhash` |
| Photo types | SigLIP 2 (`transformers`) |
| Same-room check | LightGlue + ALIKED |
| Images and video | Pillow, OpenCV, ffmpeg, PySceneDetect, yt-dlp |
| Database | `psycopg`, `pgvector` |
