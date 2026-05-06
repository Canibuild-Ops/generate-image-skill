# generate-image

A Claude Code skill for generating AI images using Google's Gemini Flash Image model.

## Features

- Text-to-image generation with customizable prompts
- Image editing with reference images
- Style transfer and character consistency
- Multiple aspect ratios (1:1, 16:9, 9:16, etc.)
- Obsidian wikilink output for easy embedding

## Installation

```bash
# Via skill-manager
skill.sh install https://github.com/Canibuild-Ops/generate-image-skill.git

# Manual
git clone https://github.com/Canibuild-Ops/generate-image-skill.git ~/.claude/skills/generate-image
```

## Setup

Add your Gemini API key to `~/.claude/settings.json`:

```json
{
  "env": {
    "GEMINI_API_KEY": "your-api-key-here"
  }
}
```

Get an API key at [aistudio.google.com/apikey](https://aistudio.google.com/apikey).

### Optional: Set output directory

By default, images save to the current working directory. To customize:

```json
{
  "env": {
    "IMAGE_OUTPUT_DIR": "/path/to/your/images"
  }
}
```

## Usage

Ask Claude to generate an image:

```
"Generate an image of a sunset over mountains"
"Create a watercolor illustration of a cozy reading nook"
"Edit this image to make it more vibrant" (with reference)
```

### Direct script usage

```bash
# Basic generation
python3 $SKILL_DIR/scripts/generate.py "a sunset over mountains" --name "sunset"

# Custom aspect ratio
python3 $SKILL_DIR/scripts/generate.py "a wide landscape" --name "landscape" --aspect 16:9

# Edit with reference image
python3 $SKILL_DIR/scripts/generate.py "Make this more vibrant" --name "vibrant" --reference photo.png

# Multiple references for style transfer
python3 $SKILL_DIR/scripts/generate.py "Combine these styles" --name "merged" --reference img1.png img2.png

# Custom output path
python3 $SKILL_DIR/scripts/generate.py "a robot" --output /tmp/robot.png
```

### Options

| Flag | Description | Default |
|------|-------------|---------|
| `prompt` | Image description (required) | — |
| `--name` | Descriptive filename | `generated` |
| `--aspect` | Aspect ratio | `1:1` |
| `--reference` | Reference image(s), up to 14 | — |
| `--output` | Custom output path | `$IMAGE_OUTPUT_DIR/<name>-<timestamp>.png` |

### Supported aspect ratios

`1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `4:5`, `5:4`, `9:16`, `16:9`, `21:9`

## Tips for better results

- **Be specific**: "A golden retriever puppy playing in autumn leaves, soft afternoon light" > "a dog"
- **Include style**: "watercolor illustration", "photorealistic", "minimalist vector art"
- **Specify mood**: "warm and cozy", "dramatic and moody", "bright and cheerful"

## Requirements

- Python 3.8+
- No external dependencies (uses stdlib only)
- A [Gemini API key](https://aistudio.google.com/apikey)

## License

MIT
