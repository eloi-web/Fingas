# FingaScan

> Generative fingerprint art — scan your fingertip through your camera and watch your unique ridge patterns emerge in real time.

FingaScan uses your device's rear camera combined with Sobel edge detection, multi-frame accumulation, and seeded procedural generation to build a rich, artistic rendering of your fingertip. This is **not** biometric security — it's a fun, generative art piece. Every scan is unique and shareable.

---

## Features

- **Live edge detection** — full 3×3 Sobel operator runs on every camera frame in real time
- **Multi-frame accumulation** — 45 consecutive frames screen-blended at `α=0.10`, so consistent edges reinforce while noise averages out
- **Seeded procedural ridges** — arch/loop/whorl pattern generated from actual camera pixel color samples, making every scan deterministically unique
- **Composite output** — final image layers darkened video + Sobel accumulation + procedural ridges for a tactile, cinematic look
- **Auto-scan** — scanning begins the instant the camera is ready, no extra button press needed
- **Save to device** — download the result as a PNG with one tap

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | React 19 + TypeScript |
| Bundler | Vite 6 |
| Styling | Tailwind CSS v4 |
| Animation | Motion (Framer Motion) |
| Image processing | Canvas API + Sobel edge detection + seeded LCG PRNG |
| Icons | Lucide React |
| Font | Dynamix (local) + Space Grotesk + Plus Jakarta Sans |

No TensorFlow.js dependency — all processing is a pure Canvas API pixel loop, keeping the bundle small and the processing fast.

---

## How It Works

```
Camera feed (getUserMedia)
        │
        ▼
  startRealtimeProcessing() [rAF loop]
        │
        ├─ camera_ready ──► 3×3 Sobel ──► processingCanvasRef (live preview)
        │
        └─ scanning ──────► 3×3 Sobel ──► processingCanvasRef (live preview)
                                       └─► accumCanvasRef (screen-blend, α=0.10, ×45 frames)
                                                   │
                                                   ▼
                                         sample raw pixel colors for seed
                                                   │
                                         ┌─────────┴──────────┐
                                         ▼                    ▼
                                   Sobel layer        procedural ridges
                                   (55% opacity)    (seeded arch/loop/whorl)
                                         └─────────┬──────────┘
                                                   ▼
                                         video bg + composite ──► 512×512 PNG
```

**Edge detection** — a full 3×3 Sobel kernel computes `√(Gx²+Gy²)` magnitude per pixel, amplified ×6 to catch low-contrast skin micro-texture. Edge pixels are mapped to gold tones.

**Procedural generation** — a seeded LCG PRNG (`makePrng`) is initialized from pixel color samples taken at the end of the scan. It draws 24–34 organic ridges in an arch, loop, or whorl pattern. Because the seed comes from actual camera data, each scan produces a different but reproducible result.

---

## Getting Started

### Prerequisites

- Node.js 18+
- A device with a camera (works best on mobile with a rear camera)

### Install & Run

```bash
npm install
npm run dev
```

Open `http://localhost:3000` in your browser (or the LAN address on mobile).

### Build

```bash
npm run build
npm run preview
```

---

## Usage

1. Tap **Start Sequence** — the app requests camera permission
2. Hold your fingertip close to the lens — scanning starts automatically
3. Hold still for ~1.5 seconds while 45 frames accumulate
4. Your fingerprint art appears — tap **Save** to download it as a PNG

---

## Challenges & Lessons Learned

Building this looked simple on paper but hit several hard walls.

### 1. Cameras cannot resolve fingerprint ridges

The single biggest constraint: fingerprint ridges are 400–500 µm wide. A phone camera lens is not a macro lens, and even at minimum focus distance, skin micro-texture is below the optical resolution limit. Every attempt at pure camera-based ridge detection produced a blank or noise-only output.

**Solution:** Abandon pure camera detection. Instead, use the Sobel layer to capture whatever macro-texture exists (fingertip boundary, light gradients, skin tone variation), then overlay procedural ridges seeded by actual pixel color samples from the camera feed. The result looks like a fingerprint and is unique per scan without pretending the camera can see what it cannot.

### 2. Single-frame capture is useless for texture

The original approach called `video.pause()`, took one frame, and displayed it. A single frame is dominated by noise, lighting artifacts, and motion blur — there was no usable edge signal.

**Solution:** Replace the snapshot with a 45-frame accumulation loop. Each frame is Sobel-processed and screen-blended into an off-screen canvas at `globalAlpha = 0.10`. Over ~1.5 seconds, real edges (which appear in the same place frame after frame) accumulate, while random noise cancels out.

### 3. The blur overlay killed the camera feed

A decorative `glass-panel` div styled with `backdrop-filter: blur(20px)` was layered at `z-10` directly over the video element. It made the entire camera viewport look blurry — not as an artistic choice, but because the blur filter was composited on top of the live feed. This was invisible in code review because the div had no background color.

**Solution:** Remove the glass-panel from the scanner viewport entirely. The corner reticles and grid decorations (which don't use backdrop-filter) remain.

### 4. The quality gate rejected every finger

An early quality gate sampled the *Sobel output* canvas to check if enough edges were present before allowing a scan to proceed. The problem: after Sobel with no visible ridges, the output is near-black, so the pixel average was always below the threshold. Every finger was rejected as "not enough texture."

**Solution:** Remove the gate. The Sobel layer will simply be dim if there is no macro-texture — the procedural layer still produces a meaningful result regardless.

### 5. Simplified edge detection produced nothing

The first Sobel implementation used only a 2-sample gradient (`pixel[x] - pixel[x+1]`), which is effectively a 1D first derivative without the full 3×3 neighbourhood. On low-contrast skin, this produced near-zero values everywhere.

**Solution:** Replace with a proper 3×3 Sobel kernel — `Gx` and `Gy` computed separately, combined as `√(Gx²+Gy²)`, then amplified ×6. This is sensitive enough to detect tone transitions that exist even on smooth skin.

### 6. Git push blocked on HTTPS port 443

The network firewall blocked outbound port 443 to `github.com`, so `git push` over HTTPS failed silently with a timeout.

**Solution:** Switch the remote to SSH (`git remote set-url origin git@github.com:...`). Generate an `ed25519` key, add the public key to the GitHub account, and push over SSH.

### 7. Git fetch/pull timing out on SSH port 22

After fixing push, `git fetch` timed out because port 22 was also blocked (or asymmetrically filtered — push worked but pull did not, likely due to stateful firewall rules).

**Solution:** Configure `~/.ssh/config` to route all `github.com` SSH connections through `ssh.github.com` on port 443, which GitHub provides specifically as a fallback:

```
Host github.com
  Hostname ssh.github.com
  Port 443
  User git
  IdentityFile ~/.ssh/id_ed25519
```

---

## Disclaimer

FingaScan does not store, transmit, or process biometric data. The camera feed is processed entirely on-device in the browser. No images ever leave your device.

---

## Project Status

Active development. Core scanning pipeline is functional. Planned improvements: shareable link generation, pattern color themes, WebGL-accelerated accumulation.
