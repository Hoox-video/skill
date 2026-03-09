# Hoox Skills

Agent skills for the [Hoox](https://hoox.video) AI video generation platform.

## Install

```bash
npx skills add hoox-video/skills
```

Or install to a specific agent:

```bash
npx skills add hoox-video/skills -a cursor
npx skills add hoox-video/skills -a claude-code
```

## Available Skills

| Skill | Description |
|-------|-------------|
| **hoox-api** | Interact with the Hoox public API to generate videos programmatically — script generation, video creation, export, avatars, voices, and webhooks. |

## What Can Your Agent Do With This?

Once installed, ask your coding agent things like:

- *"Generate a 60-second video about sustainable energy using the Hoox API"*
- *"List available voices and avatars from Hoox"*
- *"Create a script, generate the video, and export it to MP4"*
- *"Set up a webhook to receive video generation updates"*
- *"Duplicate an existing video with a different voice"*

The skill gives your agent full knowledge of the Hoox API: authentication, endpoints, request/response formats, polling strategies, error handling, and the credit system.

## Prerequisites

- An **Enterprise plan** on [app.hoox.video](https://app.hoox.video)
- An **API key** generated from Dashboard > Space Settings > API

## Resources

- [Hoox Website](https://hoox.video)
- [API Documentation](https://docs.hoox.video)
