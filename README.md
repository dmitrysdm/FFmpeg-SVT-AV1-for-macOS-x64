# FFmpeg for macOS Intel with SVT-AV1

A GitHub Actions workflow that builds **FFmpeg 9.0.1** for **macOS Intel (x86_64)**
with the **SVT-AV1 4.2.0** video encoder and the **Opus** audio codec, both
linked **statically** into the resulting binaries.

The package contains three binaries — `ffmpeg`, `ffprobe`, and `ffplay` —
with no third-party codecs or libraries beyond SVT-AV1 and Opus.

## What the build produces

A single tarball: `FFmpeg-9.0.1-SVT-AV1-4.2.0-for-macOS-x64.tar.gz`

```
FFmpeg-9.0.1-SVT-AV1-4.2.0-for-macOS-x64/
├── bin/
│   ├── ffmpeg          # Mach-O 64-bit executable x86_64
│   ├── ffprobe
│   └── ffplay
├── lib/
│   └── libSDL2-*.dylib # bundled for ffplay only (install name rewritten to @loader_path)
├── licenses/           # FFmpeg (LGPL/GPL), SVT-AV1, Opus, SDL2
└── README.txt
```

- **SVT-AV1 and Opus are static** — no `libSvtAv1Enc`/`libopus` dylib
  dependencies (verified with `otool -L` during the build).
- **ffplay needs only the bundled SDL2** dynamic library next to the binaries.
- Everything else (AV1/VP9/Opus decoders, MP4/MKV/WebM/OGG muxers & demuxers,
  HTTP(S) protocols, etc.) is built into the binaries.

## Components and versions

| Component | Version | Linkage |
|-----------|---------|---------|
| FFmpeg | 9.0.1 (commit `bf1b838f2ab88b4f8fd83443325c782ea0e0f7fa`) | static libs |
| SVT-AV1 | 4.2.0 | static (`libSvtAv1Enc.a`) |
| Opus | 1.5.2 | static (`libopus.a`) |
| SDL2 | Homebrew (sdl2-compat) | dynamic, bundled |

All versions are pinned in the workflow's `env:` block
(`.github/workflows/build-ffmpeg-intel-mac.yml`).

## FFmpeg configure summary

- `--disable-shared --enable-static`
- Programs: `ffmpeg`, `ffprobe`, `ffplay`
- No external codecs autodetected (`--disable-autodetect`); the only external
  libraries enabled are `libsvtav1`, `libopus`, and `sdl2`
- Protocols: `file`, `pipe`, `http`, `https`
- Demuxers: `mov`, `matroska`, `ogg`
- Muxers: `mp4`, `matroska`, `ogg`, `webm`
- Filters: `scale`, `fps`, `setsar`, `dynaudnorm`
- `--arch=x86_64 --target-os=darwin`, compiled with `clang`/`clang++`
- `--enable-gpl --enable-version3` (required by libsvtav1)

Note: `--enable-postproc` is intentionally **not** used — `libpostproc` was
removed upstream in FFmpeg 8.0, so the option does not exist in 9.0.x.

## How the build works

Runner: `macos-15-intel` (Intel x86_64). The job:

1. Installs build tools via Homebrew: `cmake`, `nasm`, `pkg-config`,
   `autoconf`, `automake`, `libtool`, `sdl2`.
2. Builds **SVT-AV1** from the `v4.2.0` tag with CMake
   (`-DBUILD_SHARED_LIBS=OFF -DBUILD_APPS=OFF`, `x86_64`) and installs it to
   a private prefix. Verifies that only a static `libSvtAv1Enc.a` was produced.
3. Builds **Opus 1.5.2** with autotools
   (`--disable-shared --enable-static`) into the same prefix. Verifies the
   static `libopus.a`.
4. Downloads the exact FFmpeg commit, runs `./configure` (see summary above)
   and `make`.
5. Verifies the binaries:
   - exist and are `Mach-O 64-bit executable x86_64`;
   - expose `libsvtav1` and `libopus` encoders;
   - have **no** dynamic `libSvtAv1`/`libopus` dependency (`otool -L`);
   - `ffplay` works (`-version`, `-h full`).
6. Runs functional tests: encodes a short test pattern with `libsvtav1`
   (10-bit), encodes a sine tone with `libopus`, and encodes a combined
   AV1+Opus Matroska clip; checks the results with `ffprobe`.
7. Copies the SDL2 dylib used by `ffplay` into the package and rewrites its
   install name to `@loader_path/../lib/...` so the package is self-contained.
8. Collects licenses, writes `README.txt`, packs the tarball, and uploads it
   as a GitHub Actions artifact (30-day retention).

## How to run the build

1. Open the repository → **Actions** tab.
2. Select **"Build FFmpeg for macOS Intel"** → **Run workflow**.
3. When the run finishes (roughly 30–60 minutes), download the
   `FFmpeg-9.0.1-SVT-AV1-4.2.0-for-macOS-x64` artifact.

The workflow is manual-only (`workflow_dispatch`); nothing runs on push/PR.

## Using the package

```bash
tar -xzf FFmpeg-9.0.1-SVT-AV1-4.2.0-for-macOS-x64.tar.gz
cd FFmpeg-9.0.1-SVT-AV1-4.2.0-for-macOS-x64
./bin/ffmpeg -version
```

Example: encode to AV1 + Opus in a Matroska container:

```bash
./bin/ffmpeg -i input.mp4 -c:v libsvtav1 -preset 6 -crf 30 \
  -c:a libopus -b:a 128k output.mkv
```

Requires macOS on an Intel (x86_64) Mac. `ffplay` additionally requires the
bundled `lib/libSDL2-*.dylib` (already in the package).

## Repository layout

```
.github/workflows/build-ffmpeg-intel-mac.yml   # the entire build (single job)
README.md
```

There is no source code in this repository — the workflow is the project.
