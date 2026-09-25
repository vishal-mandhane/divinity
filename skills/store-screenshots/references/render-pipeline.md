# Render pipeline

Goal: reproducible, offline, pixel-exact panels. No design tool, no manual export,
no device farm. Re-runnable when a caption changes.

Three stages: **source render → comp → export.**

## Stage 1 — source renders at master resolution

Never upscale an existing screenshot. Check its true size first:

```bash
python -c "
import struct
d = open('path/to/shot.png','rb').read(33)
print(*struct.unpack('>II', d[16:24]))
"
```

### Flutter (headless, no device or simulator)

Widget tests render real screens at any resolution with the real bundled fonts.
Set the viewport before pumping:

```dart
testWidgets('store: the home screen', (tester) async {
  tester.view.physicalSize = const Size(1320, 2868);  // the store master
  tester.view.devicePixelRatio = 3.0;
  addTearDown(tester.view.reset);

  await tester.pumpWidget(MyApp(state: AppState.demo()));  // one pinned state
  await tester.pump(const Duration(milliseconds: 300));

  await expectLater(
    find.byType(MyApp),
    matchesGoldenFile('store/raw/home.png'),
  );
});
```

Run with `flutter test --update-goldens <file>` to write the PNGs.

Notes that matter:
- A `flutter_test_config.dart` that loads real fonts is required, or every panel
  renders in Ahem boxes.
- `pumpAndSettle` hangs on any screen with a live `Timer.periodic`. Pump by hand.
- Scroll to the payload before capturing: `tester.drag` + `pump`. A panel shows
  roughly the top 60% of a screen, so a screen whose value is below the fold must
  be scrolled first, not cropped after.
- One seeded/demo state with a pinned clock across every shot (SKILL.md Rule 7).

### Other stacks

Any deterministic headless render works — the requirement is the *same* state
every run, real fonts, and master resolution. Avoid hand-taken device captures:
they drift between frames and carry status bars you then have to mask.

## Stage 2 — comp the panel

Compose caption + screen plate in HTML at the exact canvas size, then screenshot
it. Any headless browser works. With Playwright:

```bash
npx playwright screenshot --viewport-size=1080,1920 "file://$PWD/panel-01.html" panel-01.png
```

Optional: if the `hyperframes` CLI is installed, it renders HTML deterministically
with local fonts and also audits overflow and contrast:

```bash
npx hyperframes init panels --non-interactive
# index.html: data-width="1080" data-height="1920", one .clip per panel,
# each with data-start/data-duration of 1s
npx hyperframes check                       # layout overflow + WCAG contrast
npx hyperframes snapshot --at 0.5,1.5,2.5   # one PNG per panel
```

`check` is worth the detour: it catches text overflow and low-contrast captions
before export, which is exactly what fails at thumbnail size.

Comp rules:
- Embed fonts with `@font-face` from local files — a near-miss web substitute is
  visible next to a screenshot of the real thing.
- Match the product's own design laws. If the app bans gradients and shadows, a
  floating-device-on-a-gradient store frame advertises a different product.
- Give the plate an edge that survives a white store card (a darker ground, or an
  offset second sheet behind it).

## Stage 3 — export per store

Strip alpha and pin the profile. Play rejects an alpha channel:

```bash
# PNG, 24-bit, no alpha
ffmpeg -i panel-01.png -pix_fmt rgb24 -y play/01.png

# or JPEG if the file-size cap is tight
ffmpeg -i panel-01.png -q:v 2 -y play/01.jpg
```

Verify before upload:

```bash
ffprobe -v error -show_entries stream=width,height,pix_fmt -of default=nw=1 play/01.png
```

`pix_fmt=rgb24` means no alpha. Confirm width/height against
`references/store-specs.md` and check the ratio cap.

## Thumbnail proof (SKILL.md Rule 6)

```bash
ffmpeg -i panel-%02d.png -vf "scale=150:-1,tile=8x1" -y proof-strip.png
```

Open it and read the headlines. Anything illegible needs fewer words, not smaller
type. The strip also shows whether the set has a colour beat or reads as one block.
