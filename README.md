# Apollo-IDR

A fork of [Apollo](https://github.com/ClassicOldSong/Apollo) that makes the NVENC encoder emit **periodic keyframes**, so decode glitches no longer stay on screen forever.

If you streamed a desktop or a game over Moonlight/Artemis and watched blocky squares appear in static areas — a wallpaper, a taskbar, a HUD, a game menu — and pile up over time until you reconnected, this fork is for you.

> This is not a general-purpose replacement for Apollo. It is upstream Apollo master with one behaviour changed, plus the compilation fixes needed to build it with a current toolchain. Read [What changed](#what-changed), then decide.

## The bug

Upstream Apollo (and Sunshine before it) configures NVENC with an **infinite GOP**:

```cpp
enc_config.gopLength = NVENC_INFINITE_GOPLENGTH;
format_config.idrPeriod = NVENC_INFINITE_GOPLENGTH;
```

The encoder sends exactly one keyframe, at the start of the session, and never another. A keyframe (IDR) is the only frame that is fully self-contained — everything after it is a delta against the previous picture. So when the decoder produces a bad block once — a lost packet, a driver hiccup, an out-of-order frame — that error becomes part of the reference picture, and every following frame inherits it.

Nothing ever repairs it:

- the client only asks for a new keyframe after it detects packet loss (`moonlight-common-c`), and on a wired LAN there is no packet loss;
- `nvenc_intra_refresh` in `sunshine.conf` is a **no-op for Moonlight clients** — the server only enables intra refresh when the client sends the RTSP argument `x-ss-video[0].intraRefresh=1`, and both Moonlight and Artemis leave it at the default `0`;
- raising the bitrate changes nothing, because the damage is not caused by compression;
- changing the codec (H.264 vs HEVC) or the client app changes nothing either.

The result is what people describe as "the blocks appear and never go away", and it is worse in static scenes because nothing in the picture changes to mask the damage.

## What this fork does

Puts a real keyframe interval back:

| Setting | Default | What it does |
| --- | --- | --- |
| `nvenc_keyframe_interval` | `2` | Seconds between keyframes. Fractions allowed (`0.5`). `0` restores upstream behaviour (infinite GOP). |
| `nvenc_rolling_refresh` | `disabled` | Rolling intra refresh instead of hard keyframes — smoother, repairs slower. **Known to give a black screen (audio only) on Realtek/SCBC TV decoders.** |
| `nvenc_temporal_aq` | `disabled` | NVENC temporal AQ: shift bits towards flat, low-motion areas where banding and blockiness show first. |

All three are plain `sunshine.conf` keys and appear in the web UI under **Configuration → Audio/Video → NVIDIA NVENC Encoder**.

Measured cost of keyframes at 720p60, CBR 30 Mbps, static-ish content: **+0.8 % bitrate at 2 s**, roughly +3–4 % at 0.5 s. No measurable latency, because the keyframe is encoded in the same frame slot and the VBV buffer absorbs the burst.

Two environment variables still work as build-time overrides: `APOLLO_IDR_SECONDS` and `APOLLO_INTRA_REFRESH`, plus `APOLLO_NVENC_TUNING` (`quality` / `low` / unset) to switch the NVENC tuning info away from `ULTRA_LOW_LATENCY`.

### Diagnostics it adds to the log

```
NvEnc: keyframe interval (idr/gop) = 120 frames
NvEnc: refresh mode = idr
NvEnc: intra refresh period = 60 frames     (only with rolling refresh)
NvEnc: temporal AQ enabled                  (only with temporal AQ)
```

## What changed

Against `ClassicOldSong/Apollo` master (`adc5c5a`):

- `src/nvenc/nvenc_base.cpp` — periodic `gopLength` / `idrPeriod` from `nvenc_keyframe_interval`, optional rolling intra refresh, optional temporal AQ, log lines above.
- `src/nvenc/nvenc_config.h` — three new config fields.
- `src/config.cpp` — parse `nvenc_keyframe_interval`, `nvenc_rolling_refresh`, `nvenc_temporal_aq`.
- `src_assets/common/assets/web/` — the three controls, their strings, and their defaults (`configs/tabs/encoders/NvidiaNvencEncoder.vue`, `config.html`, `public/assets/locale/en.json`).
- `docs/configuration.md` — documentation for the new keys.

Plus three fixes that are **not** mine and **not** related to the change, but without which upstream master does not build on a current toolchain:

1. **OpenSSL 4.x** — `ASN1_BIT_STRING` became opaque and `X509_get_subject_name()` returns `const`. Fixed in `src/crypto.cpp` with `ASN1_STRING_get0_data()` / `ASN1_STRING_length()` and a `const_cast`.
2. **`src/video.cpp`** — master references `cfg.profile`, which no longer exists in `video::config_t`, so the file does not compile at all. The AMD AMF option table now returns `"high"`.
3. **Boost + CMake 4.x** — Boost's own `CMakeLists.txt` calls `source_group()` with a file outside the tree, which CMake 4 rejects. This one is applied to the downloaded Boost source inside `build/_deps`, not to a tracked file; see [Building](#building).

## Building (Windows)

Requirements: MSYS2 with the UCRT64 toolchain, CMake ≥ 3.20, Ninja, and the **official** Node.js build (not the MSYS2 one) visible on `PATH` as `npm`.

```powershell
# 1. MSYS2 once
#    install msys2, then from a UCRT64 shell:
pacman -Syu
pacman -S --needed base-devel git mingw-w64-ucrt-x86_64-toolchain mingw-w64-ucrt-x86_64-cmake \
  mingw-w64-ucrt-x86_64-ninja mingw-w64-ucrt-x86_64-boost mingw-w64-ucrt-x86_64-cppwinrt \
  mingw-w64-ucrt-x86_64-curl-winssl mingw-w64-ucrt-x86_64-openssl mingw-w64-ucrt-x86_64-opus \
  mingw-w64-ucrt-x86_64-miniupnpc mingw-w64-ucrt-x86_64-onevpl mingw-w64-ucrt-x86_64-nlohmann-json \
  mingw-w64-ucrt-x86_64-nsis

# 2. clone
git clone --recurse-submodules --shallow-submodules https://github.com/atorixa/Apollo-IDR.git
cd Apollo-IDR

# 3. configure — this is the step where Boost's source_group() breaks on CMake 4
cmake -B build -G Ninja -S . -DCMAKE_BUILD_TYPE=Release
```

If configure dies with `source_group ... is not a prefix of ...`, comment out every `source_group` line in the downloaded Boost tree and configure again:

```bash
grep -rl '^\s*source_group' build/_deps/boost-src/libs/*/CMakeLists.txt \
  | xargs sed -i 's/^\(\s*source_group\)/# disabled (cmake4): \1/'
cmake -B build -G Ninja -S . -DCMAKE_BUILD_TYPE=Release
```

```bash
ninja -C build          # produces build/sunshine.exe (~55 MB, fully static)
cpack -G ZIP            # optional: portable archive
```

The web UI is a separate vite build that runs as part of the cmake build; its output lands in `build/assets/web/`.

## Installing

The Windows build is a **drop-in replacement for `sunshine.exe`**, but the web UI is served **from disk**, not from the binary. If you only copy the exe you keep the old settings page and the new fields will be missing.

1. Stop the `ApolloService` service.
2. Back up `C:\Program Files\Apollo\sunshine.exe` and `C:\Program Files\Apollo\assets\web\`.
3. Copy the new `sunshine.exe` into `C:\Program Files\Apollo\`.
4. Copy `assets/web/` from the build over `C:\Program Files\Apollo\assets\web\`.
5. Start `ApolloService`, then hard-reload the web UI (`Ctrl+Shift+R`) — the browser caches the old javascript bundle.

Release assets on this repo contain a ready `sunshine.exe` plus the matching `assets/web/`.

## Notes and caveats

- **No update mechanism, no telemetry.** Upstream Apollo has no self-updater and this fork does not add one; nothing here phones home and nothing will overwrite your installation. Updates are manual: pull, build, copy. The version string comes from the git tag, so a build from this repo can never be mistaken for an upstream release.
- **The client still matters.** A client that requests a keyframe after packet loss is still the primary repair mechanism; this fork only guarantees a ceiling on how long damage can survive.
- **Rolling refresh** (`nvenc_rolling_refresh`) reproduces a black screen with audio on the Realtek/SCBC decoder found in cheap Android TV boxes (tested on a Thomson ATV R1 / rtd284o). It is documented, but leave it off there.
- Upstream documentation, virtual display, HDR, clipboard sync, input modes and everything else are unchanged — see [ClassicOldSong/Apollo](https://github.com/ClassicOldSong/Apollo) and [LizardByte's Sunshine docs](https://docs.lizardbyte.dev/projects/sunshine).

## License

GPL-3.0, same as upstream. Original work © the Apollo and Sunshine contributors; the keyframe-interval patch and the build fixes are released under the same license.
