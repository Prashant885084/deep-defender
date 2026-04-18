# Deep Defender

Deep Defender is a Flask-based prototype for detecting potentially fake or synthetic media. It combines a lightweight web UI, postgreSQl-backed detection history, URL-based media analysis, and optional model code for image and audio deepfake detection.

This repository  have been built as a hackathon/demo project, and the current implementation mixes production-like pieces with simulated detection flows. The README below reflects the code as it exists today.

## What the project does

- Analyzes image and audio and URLs through `/api/url/analyze` /api/image/detect /api/voice/detect
- Extracts linked media from HTML pages when the URL is not a direct media file
- Stores image, voice, and URL detection history in `detections.db`
- Exposes analytics and history endpoints for the UI
- Includes optional TensorFlow and Hugging Face model wrappers for future or partial integration

## Current behavior

There are two different detection paths in the codebase:

1. `Upload Media` flow in the UI
   This is currently demo behavior. The frontend sends only the filename to the backend, and the backend generates a random fake/real result for image and voice uploads.

2. `Analyze URL` flow in the UI
   This is the more real analysis path. The backend fetches a remote image or audio file, tries to use optional local TensorFlow weights if present, and otherwise falls back to heuristic analysis based on media characteristics and URL metadata.

## Tech stack

- Backend: Flask
- Frontend: HTML, CSS, vanilla JavaScript
- Database: PostgreSQL
- Optional ML/runtime dependencies:
  - TensorFlow
  - OpenCV
  - librosa
  - NumPy
  - Pillow
  - PyTorch
  - transformers

## Project structure

```text
deep-defender/
├── app.py                          # Main Flask app, routes, DB setup, URL analysis
├── .app.py                         # Small launcher that calls main()
├── index.html                      # Frontend UI
├── script.js                       # Frontend interactions and API calls
├── style.css                       # Frontend styling
├── detections.db                   # SQLite database used by the app
├── image_preprocess.py             # Image preprocessing for TensorFlow model path
├── image_xception.py               # Xception-based image classifier definition
├── audio_preprocess.py             # Audio preprocessing for TensorFlow model path
├── audio_transformer.py            # Audio transformer model definition
├── auto_transformer.py             # Duplicate audio transformer model file
├── backend/pipelines/
│   ├── image_pipeline.py           # HF image pipeline wrapper
│   └── audio_pipeline.py           # HF audio pipeline wrapper
└── src/
    ├── models/
    │   ├── hf_image_model.py       # Hugging Face image model loader
    │   └── hf_audio_model.py       # Hugging Face audio model loader
    └── utils/
        └── postprocess.py          # Label mapping helper
```

## How detection works

### 1. File upload endpoints

The frontend chooses one of these endpoints based on file extension:

- `POST /api/image/detect`
- `POST /api/voice/detect`

These routes:

- accept JSON with a `filename`
- generate a fake or real result using random values
- store the record in PostgreSQl
- return a response for the UI

They do not currently upload or analyze the actual media file contents.

### 2. URL analysis endpoint

`POST /api/url/analyze` is the most substantial path in the app.

Flow:

1. Validate the URL and reject private/local hosts
2. Fetch the remote resource with size limits
3. If the URL is HTML, parse it and try to find a linked media candidate
4. Detect whether the resolved file is an image or audio asset
5. Try to load optional local TensorFlow weights:
   - `models/image_xception.weights.h5` or `image_xception.weights.h5`
   - `models/audio_transformer.weights.h5` or `audio_transformer.weights.h5`
6. If weights are unavailable or inference fails, fall back to heuristic analysis
7. Store the result in `url_detections`

### 3. Heuristic fallback behavior

When trained weights are not available, the app uses heuristics:

- Image heuristics:
  - URL metadata keyword scoring
  - edge detail via Laplacian variance
  - saturation level
  - grayscale contrast

- Audio heuristics:
  - URL metadata keyword scoring
  - spectral flatness
  - RMS variation
  - short-duration penalty

## API overview

### Detection routes

- `POST /api/image/detect`
- `GET /api/image/history`
- `POST /api/voice/detect`
- `GET /api/voice/history`
- `POST /api/url/analyze`
- `GET /api/url/history`

### Analytics route

- `GET /api/analytics`

Returns:

- total image detections
- total voice detections
- fake vs real counts
- average confidence
- 7-day image and voice activity trends

## Database schema

The app creates three tables on startup:

- `image_detections`
- `voice_detections`
- `url_detections`

The database is initialized in `init_db()` and seeded with demo image/voice history in `seed_demo_data()` if the history tables are empty.

## Setup

### 1. Create a virtual environment

```bash
python -m venv .venv
source .venv/bin/activate
```

### 2. Install the base dependency

```bash
pip install flask
```

### 3. Install optional analysis dependencies

For the current URL-analysis path and model helpers, you will likely want:

```bash
pip install numpy pillow opencv-python librosa tensorflow torch transformers
```

If you only want to launch the demo UI and simulated file detection routes, Flask is the only strict dependency visible from `app.py`. The optional libraries are used when the URL-analysis route performs media decoding, feature extraction, or model loading.

## Run the app

```bash
python app.py
```

Or:

```bash
python .app.py
```

The server starts on:

```text
http://localhost:9000
```

## Frontend usage

Open the app in a browser and use one of the tabs:

- `Upload Media`
  - choose an image or audio file
  - current behavior is demo-only and uses filename-based simulated results

- `Analyze URL`
  - paste a public `http://` or `https://` URL
  - the backend fetches and analyzes image or audio content from that URL

## Model-related code in the repo

The repository contains model code that is only partially wired into the live app:

- TensorFlow image model:
  - [image_xception.py](/home/prashant/Downloads/deep-defender/image_xception.py)
- TensorFlow audio model:
  - [audio_transformer.py](/home/prashant/Downloads/deep-defender/audio_transformer.py)
- Hugging Face image/audio wrappers:
  - [src/models/hf_image_model.py](/home/prashant/Downloads/deep-defender/src/models/hf_image_model.py)
  - [src/models/hf_audio_model.py](/home/prashant/Downloads/deep-defender/src/models/hf_audio_model.py)
- Pipeline wrappers:
  - [backend/pipelines/image_pipeline.py](/home/prashant/Downloads/deep-defender/backend/pipelines/image_pipeline.py)
  - [backend/pipelines/audio_pipeline.py](/home/prashant/Downloads/deep-defender/backend/pipelines/audio_pipeline.py)

Important note:

- The Flask routes currently use the TensorFlow weight-loading path only for URL analysis
- The Hugging Face pipeline classes are present but not currently called from `app.py`

## Security and safety measures already present

- Blocks localhost and private/reserved IP ranges for URL analysis
- Limits HTML fetch size to 1 MB
- Limits media download size to 15 MB
- Restricts served frontend files to `index.html`, `script.js`, and `style.css`

## Known limitations

- File upload analysis is simulated and does not inspect actual uploaded bytes
- There is no `requirements.txt` or `pyproject.toml` yet
- The TensorFlow model path depends on local `.weights.h5` files that are not included here
- Hugging Face model wrappers are present but not integrated into the running Flask app
- `auto_transformer.py` appears to duplicate `audio_transformer.py`
- `detections.db` is committed into the repo and may contain mutable local demo data
- The UI focuses on a single detection result and does not surface history or analytics yet

## Good next steps

- Replace simulated file detection with real multipart upload handling
- Add a proper dependency manifest
- Wire the Hugging Face pipelines into the Flask routes
- Add tests for URL validation, media fetching, and analytics responses
- Split the Flask backend and frontend assets into clearer modules
- Add environment-based config for port, debug mode, and model locations

## Demo-friendly summary

Deep Defender is a strong prototype for a deepfake-detection dashboard. The URL-analysis pipeline contains the most meaningful backend logic today, while the upload flow is still a demo stub. If you want to evolve this into a more complete system, the existing repo already gives you a usable Flask shell, SQLite history, frontend scaffolding, and multiple starting points for ML model integration.
