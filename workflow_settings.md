# HR-Endless-Sampler — Recommended Node Settings for THE LATTICE

## HREndlessSampler

| Setting | Recommended | Rationale |
|---------|-------------|-----------|
| `fps` | **24** | H3 native frame rate; all scene timestamps use 24 fps |
| `chunk_frames` | **56** (1080p/16 GB) or **124** (24 GB+) | 56 = 17×3+5, fits 16 GB with KJNodes Low VRAM Attention=4. 124 = 17×7+5, needs 24 GB+. Larger chunks = fewer handoffs = fewer dialog-repetition risks |
| `video_continuation` | **22** | Default. 22 frames ≈ 0.92 s of overlap. Minimum is 5. Larger values increase VRAM and the window in which dialogue can be duplicated. **If dialog repetition persists, try 17 (one grid step) to reduce the overlap window** |
| `video_continuation_res` | **`0.30mp (736x416)`** | Reduces H3 reference-attention VRAM for the Video1 continuation, allowing larger `chunk_frames`. The full-resolution 5-frame boundary keyframe and Audio1 latent are unaffected |
| `cache_gemma_preproduction` | **`true`** | Caches Gemma's static preproduction context in system RAM (`/dev/shm`). Speeds up all subsequent chunk requests. Uses several GiB of RAM, zero VRAM |
| `gemma4_mtp` | **`true`** | Native 4-token draft-MTP speculative decoding. Faster Gemma inference. Disable only if the platform wheel lacks required symbols |
| `pytorch_memory_fraction` | **0.85** | Reserves 15% of VRAM outside PyTorch's allocator. Prevents driver-level OOM during large H3 temporaries. Set to 1.0 only if you need the full VRAM |
| `debug` | **`false`** for production, **`true`** for troubleshooting | Logs every chunk prompt and VRAM snapshots. Also runs a 3-step continuation preflight before multi-chunk renders |
| `debug_stop_chunk` | **0** | 0 = render the whole video. Set to N to stop after chunk N for testing |
| `debug_start_chunk` | **0** | 0 = auto-continue. Set to N to force-rerun from chunk N (reuses saved noise/frames) |

## MiniMaxH3ReferenceToVideo

| Setting | Recommended | Rationale |
|---------|-------------|-----------|
| `width` × `height` | **1344 × 768** (720p) or **1920 × 1088** (1080p) | Must be 32-pixel-aligned. 1080p needs the VRAM settings above |
| `frames` | **`max(5, round(duration × 24))` snapped to 5+17k** | Use the ComfyMathExpression: `max(5, round(a * 24)) + (5 - (max(5, round(a * 24)) % 17)) % 17` where `a` = duration in seconds. For 70 s → 1680 → 1680+5=1685 → 1685 mod 17 = 0 → 1685 frames |
| `ref_images` | **Only the images for subjects in the current scene** (1–9 max) | Per the updated prompts: room image first, then character images in `<Subject N>` order. Pure THE_LATTICE scenes (3, 4, 7, 10, 12, 23, 25) use **0** reference images |
| `prompt` | **Paste the full prompt block from the scene file** | The six-section Ref2VA prompt. The sampler extracts `detailed_description` for Gemma and passes the rest to H3 |

## BasicScheduler

| Setting | Recommended | Rationale |
|---------|-------------|-----------|
| `scheduler` | **`simple`** | H3's expected noise schedule |
| `steps` | **20** | H3 standard. More steps = higher quality but slower. 15 is a minimum |
| `denoise` | **1.0** | Full denoising (text-to-video). Use <1.0 only for img2vid |

## KSamplerSelect

| Setting | Recommended | Rationale |
|---------|-------------|-----------|
| `sampler_name` | **`res_multistep`** | H3's expected sampler |

## MiniMaxLowVRAMAttention (KJNodes)

| Setting | Recommended | Rationale |
|---------|-------------|-----------|
| `head_chunks` | **4** | Required for 1080p on 16 GB. Reduces VRAM peaks during reference-attention, especially critical in chunks 2+ which carry the 22-frame Video1 continuation |

## SpectrumApplyMiniMaxH3

| Setting | Recommended | Rationale |
|---------|-------------|-----------|
| `enabled` | **`true`** | Applies H3-specific spectral conditioning |
| `blend_weight` | **0.5** | Default |
| `degree` | **1** | Default |

## HREndlessSamplerSaveVideo

| Setting | Recommended | Rationale |
|---------|-------------|-----------|
| `fps` | **24** | Match the generation fps |
| `format` | **`video/h264-mp4`** | Native ComfyUI encoder, no VHS dependency. Supports audio muxing and embedded timeline metadata |
| `pixel_format` | **`auto`** | Let the encoder choose |
| `crf` | **19** | High quality. Lower = better but larger files |
| `filename_prefix` | **`THE_LATTICE_scene_[N]`** | One prefix per scene for easy comparison in the Matching Videos dropdown |

## HREndlessSamplerPreview

| Setting | Recommended | Rationale |
|---------|-------------|-----------|
| `tiny_vae` | **`taeh3.safetensors`** | More representative preview than the fast Latent2RGB. Costs more VRAM |
| `max_resolution` | **0** | Keep the latent preview resolution (no upscale) |
| `fps` | **24** | Match generation |

## Addressing the Specific Issues

### Gemma 4 S1.V1 / S1.V2 Validation Errors (Comment 1)

**Cause:** Gemma's preproduction planner identifies "visual beats" within each shot (e.g., `S1.V1` = first visual action in Shot 1). When a chunk's frame slice overlaps a beat, the chunk director must include an `evidence` string in its JSON response that is a **literal substring** of the `detailed_description` it wrote. If the shot description is too vague or the beat's action is split across ambiguous phrasing, the evidence check fails.

**Fix (prompt-side):** The rewritten prompts already use explicit, concrete shot descriptions with clear action verbs ("a monospace line arrives whole-line reading …", "the camera dollies geometrically forward through the wall"). This gives Gemma unambiguous beat actions to evidence. **No workflow setting change is needed.** If the error persists on a specific scene, enable `debug: true` and inspect `${TMPDIR}/comfyui-hr-endless-sampler/last_gemma_chunk_prompts.txt` to see which beat's evidence string failed to match, then make that shot's description more explicit.

### Off-Screen Voice Not Detected (Comment 3)

**Cause:** The voiceover in Scene 1 Shot 6 was the last content before the video ended. With no fade-out shot, the generator likely truncated the audio before the voiceover completed. The 22-frame `video_continuation` overlap also means the last ~0.9 s of audio from the previous chunk is carried forward, which can clip a voice that starts near the chunk boundary.

**Fix (prompt-side):** The added fade-out shot (Shot 7 at 01:16.000) gives the generator 8 additional seconds to complete the voiceover and fade the audio. The voiceover line is now well within the scene's duration rather than at the very end.

**Fix (workflow-side):** Ensure `chunk_frames` is large enough that the voiceover shot (Shot 6, starting at 01:08 = frame 1536) and the fade-out shot (Shot 7, starting at 01:16 = frame 1728) fall within the **same** chunk or at least that the voiceover completes before a chunk boundary. With `chunk_frames=124` (17×7+5), chunk boundaries are at frames 124, 248, 372, … The voiceover at frame 1536 falls in chunk 13 (frames 1513–1636). The fade-out at frame 1728 falls in chunk 15 (frames 1761–1884). This is fine — the voiceover completes within chunk 13.

### Dialog Repetition Across Clips (Comment 4a)

**Cause:** As detailed above, the Gemma 4 chunk director is **instructed** to include a dialogue line in every chunk whose planned interval overlaps the chunk's frame slice. Combined with the 22-frame audio continuation overlap, this causes the line to be generated (and heard) in two consecutive chunks.

**Mitigations (in order of preference):**

1. **Increase `chunk_frames`** to 124 (or higher if VRAM allows). Fewer chunks = fewer boundaries = fewer opportunities for a dialogue line to straddle a boundary.
2. **Reduce `video_continuation`** from 22 to **17** (one 17-frame grid step). This shrinks the audio overlap window from 0.92 s to 0.71 s, reducing the chance of audible duplication.
3. **Prompt-side:** Ensure dialogue lines are **not** positioned within the last 1 s of a shot. Each scene's dialogue is already well within its shot's duration (shots are 11–16 s long, dialogue is 2–5 s), so this is already satisfied.
4. **If repetition persists on a specific line:** Use `debug_start_chunk` to re-render from the chunk where the repetition occurs, with `debug: true` to inspect Gemma's chunk prompts. The line will appear in the transcript at `${TMPDIR}/comfyui-hr-endless-sampler/last_gemma_chunk_prompts.txt`.

### Room Consistency (Comment 4b)

**Fix (prompt-side, done):** Each scene now includes a room reference image as `<Subject 1>` with a specific visual description. The `retention_analysis` marks it `fully_preserved` across all shots. This anchors the room's appearance to a concrete image, so H3 generates consistent room geometry, lighting, and furniture across all cuts.

**Action required from the reviewer:** Generate one reference image per unique room (9 total):
1. Helion Labs conference room (San Francisco, hard daylight)
2. Helion Labs office (San Francisco, desk + transcripts)
3. Helion security operations room (San Francisco)
4. Helion operations room (San Francisco)
5. The Commons war room (Paris, overcast daylight)
6. Press room (Paris)
7. Black Hat auditorium (San Francisco)
8. Senate hearing room (packed)
9. Dark office at night (San Francisco)

Load these into the `ref_image_0` slots of `MiniMaxH3ReferenceToVideo` **in the order they appear in each scene's `subject_definitions`**. For scenes with no room (pure THE_LATTICE scenes 3, 4, 7, 10, 12, 23, 25), load **zero** reference images.
