## PureCut Pro

This repo contains the **PureCut Pro** React frontend (Vite + TypeScript + Tailwind + shadcn-ui) and a **Python FastAPI backend** that powers AI image tools:

- Background Remover
- Image Enhancer
- Object Remover
- Image Upscaler


---

## Contributors

* **[Dhruv Gupta]** ([@dhruvvv07](https://github.com/dhruvvv07))
* **[Prateek Reddy]** ([@PrateekReddy116](https://github.com/PrateekReddy116))

---

---

## 1. Prerequisites

- **Node.js** (LTS) and **npm**
- **Python 3.11** (recommended – must be a version with NumPy wheels available)

On Windows, during Python install, select **“Add Python to PATH”**.

---

## 2. Backend (Python / FastAPI)

All backend code lives in `backend/`.

### 2.1. Create and activate a virtual environment

From the project root (Windows / PowerShell):

```powershell
cd "E:\PureCut Pro\purecut-suite"
py -3.11 -m venv .venv
.\.venv\Scripts\activate
```

On macOS / Linux:

```bash
cd "/path/to/purecut-suite"
python3.11 -m venv .venv
source .venv/bin/activate
```

Notes:

- The `.venv` directory is **already ignored** by `.gitignore`, so it will not be committed to Git.
- Each time you start a new terminal, **re‑activate** the venv before running backend commands:
  - Windows: `.\.venv\Scripts\activate`
  - macOS/Linux: `source .venv/bin/activate`
- To deactivate: run `deactivate` in the terminal.

### 2.2. Install backend dependencies

```powershell
cd backend
python -m pip install --upgrade pip
pip install -r requirements.txt
```

If `realesrgan` or its dependencies fail to install, the **Image Upscaler** endpoint will still work using a high-quality bicubic fallback.

**Protect PDF** requires the `cryptography` dependency (pulled in via `pypdf[crypto]` in `requirements.txt`). Without it, `/pdf/protect` returns **503** with an install hint.

### 2.3. (Optional) Real-ESRGAN weights for AI upscaling

If you want model-based upscaling instead of just bicubic resize:

1. Create a weights directory:
   ```powershell
   mkdir weights
   ```
2. Download `RealESRGAN_x2plus.pth` from the official Real-ESRGAN releases and place it at:
   ```text
   backend/weights/RealESRGAN_x2plus.pth
   ```

If weights or dependencies are missing, the backend automatically falls back to bicubic resize.

### 2.4. Run the backend

From `backend/` with the venv active:

```powershell
uvicorn app:app --host 0.0.0.0 --port 8000 --reload
```

The API will be available at `http://127.0.0.1:8000`.

### 2.5. Verify backend is running

- Health check: open `http://127.0.0.1:8000/health` → `{"status": "ok"}`  
- Interactive docs: open `http://127.0.0.1:8000/docs` and test endpoints:
  - `POST /background-remover`
  - `POST /image-enhancer`
  - `POST /object-remover`
  - `POST /image-upscaler`

**CORS** is configured for common dev origins: `http://localhost:8080`, `http://127.0.0.1:8080`, `http://localhost:5173`, `http://127.0.0.1:5173`.

---

## 3. Frontend (Vite + React)

All frontend code lives in `src/`.

### 3.1. Install dependencies

From the project root:

```bash
cd "E:\PureCut Pro\purecut-suite"
npm install
```

### 3.2. Run the frontend

```bash
npm run dev
```

By default, Vite runs on `http://127.0.0.1:5173` (or a similar port). The AI tools pages are under **AI Tools** in the navbar:

- Background Remover → `/ai-tools/background-remover`
- Image Enhancer → `/ai-tools/enhancer`
- Object Remover → `/ai-tools/object-remover`
- Image Upscaler → `/ai-tools/upscaler`

Make sure the **backend** is running before using these tools, otherwise you’ll see network errors in the UI.

### 3.3. Run frontend and API together

From the project root (after `npm install` and a working Python venv with backend deps):

```bash
npm run dev:all
```

This runs **Vite** and **`uvicorn`** concurrently (see `package.json`). Stop both with one Ctrl+C.

**Alternative (two terminals):**

- Windows: open PowerShell in `backend/` and run `python -m uvicorn app:app --host 0.0.0.0 --port 8000 --reload`, then from the repo root run `npm run dev`.
- Or run `.\scripts\dev-start.ps1` from the repo root (starts API in a second window, then Vite here).
- macOS/Linux: `bash scripts/dev-start.sh` starts the API in the background and Vite in the foreground.

For a longer-term **desktop installer / bundled Python** approach, see [PACKAGING.md](PACKAGING.md).

---

## 4. AI models and algorithms used

For the tools implemented in the Python backend:

- **Background Remover (`/background-remover`)**
  - Uses `rembg`, which wraps a **U²-Net**-family segmentation model to separate foreground from background.
  - Returns a PNG with a transparent background.

- **Image Enhancer (`/image-enhancer`)**
  - Uses lightweight, classical image-processing via **Pillow**:
    - Median denoise
    - Unsharp mask for detail enhancement
    - Adjustable **sharpness**, **contrast**, and **brightness** based on the UI slider.
  - No heavy ML model here – intentionally fast and lightweight.

- **Object Remover (`/object-remover`)**
  - Uses **OpenCV Telea inpainting** (`cv2.inpaint` with `INPAINT_TELEA`):
    - Input: original image + a **mask** (PNG; white = regions to remove). The UI builds this mask by painting over the image.
    - Fills in masked regions using surrounding pixels. Higher-quality inpainting (e.g. LaMa) would need extra models and GPU-friendly packaging.

- **Image Upscaler (`/image-upscaler`)**
  - Tries to use **Real-ESRGAN** (RRDBNet-based) if `realesrgan` is installed and model weights are available at `backend/weights/RealESRGAN_x2plus.pth`.
  - If Real-ESRGAN is unavailable or fails, falls back to **high-quality bicubic resize** using Pillow.

These choices balance **quality**, **performance**, and **ease of installation**, especially on Windows.

---

## 5. PDF tools (backend + frontend)

PureCut Pro also includes a full set of **server-backed PDF tools**, implemented in the FastAPI backend and wired to the React frontend.

### 5.1. Backend PDF endpoints

All PDF endpoints live in `backend/app.py` and are available under the same base URL as the image tools (e.g. `http://127.0.0.1:8000`):

- **Merge PDFs**
  - **Endpoint**: `POST /pdf/merge`
  - **Body**: `files` – multiple PDF files (`multipart/form-data`)
  - **Response**: single merged PDF (`merged.pdf`)

- **Split PDF**
  - **Endpoint**: `POST /pdf/split`
  - **Body**:
    - `file` – single PDF
    - `mode` – `"pages"` or `"range"`
    - `ranges` – optional ranges string like `"1-3,5,7-10"` (used when `mode="range"`)
  - **Response**: ZIP archive with the resulting PDFs (`split.zip`)

- **PDF → Images**
  - **Endpoint**: `POST /pdf/to-images`
  - **Body**:
    - `file` – single PDF
    - `format` – `"png"` or `"jpg"`
    - `quality` – `"high" | "medium" | "low"` (maps to ~300/150/72 DPI)
  - **Response**: ZIP archive with one image per page (`pages.zip`)

- **Images → PDF**
  - **Endpoint**: `POST /pdf/from-images`
  - **Body**:
    - `files` – multiple images (PNG/JPG/WebP)
    - `page_size` – `"a4" | "letter" | "fit"`
  - **Response**: multi-page PDF (`images.pdf`)

- **Compress PDF**
  - **Endpoint**: `POST /pdf/compress`
  - **Body**:
    - `file` – single PDF
    - `level` – `"low" | "medium" | "high"` (PyMuPDF only; selects garbage-collection strength)
    - `engine` – `"pymupdf" | "qpdf" | "ghostscript"` (default: `"pymupdf"`)
    - `preset` – `"screen" | "ebook" | "printer" | "prepress"` (Ghostscript only; default: `"ebook"`)
  - **Implementation**:
    - `engine=pymupdf`: **lossless** optimization with **PyMuPDF** (deflate streams, object streams, xref cleanup). Does **not** re-encode embedded images.
    - `engine=qpdf`: **fast, lossless** cleanup/deflate using the `qpdf` binary (must be installed separately).
    - `engine=ghostscript`: **aggressive** compression using Ghostscript presets (may reduce quality; must be installed separately).
  - **Response**: optimized PDF (`compressed.pdf`)

- **Protect PDF**
  - **Endpoint**: `POST /pdf/protect`
  - **Body**:
    - `file` – single PDF
    - `password` – required user password
    - `allow_print`, `allow_copy`, `allow_edit` – booleans for permissions
  - **Implementation**: uses **pypdf** with **AES-256** and **`pypdf[crypto]`** (installs `cryptography`). User and owner passwords are set to the same value you enter for maximum viewer compatibility.
  - **Response**: password-protected PDF (`protected.pdf`)

- **Split PDF thumbnails (preview)**
  - **Endpoint**: `POST /pdf/preview-pages`
  - **Body**: `file`, optional `max_pages` (default 24, max 50), `max_width` (default 132px)
  - **Response**: JSON `{ total_pages, previews: [{ page, image }] }` where `image` is a JPEG data URL for UI thumbnails.

- **Unlock PDF**
  - **Endpoint**: `POST /pdf/unlock`
  - **Body**:
    - `file` – password-protected PDF
    - `password` – correct password
  - **Response**: unlocked PDF (`unlocked.pdf`); returns **401** if the password is incorrect.

### 5.2. Frontend routes for PDF tools

The React frontend exposes each tool under `/pdf-tools/...`:

- Merge PDFs → `/pdf-tools/merge`
- Split PDF → `/pdf-tools/split`
- PDF to Images → `/pdf-tools/to-images`
- Images to PDF → `/pdf-tools/from-images`
- Compress PDF → `/pdf-tools/compress`
- Protect PDF → `/pdf-tools/protect`
- Unlock PDF → `/pdf-tools/unlock`
