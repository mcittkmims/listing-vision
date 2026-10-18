# Apartment Lens · Vision service

Handles photos and videos for [Apartment Lens](https://github.com/mcittkmims/apartment-lens). A Python worker with no API: it takes jobs from the database queue and writes results back. It collects photo evidence, but the core service makes the matching decisions.

> **Status:** design stage. There is no code yet.

## Workers

The same code runs as two workers. The **online worker** runs on the server and handles everything that must be fast. The **batch worker** runs once a day on a GPU machine, started with `python -m worker --drain`, and exits when the queue is empty. Both must use the same DINOv2 model file, or their vectors won't be comparable.

| Job | Worker | What it does |
|---|---|---|
| Fingerprint | Online | PDQ hash of each new photo; groups near-identical photos and marks photos shared across a building |
| Photo search | Online | Finds the listing photos closest to an uploaded photo and groups them by property |
| Video search | Online | Searches one sharp frame per shot and lets the frames vote for a property |
| Vectors | Batch | A 384-number DINOv2 vector for every new photo |
| Photo role | Batch | Labels each photo interior, exterior, render, floor plan, map or logo (SigLIP 2) |
| Same-room check | Batch | Confirms with LightGlue whether two look-alike photos show the same room (after version 1) |

## Data and technology

Reads jobs, photos and ads; writes hashes, vectors and check results to `media`, and search results to `app` ([schemas](https://github.com/mcittkmims/apartment-lens/blob/main/database/README.md)). Built with Python 3.12, PyTorch on the GPU machine, ONNX Runtime on the server, `pdqhash`, `transformers`, LightGlue, OpenCV, ffmpeg, PySceneDetect, yt-dlp and `psycopg` with `pgvector`.
