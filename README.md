
# 🎵 Audio Steganography — DWT + Chaotic Encryption

> Hide text and images inside audio files — imperceptibly.  
> Final project for the **Speech & Audio Processing** course at **Holon Institute of Technology (HIT)**.

**Authors:** Svetlana Gavris · Yaniv Hananis · Amit Wagensberg

---

## What Is This?

This project implements an **audio steganography system** that conceals secret messages (text or grayscale images) inside ordinary WAV audio files. The hidden payload is encrypted with three chaotic maps before embedding, and the resulting audio is virtually indistinguishable from the original.

Unlike cryptography (which scrambles content), steganography hides the very *existence* of the message. This system does both: encrypt first, then hide.

---

## Repository Structure

```
Fin-Project-Audio/
├── fin project/
│   └── audio_steganography.py   # Core steganography library
├── stego_web_app.zip            # Flask web app (source, requires Python)
└── powerpoint                   # Project presentation slides
```

The web app folder (inside `stego_web_app.zip`) looks like this:

```
stego_web_app/
├── main.py                      # App launcher — starts server & opens browser
├── audio_steganography.py       # Core steganography library
├── app.py                       # Flask routes
├── StegoSound.spec              # PyInstaller config (for building .exe)
├── build.bat                    # Windows build script → produces StegoSound.exe
├── build.sh                     # macOS/Linux build script
├── requirements.txt             # Python dependencies
└── templates/
    └── index.html               # Web UI
```

---

## Running the App — 3 Options

### Option 1 — Double-click `.exe` (Windows, no Python needed) ⭐ Easiest

> Use this if you just want to run the app without installing anything.

1. Download **`StegoSound.exe`** from the [Releases](../../releases) page
2. Double-click it
3. A browser tab opens at `http://localhost:5000` automatically
4. When done, close the console window to stop the server

Output files are saved in an `outputs/` folder next to the `.exe`.

---

### Option 2 — Build the `.exe` yourself (requires Python once)

> Use this if you want to create the `.exe` from source.

**Prerequisites:** Python 3.9+ installed ([python.org](https://python.org))

```bash
# 1. Clone the repo and unzip the web app
git clone https://github.com/SuperCowPrime/Fin-Project-Audio.git
cd Fin-Project-Audio
unzip stego_web_app.zip
cd stego_web_app
```

**Windows:**
```
build.bat
```

**macOS / Linux:**
```bash
chmod +x build.sh
./build.sh
```

The build takes about 1–2 minutes. The result is:
- **Windows:** `dist/StegoSound.exe`
- **macOS/Linux:** `dist/StegoSound`

Double-click (or run) that file — no Python needed from that point on.

---

### Option 3 — Run directly with Python

> Use this if you're a developer and already have Python set up.

```bash
# 1. Clone and unzip
git clone https://github.com/SuperCowPrime/Fin-Project-Audio.git
cd Fin-Project-Audio
unzip stego_web_app.zip
cd stego_web_app

# 2. Install dependencies
pip install -r requirements.txt

# 3. Run
python main.py
```

The app will print a link and open `http://localhost:5000` in your browser automatically. Press `Ctrl+C` in the terminal to stop it.

---

## How to Use the Web App

### Encode (hide a secret)

1. **Upload a WAV file** as the audio carrier
2. **Choose what to hide** — a text message or a grayscale image
3. **Choose a protocol:**
   - **Addition** — better audio quality (SNR ~74 dB), but decoding requires the original audio file
   - **Override** — slightly lower quality (SNR ~18 dB), but decoding works without the original
4. Click **Generate Stego-Audio** — the file downloads automatically

### Decode (extract the secret)

1. **Select the same protocol** you used when encoding
2. **Upload the stego-audio file**
3. If you used **Addition**, also upload the original carrier audio
4. Click **Extract Hidden Data** — the text or image appears on screen

> ⚠️ The decryption keys (Hénon / Arnold settings) must match what was used during encoding. Default values work if you didn't change them.

---

## How It Works

### Step 1 — Chaotic Encryption (3 layers)

Before embedding, the payload is encrypted using three chaotic maps in sequence:

| Map | What it does |
|-----|-------------|
| **Hénon Map** | XORs byte values with a chaotic sequence |
| **Arnold Cat Map** | Scrambles element positions in a square grid |
| **Baker Map** | Rearranges vertical strips of the grid |

Decryption applies the inverse maps in reverse order.

### Step 2 — DWT Embedding

The encrypted bytes are embedded into the audio's **Discrete Wavelet Transform (DWT)** domain:

1. Audio is decomposed using `db4` wavelet at 3 levels
2. The **highest-frequency sub-band** (`coeffs[1]`) is used — least audible to humans
3. Each bit is encoded as a ±0.0005 perturbation of a DWT coefficient
4. IDWT reconstructs the stego audio

### Step 3 — Self-Describing Header

A **25-byte header** is embedded alongside the payload, storing everything needed to decode — no metadata needs to be shared separately:

```
mode (1B) | payload_len (4B) | grid_side (4B) | img_H (4B) | img_W (4B) | n_bits (4B) | coeff_scale (2B) | alpha (2B)
```

---

## Two Embedding Methods

| | Addition | Override |
|---|---|---|
| **How** | Adds ±0.0005 to existing coefficients | Replaces coefficients entirely |
| **Audio SNR (text)** | ~73.65 dB ✅ | ~18.17 dB ⚠️ |
| **Audio SNR (image)** | ~57.46 dB ✅ | ~10.85 dB ⚠️ |
| **Image MSE** | 0.00 (perfect) | 0.00 (perfect) |
| **Needs original audio to decode?** | Yes | **No (blind decode)** |

**Addition** is recommended when audio quality matters. **Override** is useful when the original audio is unavailable to the receiver.

---

## Python API

You can also use the library directly in your own Python scripts:

```python
import soundfile as sf
import numpy as np
from PIL import Image
from audio_steganography import (
    embed_addition, decode_addition,
    embed_override, decode_override,
)

audio, sr = sf.read("cover.wav")

# --- Hide text (Addition) ---
stego = embed_addition(audio, secret="Hello, hidden world!")
sf.write("stego.wav", stego, sr)

original, _ = sf.read("cover.wav")
print(decode_addition(stego, original))   # "Hello, hidden world!"

# --- Hide an image (Addition) ---
img = np.array(Image.open("secret.png").convert("L"))
stego = embed_addition(audio, secret=img)
recovered = decode_addition(stego, original)
Image.fromarray(recovered.astype(np.uint8)).save("recovered.png")

# --- Override (no original needed to decode) ---
stego = embed_override(audio, secret="Blind decode message")
print(decode_override(stego))
```

### Quality metrics

```python
from audio_steganography import compute_audio_snr, compute_psnr, compute_mse, text_match_score

snr   = compute_audio_snr(original, stego)       # dB — higher is better
psnr  = compute_psnr(orig_img, recovered_img)    # dB — higher is better
mse   = compute_mse(orig_img, recovered_img)     # lower is better
score = text_match_score(original_text, decoded)  # 0–100%
```

---

## Results Summary

| Method | Payload | Audio SNR | Recovery |
|--------|---------|-----------|---------|
| Addition | Text | 73.65 dB | 100% match |
| Addition | Image | 57.46 dB | MSE = 0.00 |
| Override | Text | 18.17 dB | 100% match |
| Override | Image | 10.85 dB | MSE = 0.00 |

---

## Background & Research

This project is based on:

> Nasr, M., Shahriari, H., & Ghasemzadeh, M. (2024). *Audio steganography using chaotic maps and discrete wavelet transform.* Journal of Information Security and Applications, 80, 103664.

**Key finding from our implementation:** STFT/ISTFT-based embedding is unsuitable for exact payload recovery — the synthesis step introduces irreversible distortion. DWT bit-to-bit encoding solves this completely.

---

## License

Academic project — HIT, 2025. For educational use.
