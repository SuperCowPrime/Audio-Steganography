
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
├── stego_web_app.zip            # Flask web demo (unzip to run)
└── powerpoint                   # Project presentation slides
```

---

## How It Works

### Step 1 — Chaotic Encryption (3 layers)

Before embedding, the payload is encrypted using three chaotic maps applied in sequence:

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

## Quick Start

### Requirements

```bash
pip install numpy pywt Pillow
```

### Hide a text message

```python
import soundfile as sf
from audio_steganography import embed_addition, decode_addition

# Load cover audio
audio, sr = sf.read("cover.wav")

# Embed
stego = embed_addition(audio, secret="Hello, hidden world!")
sf.write("stego.wav", stego, sr)

# Decode (needs original)
original, _ = sf.read("cover.wav")
message = decode_addition(stego, original)
print(message)  # "Hello, hidden world!"
```

### Hide an image

```python
import numpy as np
from PIL import Image

img = np.array(Image.open("secret.png").convert("L"))  # grayscale
stego = embed_addition(audio, secret=img)

recovered_img = decode_addition(stego, original)
Image.fromarray(recovered_img.astype(np.uint8)).save("recovered.png")
```

### Override method (no original needed to decode)

```python
from audio_steganography import embed_override, decode_override

stego = embed_override(audio, secret="Blind decode message")
message = decode_override(stego)  # no original required
```

---

## Web App

A Flask-based web interface is included in `stego_web_app.zip`.

```bash
unzip stego_web_app.zip
cd stego_web_app
pip install flask numpy pywt Pillow
python app.py
```

Then open `http://localhost:5000` to upload audio, embed a secret, and download the stego file.

---

## API Reference

### `embed_addition(cover_audio, secret, alpha, wavelet, dwt_level, encrypt, henon_params, arnold_iterations)`
Embeds payload by adding to DWT coefficients. Returns stego audio array.

### `decode_addition(stego_audio, original_audio, wavelet, dwt_level, decrypt, henon_params, arnold_iterations)`
Decodes payload by comparing stego and original DWT bands. Returns `str` or `np.ndarray`.

### `embed_override(cover_audio, secret, ...)`
Embeds payload by replacing DWT coefficients. Returns stego audio array.

### `decode_override(stego_audio, ...)`
Decodes payload from stego audio alone — no original needed. Returns `str` or `np.ndarray`.

### Quality metrics

```python
from audio_steganography import compute_audio_snr, compute_psnr, compute_mse, text_match_score

snr   = compute_audio_snr(original, stego)      # dB — higher is better
psnr  = compute_psnr(orig_img, recovered_img)   # dB — higher is better
mse   = compute_mse(orig_img, recovered_img)    # lower is better
score = text_match_score(original_text, recovered_text)  # 0–100%
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

Academic project — HIT, 2026. For educational use.
