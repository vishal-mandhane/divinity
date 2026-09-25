# Divinity

Five skills for [Claude Code](https://claude.com/claude-code) that review an app
the way a strict senior would: every finding points at a file and line, every
number comes from a real source, and anything not checked is said out loud.

They were built while shipping a real Flutter + Firebase app, then cleaned up
for general use.

| Skill | What it does |
|---|---|
| `security-audit` | Finds security holes with proof: who can attack, what they send, what they get. Deepest on Flutter + Firebase (rules, Cloud Functions, App Check); the method works on any stack. |
| `product-review` | Reviews the app as a product: every screen scored 1–5, every flow counted, and every hardcoded product decision (time limits, caps, gates) listed with a recommendation. Read-only. |
| `ui-ux-review` | Checks a screen against real standards: WCAG contrast, touch target sizes, text scaling, dark mode, reduced motion, and the loading / empty / error / success states. |
| `store-screenshots` | Plans and exports Google Play and App Store screenshots at legal sizes, with captions that only quote text the app really contains. |
| `lottie-animations` | Suggests specific, non-generic animations for a Flutter screen (loaders, empty states, success moments), with the code to wire them in. |

## Install

**As a plugin (recommended):**

```
/plugin marketplace add vishal-mandhane/divinity
/plugin install divinity@divinity
```

**Or copy the skills** into your personal skills folder:

```bash
git clone https://github.com/vishal-mandhane/divinity
cp -r divinity/skills/* ~/.claude/skills/
```

Or into one project only: copy them to `<project>/.claude/skills/`.

## Use

Claude picks a skill on its own when your request matches its description.
You can also ask for one by name:

- "Run a security audit on the Firestore rules."
- "Do a product review of the whole app."
- "Review `lib/screens/profile_screen.dart` for accessibility and dark mode."
- "Plan the Play Store screenshots."
- "Suggest Lottie animations for the home screen."

If you installed the plugin, the skills are namespaced, e.g. `divinity:security-audit`.

## What these skills will not do

- `security-audit` and `product-review` do not change your code while auditing.
  Fixes come after, as a separate step you approve.
- `product-review` reads code only. It never touches your production database.
- None of them send your code anywhere.

## Standards and dates

Standards move. Each skill says when its facts were last checked:
OWASP Top 10 2025, OWASP Mobile Top 10 2024, WCAG 2.2, and the Google Play /
App Store asset specs as of September 2026. If a store rejects an asset, check
the console wording first.

## Contributing

Issues and pull requests are welcome. A good change fixes a wrong fact (with a
source), removes a false positive, or adds a check that caught a real bug.

## Star History

<a href="https://www.star-history.com/#vishal-mandhane/divinity&Date">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=vishal-mandhane/divinity&type=Date&theme=dark" />
    <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=vishal-mandhane/divinity&type=Date" />
    <img alt="Star History Chart" src="https://api.star-history.com/svg?repos=vishal-mandhane/divinity&type=Date" />
  </picture>
</a>

## License

MIT. See [LICENSE](LICENSE).
