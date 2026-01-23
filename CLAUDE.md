---
name: remotion-elevenlabs-voiceover
description: Generate professional AI voiceovers for Remotion videos using ElevenLabs. Use when the user needs to create voiceovers, audio narration, or text-to-speech for video content. Features scene-based generation with request stitching, character presets (narrator, salesperson, expert, dramatic, calm), single scene regeneration, automatic timing validation, and pronunciation dictionaries.
allowed-tools: Bash(node:*), Bash(npx:*), Bash(ffprobe:*), Bash(ffmpeg:*)
---

# ElevenLabs Voiceover Generation

Generate professional AI voiceovers for Remotion videos using ElevenLabs API.

## Prerequisites

- `ELEVENLABS_API_KEY` in `.env.local`

## Quick Start

```bash
# Generate voiceover from text
node .claude/skills/elevenlabs/generate.js --text "Your text here" --output public/audio/voiceover.mp3

# Generate with narrator style (more natural)
node .claude/skills/elevenlabs/generate.js --text "Your text" --character narrator --output voiceover.mp3

# Generate scenes with request stitching
node .claude/skills/elevenlabs/generate.js --scenes remotion/scenes.json --output-dir public/audio/project/

# Regenerate a single scene
node .claude/skills/elevenlabs/generate.js --scenes scenes.json --scene scene2 --new-text "Updated text"

# List available voices and character presets
node .claude/skills/elevenlabs/generate.js --list-voices
node .claude/skills/elevenlabs/generate.js --list-characters
```

## Character Presets

Use character presets for more natural voiceovers instead of literal screen text reading:

| Character | Description | Best For |
|-----------|-------------|----------|
| `literal` | Reads text exactly as written | Screen text, quotes |
| `narrator` | Professional storyteller, smooth, engaging | Explainers, documentaries |
| `salesperson` | Enthusiastic, persuasive, energetic | Marketing, ads |
| `expert` | Authoritative, confident, knowledgeable | Legal content, tutorials |
| `conversational` | Casual, friendly, natural | Social media, casual content |
| `dramatic` | Intense, emotional, impactful | Hooks, problem statements |
| `calm` | Soothing, reassuring, gentle | Trust-building, conclusions |

```bash
# Use narrator style globally
node .claude/skills/elevenlabs/generate.js --scenes scenes.json --character narrator --output-dir public/audio/

# Or set per-scene in scenes.json
{
  "scenes": [
    { "id": "scene1", "text": "Problem statement", "character": "dramatic" },
    { "id": "scene2", "text": "Solution", "character": "calm" }
  ]
}
```

## Scene-Based Generation with Request Stitching

Generate multiple scenes with consistent prosody using ElevenLabs request stitching:

### scenes.json Format

```json
{
  "name": "product-demo",
  "voice": "Antoni",
  "character": "narrator",
  "scenes": [
    {
      "id": "scene1",
      "text": "Welcome to our product demo. This will change everything.",
      "duration": 4.5,
      "character": "dramatic"
    },
    {
      "id": "scene2",
      "text": "Simple setup. Powerful features. Instant results.",
      "duration": 5.5
    },
    {
      "id": "scene3",
      "text": "Get started today with our free trial. No credit card required.",
      "duration": 8,
      "delay": 0.3
    }
  ]
}
```

### Generate All Scenes

```bash
node .claude/skills/elevenlabs/generate.js \
  --scenes remotion/product-demo-scenes.json \
  --output-dir public/audio/product-demo/
```

This creates:
- `product-demo-scene1.mp3` through `sceneN.mp3`
- `product-demo-combined.mp3` (all scenes stitched)
- `product-demo-info.json` (metadata with durations)

### Single Scene Regeneration

If a scene starts too early, has wrong timing, or needs different text:

```bash
# Regenerate scene2 with new text
node .claude/skills/elevenlabs/generate.js \
  --scenes remotion/scenes.json \
  --scene scene2 \
  --new-text "Updated scene 2 text" \
  --output-dir public/audio/project/

# Regenerate scene3 with different character
node .claude/skills/elevenlabs/generate.js \
  --scenes remotion/scenes.json \
  --scene scene3 \
  --character salesperson \
  --output-dir public/audio/project/

# Just regenerate (same text, same character)
node .claude/skills/elevenlabs/generate.js \
  --scenes remotion/scenes.json \
  --scene scene1 \
  --output-dir public/audio/project/
```

The tool automatically:
- Uses request stitching from previous scenes for consistent prosody
- Updates the info.json file with new metadata
- Updates scenes.json if `--new-text` is provided

## Timing Validation

The skill automatically validates timing after generation using `ffprobe`:

### What It Checks

| Check | Threshold | Description |
|-------|-----------|-------------|
| Duration mismatch | >15% | Warns if actual differs from expected duration |
| Leading silence | >200ms | Audio starts late (voiceover delayed) |
| Trailing silence | >500ms | Unnecessary silence at end |
| Speaking rate | 2-4.5 wps | Optimal ~3 words/second for German |

### Validate Existing Audio

```bash
# Validate all scenes in a project
node .claude/skills/elevenlabs/generate.js --validate public/audio/product-demo/
```

Output example:
```
🔍 Validating product-demo (6 scenes)

❌ scene1: 3.00s (expected: 4.5s)
   ❌ Audio 1.50s shorter than expected
   👍 8 words @ 3.1 words/sec
⚠️ scene2: 6.35s (expected: 5.5s)
   ⚠️ Leading silence: 235ms (may start late)
   🐢 10 words @ 1.8 words/sec
✅ scene4: 4.36s (expected: 4s)
   👍 9 words @ 2.3 words/sec

📊 Total duration: 30.80s (expected: 30.00s)
```

### Updated info.json

After validation, the info.json includes actual measurements:
```json
{
  "scenes": [
    {
      "id": "scene1",
      "duration": 4.5,
      "actualDuration": 3.0,
      "leadingSilence": 0.05,
      "wordsPerSecond": 3.1
    }
  ]
}
```

Use `actualDuration` in your Remotion composition for precise sync.

## Options

| Option | Description | Default |
|--------|-------------|---------|
| `--text`, `-t` | Text to convert to speech | Required (or --file/--scenes) |
| `--file`, `-f` | Read text from file | - |
| `--output`, `-o` | Output file path | `output.mp3` |
| `--output-dir` | Output directory for scenes | `public/audio` |
| `--voice`, `-v` | Voice name or ID | `Antoni` |
| `--model`, `-m` | Model ID | `eleven_multilingual_v2` |
| `--character`, `-c` | Character preset | `literal` |
| `--scenes` | JSON file with scenes | - |
| `--scene` | Regenerate single scene ID | - |
| `--new-text` | New text for scene regen | - |
| `--validate` | Validate existing audio dir | - |
| `--skip-validation` | Skip auto-validation | false |
| `--stability` | Voice stability (0-1) | varies by character |
| `--similarity` | Voice similarity (0-1) | varies by character |
| `--style` | Style exaggeration (0-1) | varies by character |
| `--no-combined` | Skip combined file | false |

## Recommended Voices

| Voice | Style | Best For |
|-------|-------|----------|
| `Antoni` | Professional, warm | Legal content, explainers |
| `Arnold` | Authoritative, deep | Corporate, serious topics |
| `Josh` | Friendly, conversational | Marketing, casual content |

## Integration with Remotion

After generating scene voiceovers, use them in your composition:

```tsx
import { Audio, Sequence, staticFile } from "remotion";

// Use individual scene audio files for precise sync
const SCENE_DURATIONS = {
  scene1: 4.5,  // From info.json
  scene2: 5.5,
  scene3: 8.0,
};

export const VideoWithVoiceover: React.FC = () => {
  const { fps } = useVideoConfig();

  const scene1Frames = Math.round(SCENE_DURATIONS.scene1 * fps);
  const scene2Frames = Math.round(SCENE_DURATIONS.scene2 * fps);

  return (
    <>
      <Sequence from={0} durationInFrames={scene1Frames}>
        <Audio src={staticFile("audio/project/project-scene1.mp3")} />
        <Scene1Visual />
      </Sequence>

      <Sequence from={scene1Frames} durationInFrames={scene2Frames}>
        <Audio src={staticFile("audio/project/project-scene2.mp3")} />
        <Scene2Visual />
      </Sequence>
    </>
  );
};
```

## Tips for Best Results

1. **Use character presets**: Don't read screen text literally - use `narrator` or `expert` for natural flow
2. **Punctuation matters**: Use periods for pauses, commas for brief breaks
3. **Numbers**: Write out numbers ("fünfhundert" not "500")
4. **Abbreviations**: Write full words ("vierundzwanzig Stunden" not "24h")
5. **Scene-by-scene**: Different scenes can have different characters (dramatic intro, calm CTA)
6. **Fine-tune**: Use `--scene` to regenerate individual scenes without redoing everything
7. **Request stitching**: Keeps voice consistent across all scenes

## Workflow Example

```bash
# 1. Create scenes.json with your script
# 2. Generate all scenes with narrator style
node .claude/skills/elevenlabs/generate.js \
  --scenes remotion/my-video-scenes.json \
  --character narrator \
  --output-dir public/audio/my-video/

# 3. Preview in Remotion, notice scene2 starts too early
# 4. Regenerate just scene2 with updated text
node .claude/skills/elevenlabs/generate.js \
  --scenes remotion/my-video-scenes.json \
  --scene scene2 \
  --new-text "Slightly longer text to fill the visual timing" \
  --output-dir public/audio/my-video/

# 5. Update video composition with new duration from info.json
# 6. Repeat until timing is perfect
```
