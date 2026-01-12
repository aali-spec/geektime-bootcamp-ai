---
name: gemini-image
description: Reference guide for using google-genai Python library to generate images with gemini-3-pro-image-preview model. Use this skill when building new projects that need Gemini image generation capabilities, to understand the correct API patterns, configuration options, and best practices.
---

# Gemini Image Generation Guide

A comprehensive reference for using the `google-genai` Python library to generate images with the `gemini-3-pro-image-preview` model.

## Quick Start

### Installation

```bash
pip install google-genai>=1.0.0
# or with uv
uv add google-genai
```

### Environment Setup

```bash
export GOOGLE_API_KEY='your-api-key-here'
```

### Minimal Example

```python
from google import genai
from google.genai import types

client = genai.Client()  # Uses GOOGLE_API_KEY from environment

response = client.models.generate_content(
    model="gemini-3-pro-image-preview",
    contents=["A cat sitting on a windowsill watching rain"],
    config=types.GenerateContentConfig(
        image_config=types.ImageConfig(
            aspect_ratio="16:9",
            image_size="2K",
        )
    ),
)

# Extract and save image
for part in response.parts:
    if part.inline_data is not None:
        with open("output.png", "wb") as f:
            f.write(part.inline_data.data)
        break
```

---

## API Reference

### Client Initialization

```python
from google import genai

# Option 1: Use environment variable (recommended)
client = genai.Client()  # Reads GOOGLE_API_KEY

# Option 2: Explicit API key
client = genai.Client(api_key="your-api-key")
```

### Core Method: `generate_content`

```python
response = client.models.generate_content(
    model="gemini-3-pro-image-preview",
    contents=contents,           # str, list, or Part objects
    config=GenerateContentConfig(...)
)
```

### Configuration Options

```python
from google.genai import types

config = types.GenerateContentConfig(
    # Response type control
    response_modalities=["TEXT", "IMAGE"],  # or ["IMAGE"] for image-only

    # Image-specific settings
    image_config=types.ImageConfig(
        aspect_ratio="16:9",   # See supported ratios below
        image_size="2K",       # "1K", "2K", or "4K"
    )
)
```

### Supported Aspect Ratios

| Ratio | Use Case |
|-------|----------|
| `1:1` | Square, social media posts |
| `2:3` | Portrait photos |
| `3:2` | Landscape photos |
| `3:4` | Portrait, mobile screens |
| `4:3` | Standard display |
| `4:5` | Instagram portrait |
| `5:4` | Large format |
| `9:16` | Vertical video, stories |
| `16:9` | Widescreen, presentations |
| `21:9` | Ultra-wide, cinematic |

### Image Sizes (gemini-3-pro-image-preview only)

| Size | Resolution | Use Case |
|------|------------|----------|
| `1K` | ~1024px | Thumbnails, previews |
| `2K` | ~2048px | Standard output (recommended) |
| `4K` | ~4096px | High-quality prints, zoom |

**Important:** Use uppercase "K" (not "1k", "2k", "4k").

---

## Common Patterns

### Pattern 1: Basic Image Generation

```python
from google import genai
from google.genai import types

def generate_image(prompt: str, output_path: str = "output.png") -> bytes:
    """Generate an image from a text prompt."""
    client = genai.Client()

    response = client.models.generate_content(
        model="gemini-3-pro-image-preview",
        contents=[prompt],
        config=types.GenerateContentConfig(
            image_config=types.ImageConfig(
                aspect_ratio="16:9",
                image_size="2K",
            )
        ),
    )

    for part in response.parts:
        if part.inline_data is not None:
            image_data = part.inline_data.data
            with open(output_path, "wb") as f:
                f.write(image_data)
            return image_data

    raise ValueError("No image generated in response")
```

### Pattern 2: With Reference/Style Image

```python
from google import genai
from google.genai import types
from pathlib import Path

def generate_with_reference(
    prompt: str,
    reference_image_path: str,
    output_path: str = "output.png"
) -> bytes:
    """Generate an image using a reference image for style consistency."""
    client = genai.Client()

    # Load reference image
    ref_data = Path(reference_image_path).read_bytes()

    # Determine MIME type
    suffix = Path(reference_image_path).suffix.lower()
    mime_types = {".png": "image/png", ".jpg": "image/jpeg", ".jpeg": "image/jpeg"}
    mime_type = mime_types.get(suffix, "image/png")

    # Create Part from bytes
    ref_part = types.Part.from_bytes(data=ref_data, mime_type=mime_type)

    # Build contents with reference image first
    contents = [
        ref_part,
        f"Reference the style of the image above and generate: {prompt}"
    ]

    response = client.models.generate_content(
        model="gemini-3-pro-image-preview",
        contents=contents,
        config=types.GenerateContentConfig(
            image_config=types.ImageConfig(
                aspect_ratio="16:9",
                image_size="2K",
            )
        ),
    )

    for part in response.parts:
        if part.inline_data is not None:
            image_data = part.inline_data.data
            with open(output_path, "wb") as f:
                f.write(image_data)
            return image_data

    raise ValueError("No image generated in response")
```

### Pattern 3: Image Editing

```python
from google import genai
from google.genai import types
from pathlib import Path

def edit_image(
    image_path: str,
    edit_prompt: str,
    output_path: str = "edited.png"
) -> bytes:
    """Edit an existing image based on text instructions."""
    client = genai.Client()

    # Load image to edit
    image_data = Path(image_path).read_bytes()
    image_part = types.Part.from_bytes(data=image_data, mime_type="image/png")

    contents = [
        image_part,
        f"Edit this image: {edit_prompt}"
    ]

    response = client.models.generate_content(
        model="gemini-3-pro-image-preview",
        contents=contents,
        config=types.GenerateContentConfig(
            response_modalities=["IMAGE"],
        ),
    )

    for part in response.parts:
        if part.inline_data is not None:
            result = part.inline_data.data
            with open(output_path, "wb") as f:
                f.write(result)
            return result

    raise ValueError("No image generated in response")
```

### Pattern 4: Async Generation (for Web Apps)

```python
import asyncio
from google import genai
from google.genai import types

class GeminiImageClient:
    """Async-compatible Gemini image generation client."""

    MODEL_NAME = "gemini-3-pro-image-preview"

    def __init__(self, api_key: str | None = None):
        if api_key:
            self.client = genai.Client(api_key=api_key)
        else:
            self.client = genai.Client()

    async def generate_image(
        self,
        prompt: str,
        style_image: bytes | None = None,
        aspect_ratio: str = "16:9",
        image_size: str = "2K",
    ) -> bytes:
        """Generate an image asynchronously."""
        contents = []

        if style_image:
            image_part = types.Part.from_bytes(
                data=style_image,
                mime_type="image/png"
            )
            contents.append(image_part)
            contents.append(
                f"Reference the style of the image above and generate: {prompt}"
            )
        else:
            contents.append(prompt)

        # Run sync API call in thread pool
        response = await asyncio.to_thread(
            self.client.models.generate_content,
            model=self.MODEL_NAME,
            contents=contents,
            config=types.GenerateContentConfig(
                image_config=types.ImageConfig(
                    aspect_ratio=aspect_ratio,
                    image_size=image_size,
                )
            ),
        )

        for part in response.parts:
            if part.inline_data is not None:
                return part.inline_data.data

        raise ValueError("No image generated in response")
```

### Pattern 5: Multiple Style Candidates

```python
import asyncio
from google import genai
from google.genai import types

async def generate_style_candidates(
    style_prompt: str,
    count: int = 3,
    aspect_ratio: str = "16:9",
) -> list[bytes]:
    """Generate multiple style candidate images."""
    client = genai.Client()
    candidates = []

    base_prompt = (
        f"Generate an artistic image showcasing the '{style_prompt}' style. "
        "Make it visually distinctive and representative of this style."
    )

    for i in range(count):
        varied_prompt = f"{base_prompt} (variation {i + 1}, make it unique)"

        response = await asyncio.to_thread(
            client.models.generate_content,
            model="gemini-3-pro-image-preview",
            contents=[varied_prompt],
            config=types.GenerateContentConfig(
                image_config=types.ImageConfig(
                    aspect_ratio=aspect_ratio,
                    image_size="2K",
                )
            ),
        )

        for part in response.parts:
            if part.inline_data is not None:
                candidates.append(part.inline_data.data)
                break

    return candidates
```

### Pattern 6: With Multiple Reference Images

```python
from google import genai
from google.genai import types
from pathlib import Path

def generate_with_multiple_refs(
    prompt: str,
    reference_paths: list[str],
    output_path: str = "output.png"
) -> bytes:
    """Generate image with multiple reference images (up to 5 for high fidelity)."""
    client = genai.Client()

    contents = []

    # Add all reference images
    for ref_path in reference_paths:
        ref_data = Path(ref_path).read_bytes()
        suffix = Path(ref_path).suffix.lower()
        mime_type = "image/jpeg" if suffix in [".jpg", ".jpeg"] else "image/png"
        ref_part = types.Part.from_bytes(data=ref_data, mime_type=mime_type)
        contents.append(ref_part)

    # Add the prompt
    contents.append(
        f"Using the style and elements from the reference images above, generate: {prompt}"
    )

    response = client.models.generate_content(
        model="gemini-3-pro-image-preview",
        contents=contents,
        config=types.GenerateContentConfig(
            image_config=types.ImageConfig(
                aspect_ratio="16:9",
                image_size="2K",
            )
        ),
    )

    for part in response.parts:
        if part.inline_data is not None:
            image_data = part.inline_data.data
            with open(output_path, "wb") as f:
                f.write(image_data)
            return image_data

    raise ValueError("No image generated in response")
```

---

## Response Handling

### Extracting Images from Response

```python
# Response structure
response.candidates[0].content.parts  # List of Part objects

# Each part can be text or image
for part in response.parts:  # Shorthand for above
    if part.text is not None:
        print(f"Text: {part.text}")
    elif part.inline_data is not None:
        # Image data
        image_bytes = part.inline_data.data
        mime_type = part.inline_data.mime_type  # e.g., "image/png"
```

### Using PIL for Image Processing

```python
from PIL import Image
from io import BytesIO

# Convert bytes to PIL Image
for part in response.parts:
    if part.inline_data is not None:
        image = Image.open(BytesIO(part.inline_data.data))
        image.save("output.png")

        # Or use the convenience method
        image = part.as_image()  # Returns PIL Image directly
        image.save("output.png")
```

---

## Error Handling

```python
from google import genai
from google.genai import types

def safe_generate(prompt: str) -> bytes | None:
    """Generate image with proper error handling."""
    client = genai.Client()

    try:
        response = client.models.generate_content(
            model="gemini-3-pro-image-preview",
            contents=[prompt],
            config=types.GenerateContentConfig(
                image_config=types.ImageConfig(
                    aspect_ratio="16:9",
                    image_size="2K",
                )
            ),
        )

        for part in response.parts:
            if part.inline_data is not None:
                return part.inline_data.data

        # No image in response (content policy, etc.)
        print("Warning: No image generated - check prompt content")
        return None

    except Exception as e:
        print(f"Error generating image: {e}")
        return None
```

---

## Prompt Engineering Tips

### Be Descriptive
```python
# Bad - too vague
prompt = "cat, sunset"

# Good - detailed description
prompt = """
A fluffy orange tabby cat sitting on a wooden fence,
watching a vibrant sunset over rolling hills.
Warm golden and pink light illuminates the scene.
Photorealistic style with soft focus background.
"""
```

### Specify Visual Elements
- **Lighting:** "soft morning light", "dramatic side lighting", "golden hour"
- **Style:** "oil painting", "watercolor", "3D render", "photorealistic"
- **Mood:** "serene", "dramatic", "whimsical", "mysterious"
- **Composition:** "close-up portrait", "wide landscape", "bird's eye view"
- **Camera:** "35mm lens", "shallow depth of field", "wide angle"

### For Style Transfer
```python
# When using reference images, be explicit about what to transfer
contents = [
    ref_part,
    "Generate an image of a mountain village at dawn. "
    "Match the color palette, brushstroke style, and artistic mood "
    "of the reference image exactly."
]
```

---

## Pricing

| Model | Cost per Image |
|-------|---------------|
| gemini-3-pro-image-preview | ~$0.134 |
| gemini-2.5-flash-image | ~$0.039 |

---

## Common Issues

### "No image generated in response"
- Prompt may violate content policies
- Try simplifying or rephrasing the prompt
- Check if response_modalities includes "IMAGE"

### "Invalid image_size"
- Use uppercase: `"1K"`, `"2K"`, `"4K"` (not lowercase)
- Only available with gemini-3-pro-image-preview

### "API key not found"
```python
# Ensure environment variable is set
import os
assert os.getenv("GOOGLE_API_KEY"), "Set GOOGLE_API_KEY env var"
```

### Rate Limits
- Add delays between requests for batch operations
- Use exponential backoff for retries

---

## Dependencies

```toml
# pyproject.toml
[project]
dependencies = [
    "google-genai>=1.0.0",
    "pillow>=10.0.0",  # Optional, for image processing
]
```

---

## Language Support (Future)

Currently supported:
- Python

Planned:
- TypeScript/JavaScript
- Go
- Rust
