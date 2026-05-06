---
name: generate-image
description: Generate AI images using Google's Nano Banana 2 (Gemini 3.1 Flash Image) model. Use when the user asks to create, generate, or make an image, picture, illustration, or visual. Saves images to Archive/Files/ and provides wikilink for embedding.
allowed-tools: Bash Read Write
---

# Image Generation Skill

Generate images using Google's Nano Banana 2 (Gemini 3.1 Flash Image) model and save them to the vault.

---

## Usage

**User asks:**
```
"Generate an image of a sunset over mountains"
"Create a picture of a cozy reading nook"
"Edit this image to make it more vibrant" (with reference)
"Combine the style of these two images" (with references)
```

**Claude will:**
1. Run the generation script with the prompt
2. Save the image to `Archive/Files/` with a timestamped filename
3. Provide the wikilink for embedding: `![[Archive/Files/generated-TIMESTAMP.png]]`

---

## Quick Reference

### Generate an Image

Run the script:
```bash
python3 $SKILL_DIR/scripts/generate.py "your detailed prompt here" --name "descriptive-name"
```

**Required:**
- `prompt` - The image description
- `--name` - Descriptive name for the file (e.g., "sunset-mountains", "robot-garden")

**Optional flags:**
- `--aspect 16:9` - Aspect ratio (default: 1:1)
- `--reference path/to/image.png` - Reference image(s) for editing or style guidance
- `--output /custom/path.png` - Custom output path (overrides --name)

**Supported aspect ratios:** 1:1, 2:3, 3:2, 3:4, 4:3, 4:5, 5:4, 9:16, 16:9, 21:9

### Image Editing with References

Edit existing images or use them as style guides:
```bash
# Edit an existing image
python3 generate.py "Make this image more vibrant and add warm lighting" \
  --name "vibrant-photo" \
  --reference "/path/to/Archive/Files/original.png"

# Use multiple references for style
python3 generate.py "Create a new image combining these styles" \
  --name "merged-style" \
  --reference style1.png style2.png

# Maintain character consistency
python3 generate.py "The same person in a different setting" \
  --name "character-variation" \
  --reference character-ref.png
```

**Reference limits:** Up to 14 images total (6 objects, 5 humans for consistency)

### Embed in Notes

After generation, embed with:
```markdown
![[Archive/Files/sunset-mountains-20250128-143052.png]]
```

Or with custom width:
```markdown
![[Archive/Files/generated-20250128-143052.png|500]]
```

---

## Setup Required

### API Key Configuration

The `GEMINI_API_KEY` must be set in `~/.claude/settings.json`:

```json
{
  "env": {
    "GEMINI_API_KEY": "your-api-key-here"
  }
}
```

**To get an API key:**
1. Go to https://aistudio.google.com/apikey
2. Create a new API key
3. Add it to settings.json as shown above

This keeps the key secure:
- Not committed to any repository
- Only available to Claude Code sessions
- Scoped to your user account

---

## Workflow

1. **Receive prompt** - User describes desired image
2. **Enhance prompt** (optional) - Add detail for better results
3. **Run script** - Execute with prompt and desired aspect ratio
4. **Verify output** - Check the generated image path
5. **Embed in note** - Provide wikilink in appropriate format

---

## Tips for Better Results

- **Be specific:** "A golden retriever puppy playing in autumn leaves, soft afternoon light" > "a dog"
- **Include style:** "watercolor illustration", "photorealistic", "minimalist vector art"
- **Specify mood:** "warm and cozy", "dramatic and moody", "bright and cheerful"
- **Add context:** "for a blog header", "as a profile picture", "for a presentation slide"

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "GEMINI_API_KEY not set" | Add key to `~/.claude/settings.json` under `env` |
| "Request failed" | Check API key validity at aistudio.google.com |
| "Rate limited" | Wait a moment and retry |
| Image not appearing | Verify path in Archive/Files/, refresh Obsidian |
