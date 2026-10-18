# Apartment Lens · Vision service

The vision service handles photos and videos for [Apartment Lens](https://github.com/mcittkmims/apartment-lens). It is a Python worker without an API: it takes jobs from the `ops.job` table and writes its results back to the database. It collects evidence about photos but never decides whether two ads describe the same property; that decision belongs to the core service.

> **Status:** design stage. This README describes the plan; there is no code yet.

## Online and batch workers

The same code runs as two workers with different responsibilities.

The online worker runs continuously on the server and handles everything that must happen within seconds. It fingerprints new photos and answers photo and video searches. PostgreSQL `LISTEN/NOTIFY` wakes it as soon as a job arrives, and it runs the image model on the CPU through ONNX Runtime.

The batch worker runs once a day on a GPU machine. It is started manually with `python -m worker --drain`, processes every waiting job, and exits. It computes vectors for the day's new photos, classifies each photo's role, and runs same-room checks on look-alike pairs. It reaches the database over a private network.

Both workers must load the exact same DINOv2 model file. The batch worker builds the vectors that are stored, and the online worker compares uploaded photos against them; if the two used different models, the vectors would not be comparable.

## Jobs

| Job | Worker | Description |
|---|---|---|
| Fingerprint | Online | Computes the PDQ hash of a new photo and groups near-identical photos into clusters. A cluster used by ads with different floors or areas is marked as building scope, because it contains facade, render or show-flat photos that many properties share. |
| Vectors | Batch | Converts every photo that has no vector yet into a 384-dimensional DINOv2 vector and stores it in pgvector. This job needs no queue entries; the batch worker selects the photos itself. |
| Photo role | Batch | Classifies each photo as interior, exterior, render, floor plan, map or logo with SigLIP 2, using text prompts and no training. |
| Same-room check | Batch | Runs LightGlue on pairs of new and existing photos that look similar but are not copies, and stores how many points line up. Each pair is checked only once. |
| Photo search | Online | Converts the uploaded photo into a vector, finds the 50 nearest listing photos, groups them by property, and returns the top five with their matching photos. |
| Video search | Online | Downloads a linked video with yt-dlp, keeps one sharp frame per shot, searches each frame, and lets the frames vote for a property. Below a minimum number of agreeing frames, it reports that there is no confident match. |

## Scope of the first version

The first version includes fingerprints, vectors and both search types. Same-room checks will be added once measurements show that vectors alone produce too many wrong matches. Fine-tuning DINOv2 on the project's own photo pairs, and reading prices from video captions, on-screen text or speech, are possible later extensions.

## Data access

The worker reads jobs from `ops`, photos from `media` and photo storage, and ads from `ingest` to group results by property. It writes hashes, roles, vectors and check results to `media`, search results to `app`, and follow-up jobs to `ops`.

## Technology

The worker uses Python 3.12. DINOv2 ViT-S/14 produces the image vectors, running in PyTorch on the GPU machine and in ONNX Runtime on the server. `pdqhash` computes perceptual hashes, SigLIP 2 from `transformers` classifies photo roles, and LightGlue with ALIKED features performs same-room checks. Pillow, OpenCV, ffmpeg, PySceneDetect and yt-dlp handle images and video, and `psycopg` with `pgvector` connects to the database.
