# Local Docker Setup Guide

This is a step-by-step guide to running the GeoShortMaker pipeline inside Docker on your local machine. After following this guide, you'll be able to generate videos without installing Python, Node.js, FFmpeg, or Playwright directly on your system.

> **Prerequisites**: You need [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed. That's the only thing you need to install manually. Verify it's working by running `docker --version` in your terminal.

---

## Step 1: Create the Dockerfile

Create a file called `Dockerfile` in the root of the project (same folder as `geoshortmaker.py`):

```dockerfile
# Use Microsoft's official Playwright image as the base.
# This gives us Python 3.11 + Chromium + all browser dependencies pre-installed.
FROM mcr.microsoft.com/playwright/python:v1.44.0-jammy

# Install system dependencies:
# - ffmpeg: video encoding/decoding
# - nodejs + npm: for the Cesium 3D renderer
# - curl: used during the Node.js install
RUN apt-get update && apt-get install -y \
    ffmpeg \
    nodejs \
    npm \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Set the working directory inside the container
WORKDIR /app

# Install Python dependencies first (cached layer — only rebuilds if requirements.txt changes)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Install Node.js dependencies (Playwright for Node)
COPY package.json .
RUN npm install

# Install the Playwright browser binaries for Node (Chromium)
RUN npx playwright install chromium --with-deps

# Copy the rest of the project files in
COPY . .
```

---

## Step 2: Create the docker-compose.yml

Create a file called `docker-compose.yml` in the root of the project:

```yaml
services:
  pipeline:
    build: .
    # Mount your local directory into the container.
    # This means edits you make on your machine are instantly reflected
    # inside the container — no rebuild needed.
    volumes:
      - .:/app
    # Load your .env file and pass all variables into the container.
    # Your API keys stay on your machine and never get baked into the image.
    env_file:
      - .env
    # Keep the container running so you can exec into it
    stdin_open: true
    tty: true
```

---

## Step 3: Build the Image

Run this once (and again any time `requirements.txt` or `package.json` changes):

```bash
docker compose build
```

This will take a few minutes the first time — it's downloading and installing everything. After the first build, subsequent builds are fast because Docker caches each layer.

---

## Step 4: Run the Pipeline

To generate a video:

```bash
docker compose run --rm pipeline python geoshortmaker.py --prompt "Why is Bangladesh so densely populated"
```

- `--rm` means the container is deleted after it finishes (keeps things clean)
- `python geoshortmaker.py` is the command that runs inside the container
- Your `runs/` and `output/` folders will appear in your normal project directory because of the volume mount in Step 2

To use a custom voice:

```bash
docker compose run --rm pipeline python geoshortmaker.py --prompt "your prompt" --voice voices/guy_michaels.mp3
```

---

## Step 5: Drop Into an Interactive Shell (Optional)

If you want to poke around inside the container, debug something, or run commands manually:

```bash
docker compose run --rm pipeline bash
```

You'll get a shell inside the container where you can run any command as if you were on a Linux machine with everything installed.

---

## How the Volume Mount Works

```
Your Machine                    Inside the Container
─────────────────               ────────────────────
geo-short-maker/          ←──►  /app/
  geoshortmaker.py               geoshortmaker.py
  .env                           .env (via env_file, not mount)
  runs/                          runs/
  output/                        output/
  voices/                        voices/
```

Files written by the pipeline (like `runs/<slug>/final_short.mp4`) appear on your normal filesystem automatically. You don't need to copy anything out of the container.

---

## Common Issues

**"docker: command not found"**
Docker Desktop isn't installed or isn't running. Start Docker Desktop first.

**"Error: GEMINI_API_KEY is missing"**
Your `.env` file either doesn't exist or is missing the key. Copy `.env.example` to `.env` and fill in your keys.

**Build fails on `playwright install`**
Make sure you're using the `mcr.microsoft.com/playwright/python` base image — it has the system dependencies Playwright needs pre-installed.

**Changes to Python files don't seem to take effect**
They should — the volume mount handles this. If it's not working, double-check the `volumes:` section in `docker-compose.yml` uses `.:/app` (current directory mapped to `/app`).

---

## Cleaning Up

Remove the built image when you no longer need it:

```bash
docker compose down --rmi all
```

Remove intermediate cache and stopped containers:

```bash
docker system prune
```
