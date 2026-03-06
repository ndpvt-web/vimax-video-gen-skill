# ViMax Video Generation Skill

Multi-shot AI video generation pipeline with face identity consistency across scenes. Adapted from [HKUDS/ViMax](https://github.com/HKUDS/ViMax) for the [HappyCapy AI Gateway](https://happycapy.ai).

---

## What This Is

A fork of ViMax rewired to run entirely through the HappyCapy AI Gateway. Takes a script or idea as input and produces a complete multi-shot video with consistent character faces across all scenes.

This repo also includes findings from **257 automated experiments** that improved face identity consistency by **70%** (face distance from 0.74 down to 0.221).

---

## How It Works

The pipeline runs through these stages:

1. **Character extraction** -- Pull characters from the script with detailed physical features
2. **Portrait generation** -- Generate front/side/back portraits for each character
3. **Storyboard design** -- Break the script into a shot-by-shot plan with first frame, last frame, and motion descriptions
4. **Reference image selection** -- Pick the best reference images for each frame (face identity is the top priority)
5. **Frame generation** -- Generate key frames using Gemini with character portraits as references
6. **Video generation** -- Generate video clips from frames using Veo
7. **Assembly** -- Concatenate all clips into a final video

---

## Pipelines

**Script-to-Video** -- Input a formatted screenplay. The pipeline extracts characters, builds storyboards, generates frames and video clips, and assembles the final output.

**Idea-to-Video** -- Input a brief idea (1-3 paragraphs). The pipeline first generates a full script via an LLM screenwriter agent, then runs the same video pipeline.

---

## Quick Start

### Prerequisites

- Python 3.12+
- [uv](https://docs.astral.sh/uv/getting-started/installation/) package manager
- `AI_GATEWAY_API_KEY` environment variable set to your HappyCapy API key

### Install

```bash
git clone https://github.com/ndpvt-web/vimax-video-gen-skill.git
cd vimax-video-gen-skill
uv sync
```

### Run Script-to-Video

Edit the `script`, `user_requirement`, and `style` variables in `main_happycapy_script2video.py`, then:

```bash
.venv/bin/python main_happycapy_script2video.py
```

### Run Idea-to-Video

Edit the `idea`, `user_requirement`, and `style` variables in `main_happycapy_idea2video.py`, then:

```bash
.venv/bin/python main_happycapy_idea2video.py
```

### Output

Results land in `.working_dir/script2video/` (or `idea2video/`):

```
.working_dir/script2video/
  characters.json
  character_portraits_registry.json
  character_portraits/
    0_CharacterName/
      front.png
      side.png
      back.png
  storyboard.json
  camera_tree.json
  shots/
    0/
      shot_description.json
      first_frame.png
      last_frame.png
      video.mp4
    1/
      ...
  final_video.mp4
```

---

## Configuration

HappyCapy config at `configs/happycapy_script2video.yaml`:

```yaml
chat_model:
  init_args:
    model: gpt-4.1
    model_provider: openai
    api_key: ${AI_GATEWAY_API_KEY}
    base_url: https://ai-gateway.happycapy.ai/api/v1/openai/v1

image_generator:
  class_path: tools.ImageGeneratorHappyCapyAPI
  init_args:
    api_key: ${AI_GATEWAY_API_KEY}
    model: google/gemini-3.1-flash-image-preview

video_generator:
  class_path: tools.VideoGeneratorHappyCapyAPI
  init_args:
    api_key: ${AI_GATEWAY_API_KEY}
    model: google/veo-3.1-generate-preview

working_dir: .working_dir/script2video
```

You can swap models:
- **Image**: `google/gemini-3.1-flash-image-preview` (recommended for face identity)
- **Video**: `google/veo-3.1-generate-preview` (recommended) or `openai/sora-2`
- **Chat**: `gpt-4.1` (recommended) or any OpenAI-compatible model

---

## Programmatic Usage

```python
import asyncio
from langchain.chat_models import init_chat_model
from tools.render_backend import RenderBackend
from utils.config_loader import load_config
from pipelines.script2video_pipeline import Script2VideoPipeline

config = load_config("configs/happycapy_script2video.yaml")
chat_model = init_chat_model(**config["chat_model"]["init_args"])
backend = RenderBackend.from_config(config)

pipeline = Script2VideoPipeline(
    chat_model=chat_model,
    image_generator=backend.image_generator,
    video_generator=backend.video_generator,
    working_dir=config["working_dir"],
)

asyncio.run(pipeline(
    script="Your screenplay here...",
    user_requirement="No more than 8 shots total.",
    style="Cinematic, warm lighting",
))
```

---

## Face Identity Consistency

This fork includes improvements validated through 257 experiments, achieving a **70% improvement** in face identity preservation across scenes.

### Results

| Metric | Baseline | After Improvements |
|--------|----------|--------------------|
| Face distance (VGG-Face cosine) | 0.74 | 0.221 |
| Verification rate | 5% | 100% |

### What Works

**Two-stage pipeline** -- Generate a still frame first (Gemini image gen with reference portraits), then pass that frame as the starting image to Veo for video generation. This anchors the character's face.

**Hyper-detailed physical descriptions** -- Include ethnicity, age, exact nose shape, eye spacing, jawline, skin tone, hair texture/style/color, glasses, facial hair, and distinguishing marks in every prompt. Simple, direct descriptions outperform clever prompt engineering.

**Front-view portrait as mandatory reference** -- Always include the character's front portrait when their face is visible in a shot. The reference image selector treats face identity as the #1 priority.

**Face-lock prompt on video generation** -- Every video generation prompt is prepended with an explicit instruction requiring the character's face to remain identical to the starting frame throughout the clip.

**Extreme close-up shots** -- Include at least one extreme close-up per character early in the video to anchor identity.

### What Does NOT Work

- Complex prompt engineering (viseme morphing, phoneme anchoring) performs worse than simple direct prompts
- LLM-generated experiment configurations averaged 0.450 face distance vs 0.333 for random -- worse, not better
- Lip-sync to external audio is not possible with text-to-video models (Veo generates its own internal audio)

See [`FACE_IDENTITY_GUIDE.md`](./FACE_IDENTITY_GUIDE.md) for full details.

---

## Using Your Own Reference Photos

To use real photos instead of AI-generated portraits:

```python
character_portraits_registry = {
    "Alice": {
        "front": {"path": "/path/to/alice_front.png", "description": "Front view of Alice"},
        "side": {"path": "/path/to/alice_side.png", "description": "Side view of Alice"},
        "back": {"path": "/path/to/alice_back.png", "description": "Back view of Alice"},
    }
}

await pipeline(
    script=script,
    user_requirement=user_requirement,
    style=style,
    character_portraits_registry=character_portraits_registry,
)
```

---

## Architecture

### Agents

| Agent | File | Purpose |
|-------|------|---------|
| CharacterExtractor | `agents/character_extractor.py` | Extract characters with static/dynamic features from script |
| CharacterPortraitsGenerator | `agents/character_portraits_generator.py` | Generate front/side/back portraits with identity-critical detail |
| StoryboardArtist | `agents/storyboard_artist.py` | Design shot-by-shot storyboard with first/last frames and motion |
| ReferenceImageSelector | `agents/reference_image_selector.py` | Select best reference images (face identity is #1 priority) |
| CameraImageGenerator | `agents/camera_image_generator.py` | Build camera trees and generate transition videos |
| BestImageSelector | `agents/best_image_selector.py` | Select best generated image from candidates |
| Screenwriter | `agents/screenwriter.py` | Generate scripts from ideas |

### Tools

| Tool | File | Purpose |
|------|------|---------|
| ImageGeneratorHappyCapyAPI | `tools/image_generator_happycapy_api.py` | Image generation via HappyCapy Gateway (Gemini) |
| VideoGeneratorHappyCapyAPI | `tools/video_generator_happycapy_api.py` | Video generation via HappyCapy Gateway (Veo) |
| RenderBackend | `tools/render_backend.py` | Factory for instantiating generators from config |

---

## Claude Code Skill

This repo can be installed as a Claude Code skill. Copy `SKILL.md` to `~/.claude/skills/vimax/SKILL.md` and the pipeline will be available as an agent skill for video generation tasks.

---

## Credits

Based on [HKUDS/ViMax](https://github.com/HKUDS/ViMax). Adapted for the HappyCapy AI Gateway with face identity improvements from 257 automated experiments.

## License

MIT
