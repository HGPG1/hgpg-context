# HGPG Favicon Canonical Asset Set

*Last Updated: 2026-05-06*

This directory holds the canonical favicon set for all HGPG web properties. Every site under the `homegrownpropertygroup.com` domain should use these files (or files derived from `master-512.png`) for browser-tab favicons, iOS home-screen icons, Android home-screen icons, and PWA install prompts.

## What's in here

| File | Size | Purpose |
|---|---|---|
| `master-512.png` | 512×512 | **Canonical source.** Regenerate any size from this. Do NOT discard. |
| `favicon.ico` | 16/32/48 multi-res | `<link rel="icon">` for older browsers and the auto-discovered `/favicon.ico` path |
| `favicon-16x16.png` | 16×16 | Browser tab fallback |
| `favicon-32x32.png` | 32×32 | Browser tab (modern) |
| `favicon-48x48.png` | 48×48 | Windows tile |
| `icon-192.png` | 192×192 | Android home screen, PWA |
| `icon-512.png` | 512×512 | PWA splash, install prompt |
| `apple-touch-icon.png` | 180×180 | iOS home screen icon |
| `site.webmanifest` | — | PWA manifest with HGPG navy `#2A384C` theme color |

## Design

HGPG navy `#2A384C` background. Light steel blue (`#A0B2C2`-ish) house silhouette with white tree silhouette inside. No wordmark.

The earlier wordmark version (which is what's currently deployed on TM, Buyers Guide, Sellers Guide, and New Construction as of 2026-05-06) was rejected as the new canonical because the "HOME GROWN PROPERTY GROUP" text becomes illegible at 32px and below. The icon-mark-only version reads cleanly all the way down to 16px.

If consistency across sites is desired, the four sites currently using the wordmark version should be re-skinned to use this canonical set in a future pass.

## How to use in a Next.js 13+ app (`app/` directory)

The simplest path uses Next.js auto-discovery. Drop these files directly into the `app/` directory:

```
app/
  favicon.ico            (auto-discovered as the favicon)
  icon.png               (rename icon-192.png — auto-discovered)
  apple-icon.png         (rename apple-touch-icon.png — auto-discovered)
```

For full PWA support including 512px and the manifest, also drop into `public/`:

```
public/
  icon-512.png
  site.webmanifest
```

And add to `app/layout.tsx` `<head>`:

```tsx
<link rel="manifest" href="/site.webmanifest" />
<meta name="theme-color" content="#2A384C" />
```

## How to use in Vite / non-Next apps

Drop the entire set into `public/`. Add explicit `<link>` tags to `index.html`:

```html
<link rel="icon" href="/favicon.ico" sizes="any" />
<link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png" />
<link rel="icon" type="image/png" sizes="192x192" href="/icon-192.png" />
<link rel="icon" type="image/png" sizes="512x512" href="/icon-512.png" />
<link rel="apple-touch-icon" sizes="180x180" href="/apple-touch-icon.png" />
<link rel="manifest" href="/site.webmanifest" />
<meta name="theme-color" content="#2A384C" />
```

## Regenerating from the master

If the canonical design ever changes, replace `master-512.png` and re-run the generation. Pillow (Python) is the simplest path:

```python
from PIL import Image
src = Image.open('master-512.png').convert('RGBA')

# PNGs
for name, size in {
    'favicon-16x16.png': 16,
    'favicon-32x32.png': 32,
    'favicon-48x48.png': 48,
    'icon-192.png': 192,
    'icon-512.png': 512,
    'apple-touch-icon.png': 180,
}.items():
    src.resize((size, size), Image.LANCZOS).save(name, optimize=True)

# Multi-res ICO
ico_sizes = [(16, 16), (32, 32), (48, 48)]
ico_imgs = [src.resize(s, Image.LANCZOS) for s in ico_sizes]
ico_imgs[0].save('favicon.ico', format='ICO', sizes=ico_sizes, append_images=ico_imgs[1:])
```

## Related notes

- Brand color `#2A384C` is the HGPG navy (per brand kit `kAHFKMi4Q7g`).
- The reference implementation already deployed (with the older wordmark version) lives in the `hgpg-transaction-manager` repo at `public/favicon.ico`, `public/icon-192.png`, `public/icon-512.png`, `public/apple-touch-icon.png`. Newer sites should pull from this canonical directory instead.
