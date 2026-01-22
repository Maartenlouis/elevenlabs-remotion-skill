# ElevenLabs Remotion Skill

Generate professional AI voiceovers for [Remotion](https://www.remotion.dev/) videos using the [ElevenLabs](https://elevenlabs.io/) API.

## Features

- **Scene-based generation** with request stitching for consistent prosody across scenes
- **Character presets** (narrator, salesperson, expert, dramatic, calm) for natural delivery
- **Single scene regeneration** for fine-tuning without redoing everything
- **Automatic timing validation** using ffprobe (duration, silence detection, speaking rate)
- **Pronunciation dictionaries** with text preprocessing fallback
- **Combined audio output** for easy preview

## Installation

```bash
npx skills add maartenvonoesen/elevenlabs-remotion-skill
```

Or manually add to your project:

```bash
# Clone to your .claude/skills directory
git clone https://github.com/maartenvonoesen/elevenlabs-remotion-skill.git .claude/skills/elevenlabs
```

## Prerequisites

Add your ElevenLabs API key to `.env.local`:

```bash
ELEVENLABS_API_KEY=your_api_key_here
```

## Quick Start

```bash
# Generate voiceover from text
node .claude/skills/elevenlabs/generate.js --text "Your text here" --output voiceover.mp3

# Generate with narrator style
node .claude/skills/elevenlabs/generate.js --text "Your text" --character narrator --output voiceover.mp3

# Generate scenes with request stitching
node .claude/skills/elevenlabs/generate.js --scenes scenes.json --output-dir public/audio/

# Regenerate a single scene
node .claude/skills/elevenlabs/generate.js --scenes scenes.json --scene scene2 --new-text "Updated text"

# List available voices
node .claude/skills/elevenlabs/generate.js --list-voices
```

## Character Presets

Use character presets for more natural voiceovers:

| Character | Description | Best For |
|-----------|-------------|----------|
| `literal` | Reads text exactly as written | Screen text, quotes |
| `narrator` | Professional storyteller, smooth | Explainers, documentaries |
| `salesperson` | Enthusiastic, persuasive | Marketing, ads |
| `expert` | Authoritative, confident | Legal content, tutorials |
| `conversational` | Casual, friendly | Social media |
| `dramatic` | Intense, emotional | Hooks, problem statements |
| `calm` | Soothing, reassuring | Trust-building, CTAs |

## Scene-Based Generation

Create a `scenes.json` file:

```json
{
  "name": "my-video",
  "voice": "Antoni",
  "character": "narrator",
  "scenes": [
    {
      "id": "scene1",
      "text": "Welcome to our product demo.",
      "duration": 3.5,
      "character": "dramatic"
    },
    {
      "id": "scene2",
      "text": "Here's how it works.",
      "duration": 4.0
    },
    {
      "id": "scene3",
      "text": "Get started today. Free trial available.",
      "duration": 3.0,
      "character": "calm"
    }
  ]
}
```

Generate all scenes:

```bash
node .claude/skills/elevenlabs/generate.js \
  --scenes scenes.json \
  --output-dir public/audio/my-video/
```

This creates:
- `my-video-scene1.mp3` through `my-video-sceneN.mp3`
- `my-video-combined.mp3` (all scenes stitched)
- `my-video-info.json` (metadata with actual durations)

## Timing Validation

The skill automatically validates timing after generation:

| Check | Threshold | Description |
|-------|-----------|-------------|
| Duration mismatch | >15% | Warns if actual differs from expected |
| Leading silence | >200ms | Audio starts late |
| Trailing silence | >500ms | Unnecessary silence at end |
| Speaking rate | 2-4.5 wps | Optimal ~3 words/second |

Validate existing audio:

```bash
node .claude/skills/elevenlabs/generate.js --validate public/audio/project/
```

## Pronunciation Dictionaries

Create custom pronunciation rules in PLS format:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<lexicon version="1.0"
      xmlns="http://www.w3.org/2005/01/pronunciation-lexicon"
      alphabet="ipa" xml:lang="en-US">
  <lexeme>
    <grapheme>MyBrand</grapheme>
    <alias>My Brand</alias>
  </lexeme>
</lexicon>
```

Use with:

```bash
node .claude/skills/elevenlabs/generate.js \
  --scenes scenes.json \
  --dictionary mybrand \
  --output-dir public/audio/
```

## Remotion Integration

Use the generated audio in your Remotion composition:

```tsx
import { Audio, Sequence, staticFile } from "remotion";

// Durations from my-video-info.json
const SCENE_DURATIONS = {
  scene1: 3.5,
  scene2: 4.0,
  scene3: 3.0,
};

export const VideoWithVoiceover: React.FC = () => {
  const { fps } = useVideoConfig();

  const scene1Frames = Math.round(SCENE_DURATIONS.scene1 * fps);
  const scene2Frames = Math.round(SCENE_DURATIONS.scene2 * fps);

  return (
    <>
      <Sequence from={0} durationInFrames={scene1Frames}>
        <Audio src={staticFile("audio/my-video/my-video-scene1.mp3")} />
        <Scene1Visual />
      </Sequence>
      <Sequence from={scene1Frames} durationInFrames={scene2Frames}>
        <Audio src={staticFile("audio/my-video/my-video-scene2.mp3")} />
        <Scene2Visual />
      </Sequence>
    </>
  );
};
```

## All Options

| Option | Description | Default |
|--------|-------------|---------|
| `--text`, `-t` | Text to convert | - |
| `--file`, `-f` | Read text from file | - |
| `--output`, `-o` | Output file path | `output.mp3` |
| `--output-dir` | Output directory for scenes | `public/audio` |
| `--voice`, `-v` | Voice name or ID | `Antoni` |
| `--model`, `-m` | Model ID | `eleven_multilingual_v2` |
| `--character`, `-c` | Character preset | `literal` |
| `--scenes` | JSON file with scenes | - |
| `--scene` | Regenerate single scene ID | - |
| `--new-text` | New text for scene regen | - |
| `--dictionary` | Pronunciation dictionary name | - |
| `--no-dictionary` | Disable default dictionary | false |
| `--validate` | Validate existing audio dir | - |
| `--skip-validation` | Skip auto-validation | false |
| `--stability` | Voice stability (0-1) | varies |
| `--similarity` | Voice similarity (0-1) | varies |
| `--style` | Style exaggeration (0-1) | varies |
| `--no-combined` | Skip combined file | false |
| `--list-voices` | List available voices | - |
| `--list-characters` | List character presets | - |
| `--list-dictionaries` | List pronunciation dicts | - |

## Tips for Best Results

1. **Use character presets** - Don't read screen text literally
2. **Punctuation matters** - Periods for pauses, commas for brief breaks
3. **Write out numbers** - "five hundred" not "500"
4. **Write out abbreviations** - "twenty-four hours" not "24h"
5. **Request stitching** - Keeps voice consistent across scenes
6. **Fine-tune with `--scene`** - Regenerate individual scenes without redoing everything

## Compatible Agents

- Claude Code
- Cursor
- Windsurf
- Other AI coding assistants

## License

MIT
