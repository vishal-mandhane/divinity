# Store asset specifications

Checked against Google Play Console Help (answer 9866151) in September 2026. Store specs move — re-check the console wording at upload
time if an asset is rejected.

## Google Play

### Phone screenshots

| Property | Requirement |
|---|---|
| Aspect ratio | Any, within the ratio cap. **9:16 (portrait) or 16:9** is required for high-visibility placements |
| Minimum side | 320 px |
| Maximum side | 3840 px |
| **Ratio cap** | **longest side ≤ 2 × shortest side** |
| High-visibility minimum | 1080 × 1920 portrait (or 1920 × 1080 landscape) |
| Count | up to 8 per device type; at least 2 in total to publish |
| Format | JPEG or **24-bit PNG, no alpha channel** |
| File size | ≤ 8 MB each |
| Order | Console upload order is display order |

The ratio cap is the trap. Legal portrait sizes sit between 9:16 (1.78:1) and
2:1. Modern phone screens are 19.5:9 (≈2.17:1) and are **not legal** — a raw
device capture at native aspect must be re-composed, not resized.

Safe portrait choices: 1080 × 1920, 1440 × 2560, 1242 × 2208 (all ≤ 2:1).

### Other required Play assets

| Asset | Size | Notes |
|---|---|---|
| Feature graphic | 1024 × 500 | **Required.** Keep the centre clear — the play button overlays it when a promo video exists. No CTA, no price. |
| App icon | 512 × 512 | 32-bit PNG with alpha |
| Tablet screenshots | e.g. 1600 × 2560 portrait | Needed for large-screen surfacing; same ratio rules |

## Apple App Store

| Display class | Portrait size | Notes |
|---|---|---|
| iPhone 6.9" | **1320 × 2868** | Current master. Auto-scales to all smaller iPhones. |
| iPhone 6.9" (alt) | 1290 × 2796 | Older 6.7" size, still accepted |
| iPhone 6.9" (alt) | 1260 × 2736 | Also accepted |
| iPad 13" | 2064 × 2752 | Only required if the app ships on iPad |

- 1 to 10 screenshots per display class; at least 1 required.
- Uploading one 6.9" set is enough for the whole iPhone range.
- Apple's portrait ratio (≈2.17:1) is **not** Play-legal. See the ratio cap above.

## Social carousel (optional, same source art)

| Platform | Size | Notes |
|---|---|---|
| Instagram / LinkedIn feed | 1080 × 1350 (4:5) | Tallest ratio the feed shows uncropped |
| Instagram square | 1080 × 1080 | Safer for cross-posting |

4:5 is much shorter than a store frame — the caption block and the screen plate
both have to be re-laid-out, not letterboxed. Keep the bottom ~15% clear of
anything critical: overlays and captions sit there.

## Export checklist

- [ ] Correct pixel dimensions for the target store
- [ ] Ratio inside the store's legal range
- [ ] sRGB colour profile
- [ ] No alpha channel (Play)
- [ ] Under the file-size cap
- [ ] Filenames ordered so console upload order matches intended display order
