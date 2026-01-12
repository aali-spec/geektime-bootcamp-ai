# Gemini Image Generation Skill

A reference guide for using the `google-genai` Python library to generate images with Google's Gemini models.

## Purpose

This skill provides Claude Code agents with comprehensive knowledge on how to correctly use the `google-genai` library for image generation tasks. When building new projects that need Gemini image capabilities, agents can reference this skill for:

- Correct API patterns and imports
- Configuration options (aspect ratios, sizes)
- Reference image usage for style transfer
- Async patterns for web applications
- Error handling best practices

## Key Information

- **Model:** `gemini-3-pro-image-preview`
- **Library:** `google-genai>=1.0.0`
- **Auth:** `GOOGLE_API_KEY` environment variable

## Quick Example

```python
from google import genai
from google.genai import types

client = genai.Client()

response = client.models.generate_content(
    model="gemini-3-pro-image-preview",
    contents=["A serene mountain landscape"],
    config=types.GenerateContentConfig(
        image_config=types.ImageConfig(
            aspect_ratio="16:9",
            image_size="2K",
        )
    ),
)

for part in response.parts:
    if part.inline_data is not None:
        with open("output.png", "wb") as f:
            f.write(part.inline_data.data)
```

## Documentation

See [SKILL.md](./SKILL.md) for complete documentation including:

- All configuration options
- 6 common usage patterns
- Reference image handling
- Async patterns
- Error handling
- Prompt engineering tips
