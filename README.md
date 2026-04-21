# script-to-video

Turn a topic, a finished script, an audio file, or a talking-head video into a timed HyperFrames composition.

This repo contains a public, sanitized version of my `script-to-video` skill. It is meant to be copied into a skills folder and adapted to your own machine.

## What This Skill Does

The skill supports three main workflows:

1. Standard script-to-video
   Give it a topic, script text, or a file. It can generate voiceover audio, transcribe it, build a sentence-level storyboard, and compose a HyperFrames project.

2. Catalog showcase
   Start from an existing audio or video file and build a showcase composition that uses a wide range of scene types.

3. Talking-cut
   Start from a talking-head video and alternate between face-cam footage and graphic cutaway scenes while keeping the original audio in sync.

## What Is In This Repo

- `SKILL.md`
  The full skill definition and workflow instructions.

- `themes/`
  Theme JSON files used by the skill for visual styling.

## What Is Not In This Repo

This public copy does not include:

- Private API keys
- Personal file paths
- Voice sample audio files
- Local doodle libraries
- HyperFrames project examples from my machine

Those parts are referenced as placeholders in `SKILL.md` and need to be replaced with values that exist on your system.

## Requirements

You will need your own local setup for the parts this skill expects:

- HyperFrames CLI
- `ffmpeg` and `ffprobe`
- A local OmiVoice server, if you want TTS synthesis
- A reference voice sample file, if you want cloned voice output
- `PEXELS_API_KEY`, if you want to use `pexels-hero` scenes
- An optional doodle asset library, if you want to use `doodle-split`

## Quick Start

1. Copy this folder into your skills location.
2. Open `SKILL.md`.
3. Replace the placeholder local paths with paths that exist on your machine.
4. Set `PEXELS_API_KEY` in your environment if you want Pexels media fetching.
5. Make sure your local video and audio tools are installed.
6. Use one of the prompts below.

## Example Prompts

- `make a video about why user stories matter`
- `catalog showcase from "C:\media\demo.mp4"`
- `talking-cut from "C:\media\founder-intro.mp4"`

## Important Notes

- This repo is designed to be practical, not plug-and-play.
- You still need to wire it to your own local tooling.
- The public copy keeps the workflow, scene catalog, and themes, but strips out secrets and machine-specific details.

## Repo Structure

```text
script-to-video-skill/
  README.md
  SKILL.md
  themes/
```

## License

No license file has been added yet. If you plan to share or accept contributions, add one before promoting the repo widely.