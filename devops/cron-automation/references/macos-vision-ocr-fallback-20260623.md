# macOS Vision Framework OCR Fallback — 2026-06-23

## When to Use

When `vision_analyze` fails with 401 auth errors (vision backend down or misconfigured) and you need to read text from a screenshot or image file.

## Technique

Use the macOS Vision framework via a Swift script to perform on-device OCR. No external API dependencies, no network calls, works offline.

## Script

Save as `/tmp/ocr_screenshot.swift`:

```swift
import Vision
import AppKit

let args = CommandLine.arguments
guard args.count > 1 else { exit(1) }
let path = args[1]

guard let image = NSImage(contentsOfFile: path),
      let cgImage = image.cgImage(forProposedRect: nil, context: nil, hints: nil) else {
    print("Failed to load image")
    exit(1)
}

let request = VNRecognizeTextRequest()
request.recognitionLevel = .accurate
request.recognitionLanguages = ["en-US"]

let handler = VNImageRequestHandler(cgImage: cgImage)
try? handler.perform([request])

guard let observations = request.results else { exit(0) }
for obs in observations {
    if let candidate = obs.topCandidates(1).first {
        print(candidate.string)
    }
}
```

## Invocation

```bash
swift /tmp/ocr_screenshot.swift "/path/to/screenshot.png"
```

Run from Python to process multiple images:

```python
import os
for img in ['/path/to/img1.png', '/path/to/img2.png']:
    print(f"\n=== {img} ===")
    result = os.popen(f'swift /tmp/ocr_screenshot.swift "{img}" 2>&1').read()
    print(result)
```

## Verification

Tested 2026-06-23 on two 1854x1584 and 1848x1474 pixel Retina screenshots. Successfully extracted all text including:
- Automation names, schedules, model names
- Full multi-paragraph prompt text with bullet points
- UI metadata (status, next run, last ran, project, repeats)

## Performance

- ~3-5 seconds per image on M-series Macs
- No GPU required
- Accuracy: excellent for screenshots with standard system fonts

## When NOT to Use

- If `vision_analyze` is working (it provides structured analysis, not just raw text)
- For non-text visual content (diagrams, photos) — Vision OCR extracts text only, not visual meaning
- In environments without Swift/Xcode Command Line Tools