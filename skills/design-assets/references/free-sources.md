# Free Design-Asset Sources + Fetch Recipes (2026, verified)

Every source below is free and (where possible) programmatically fetchable. Commands verified July
2026. **Never hardcode API keys — read them from env.** Prefer the internal S3 stock and free/CC0
sources; during design do NOT spend effort on rights/legality (handled separately after go-live).

## API-key env map (free tiers)
| Env var | Used by | Get it |
|---|---|---|
| `UNSPLASH_ACCESS_KEY` | Unsplash (50/hr demo) | unsplash.com/developers |
| `PEXELS_API_KEY` | Pexels photos+videos (200/hr) | pexels.com/api |
| `PIXABAY_API_KEY` | Pixabay images+videos (100/60s) | pixabay.com/api/docs |
| `COVERR_API_KEY` | Coverr video | coverr.co/developers |
| `GOOGLE_FONTS_API_KEY` | Google Fonts *metadata* only | Google Cloud → Web Fonts API |

No key needed: Openverse (anon), Iconify, Simple Icons, Lucide/Heroicons/Phosphor/Tabler CDNs, Google
Fonts **CSS** API, Fontshare, Fontsource, LottieFiles CDN, Rive CDN, Poly Haven, Haikei.

---

## 0. Internal S3 stock (home-grown — check FIRST, treat like any stock site)

- **Local mount (browse as a normal folder):** `/Volumes/AiVibe-Cloud/stock/`
- **S3:** `s3://aivibe-mac-cloud/stock/` (region ap-south-1) · rclone remote `aivibe-cloud:`
- Subfolders: `icons/ images/ themes/ graphics/ videos/ illustrations/ fonts/ logos/`

```bash
# Browse locally (fastest — it's a mounted folder)
ls /Volumes/AiVibe-Cloud/stock/illustrations/
# Or via S3
aws s3 ls s3://aivibe-mac-cloud/stock/icons/
# Use it: COPY the chosen asset into the project (nothing links to S3 at runtime)
cp /Volumes/AiVibe-Cloud/stock/images/hero.jpg ./public/assets/
aws s3 cp s3://aivibe-mac-cloud/stock/videos/loop.mp4 ./public/assets/
```
Pick home-grown vs public **only** by which looks best. If neither the stock nor the sources below
satisfy the world-class benchmark, do additional research to find a better asset.

### Logos — recolor + background-treat BEFORE use (strong rule)
`stock/logos/` holds app-name-specific ready logos so agents don't waste time creating logos. Find
yours by app name — e.g. `ls /Volumes/AiVibe-Cloud/stock/logos/ | grep -i vedha` (AiVedha, AiVibe,
VibeMyTrip, VibeBuddy logos are present in SVG + PNG, dark/light variants; prefer the `.svg`). The
`images/` and `videos/` subfolders are also populated. Never apply a logo as-is — recolor to the theme
+ fix the background first:
```bash
# SVG logo — recolor by swapping fills (find current fills first)
grep -oE 'fill="#[0-9A-Fa-f]{3,6}"' logo.svg | sort -u
sed -i '' 's/#OLDHEX/#BRANDHEX/g' logo.svg           # repeat per fill; or set fill="currentColor" and drive via CSS color
# SVG logo — make it inherit theme color (single-color marks)
#   set fill="currentColor" on paths, then `color:` in CSS controls it
# PNG logo — recolor via ImageMagick + knock out a solid background to transparent
magick logo.png -fuzz 10% -transparent white logo-transparent.png     # remove white bg
magick logo.png -fuzz 10% -fill '#BRANDHEX' -opaque '#OLDHEX' logo-recolored.png
```
Prefer an SVG logo (trivially recolorable). A logo whose colors clash with the theme is a defect.

---

## 1. Photos / stock images
- **Unsplash** — `api.unsplash.com`; header `Authorization: Client-ID $UNSPLASH_ACCESS_KEY`; free commercial.
  ```bash
  curl -s "https://api.unsplash.com/photos/random?query=mountain&orientation=landscape" \
    -H "Authorization: Client-ID $UNSPLASH_ACCESS_KEY" | jq -r '.urls.regular'
  ```
- **Pexels** — `api.pexels.com/v1`; header `Authorization: $PEXELS_API_KEY`; no attribution required.
  ```bash
  curl -s "https://api.pexels.com/v1/search?query=team&per_page=5&orientation=landscape" \
    -H "Authorization: $PEXELS_API_KEY" | jq -r '.photos[].src.large2x'
  ```
- **Pixabay** — `pixabay.com/api/?key=$PIXABAY_API_KEY`; no attribution; **cache locally** (no permanent hotlink).
  `image_type=photo|illustration|vector`. `curl -s "https://pixabay.com/api/?key=$PIXABAY_API_KEY&q=flowers&image_type=photo" | jq -r '.hits[].largeImageURL'`
- **Openverse** — `api.openverse.org/v1/images/`; **no key**; add `&license=cc0` for attribution-free.
  ```bash
  curl -s "https://api.openverse.org/v1/images/?q=mountain&license=cc0&page_size=5" -H "User-Agent: app/1.0" | jq -r '.results[].url'
  ```

## 2. Illustrations
| Source | License | Fetch |
|---|---|---|
| **unDraw** (undraw.co) | MIT, recolorable | Download SVG, swap accent (default `#6c63ff`) |
| **Storyset** (storyset.com) | free w/ attribution | Customize + export SVG/PNG/**Lottie** |
| **Humaaans** (humaaans.com) | CC0 | Mix-&-match SVG people |
| **Open Peeps** (openpeeps.com) | CC0 | Hand-drawn SVG pack |
| **illlustrations.co** (3× "l") | MIT, no attribution | 120+ SVG/PNG pack |
| **DrawKit / absurd.design / Icons8 Ouch! / Blush** | free (check per pack) | site download |
| **3dicons.co** | CC0 | 1,500+ 3D icon renders |
```bash
# unDraw recolor to brand in one line
sed -i '' 's/#6c63ff/#0ea5e9/g' undraw_hero.svg
```

## 3. Icons (all free, MIT/ISC/CC0)
Flat sets on npm AND CDN:
```bash
curl -s "https://unpkg.com/lucide-static@latest/icons/house.svg"                                   # Lucide (ISC)
curl -s "https://cdn.jsdelivr.net/gh/tailwindlabs/heroicons/optimized/24/outline/home.svg"          # Heroicons (MIT)
curl -s "https://cdn.jsdelivr.net/npm/@phosphor-icons/core@latest/assets/regular/house.svg"         # Phosphor (MIT)
curl -s "https://unpkg.com/@tabler/icons@latest/icons/outline/home.svg"                             # Tabler (MIT)
```
**Iconify — one API for 200k+ icons, recolored/sized on the fly, no key:**
```bash
curl -s "https://api.iconify.design/mdi/home.svg?color=%23ff0000&width=48"     # any set, recolored
curl -s "https://api.iconify.design/search?query=home&limit=10"                # fuzzy search all sets
# React: `npm i @iconify/react` → <Icon icon="lucide:home" />
```
**Simple Icons — brand/logo icons, CC0, recolor via path:**
```bash
curl -s "https://cdn.simpleicons.org/github/1DA1F2"   # slug + hex (this IS a recolor)
```
**Material Symbols:** `<link href="https://fonts.googleapis.com/css2?family=Material+Symbols+Outlined">`.
Pick ONE icon set per project (Lucide default) for consistent stroke weight.

## 4. Animations & motion graphics
- **LottieFiles** — Lottie `.json` / dotLottie `.lottie`; hosted at `https://lottie.host/<uuid>/<name>.lottie`.
  ```html
  <script src="https://cdn.jsdelivr.net/npm/@lottiefiles/dotlottie-wc@latest/dist/index.js" type="module"></script>
  <dotlottie-wc src="https://lottie.host/<uuid>/<name>.lottie" autoplay loop style="width:300px;height:300px"></dotlottie-wc>
  ```
  npm: `@lottiefiles/dotlottie-react`, `lottie-react`.
- **Lordicon** — animated icons; free tier = attribution required (Lottie/GIF/SVG).
- **Rive** — interactive `.riv`; select the current platform package and renderer through
  `interactive-animation-systems`; community files are at `rive.app/community`.
- Others: useAnimations (micro icon animations), SVGator, Lottielab.

## 5. Video / motion backgrounds
- **Pexels Videos** (same key): `curl -s "https://api.pexels.com/videos/search?query=ocean&per_page=3" -H "Authorization: $PEXELS_API_KEY" | jq -r '.videos[].video_files[]|select(.quality=="hd").link'`
- **Mixkit** (mixkit.co) — no key, no attribution, no signup; direct download from clip page. Best instant pick.
- **Coverr** — free REST API (attribution on free tier): `curl -s "https://api.coverr.co/videos?api_key=$COVERR_API_KEY&urls=true&page_size=5" | jq '.hits'`
- **Videvo** — mixed per-clip license; check each.

## 6. Fonts
- **Google Fonts CSS API (no key)** — load or self-host:
  `<link href="https://fonts.googleapis.com/css2?family=Inter:wght@100..900&display=swap" rel="stylesheet">`
  (curl the CSS to grab the gstatic .woff2 URLs and self-host.)
- **Fontshare (no key)** — `<link href="https://api.fontshare.com/v2/css?f[]=satoshi@400,700&display=swap">`; list: `.../v2/fonts?limit=10`.
- **Fontsource (no key, self-host)** — `npm i @fontsource/inter` → `import '@fontsource/inter/400.css'`; CDN woff2 at `cdn.jsdelivr.net/fontsource/fonts/inter@latest/latin-400-normal.woff2`.
- **Google Fonts Developer API (metadata, needs key)** — `curl -s "https://www.googleapis.com/webfonts/v1/webfonts?key=$GOOGLE_FONTS_API_KEY&sort=popularity"`.

## 7. 3D / textures / gradients
- **Poly Haven** — CC0 HDRIs/textures/models, no key: `curl -s "https://api.polyhaven.com/files/<asset_id>" | jq '.Diffuse'`.
- **Haikei** (app.haikei.app) — blobs/waves/mesh/gradients → export SVG/PNG/CSS. **Get Waves** (getwaves.io) for wave dividers.
- **Spline** (spline.design) — browser 3D; 3D-Library models free for commercial; embed via `@splinetool/react-spline`.

---

## Quick decision guide (need → source → one-liner)
| Need | Best pick | Fetch |
|---|---|---|
| Polished hero photo | Unsplash | `.../photos/random?query=Q&orientation=landscape` → `.urls.regular` |
| Bulk no-attribution photos | Pexels | `.../v1/search?query=Q` → `.photos[].src.large2x` |
| Guaranteed CC0 photo, no key | Openverse | `.../v1/images/?q=Q&license=cc0` → `.results[].url` |
| Recolorable spot illustration | unDraw | download SVG → `sed 's/#6c63ff/#HEX/g'` |
| Animated illustration | Storyset / Lottie | export Lottie → dotlottie-wc |
| Any icon, recolored, no key | Iconify | `api.iconify.design/lucide/home.svg?color=%23HEX&width=24` |
| Consistent icon set in React | Lucide | `npm i lucide-react` → `<Home />` |
| Brand/company logo icon | Simple Icons | `cdn.simpleicons.org/SLUG/HEX` |
| Loading/success animation | LottieFiles | `<dotlottie-wc src="lottie.host/UUID/name.lottie" autoplay loop>` |
| Interactive/stateful animation | Rive | `.riv`; choose the runtime/renderer through `interactive-animation-systems` |
| Background video | Mixkit / Pexels Videos | Pexels: `.../videos/search?query=Q` → `video_files[].link` |
| Display/body font (CDN) | Google Fonts / Fontshare | `<link href="fonts.googleapis.com/css2?family=Inter:wght@400;700">` |
| Self-hosted font (no external calls) | Fontsource | `npm i @fontsource/inter` |
| CC0 texture/HDRI/3D | Poly Haven | `api.polyhaven.com/files/ID` → `.Diffuse` |
| Gradient/blob/wave bg | Haikei | app.haikei.app → export SVG |

## Sources
unsplash.com/documentation · pexels.com/api/documentation · pixabay.com/api/docs · api.openverse.org ·
iconify.design/docs/api · simpleicons.org · lucide.dev · heroicons.com · phosphoricons.com · tabler.io/icons ·
developers.lottiefiles.com · rive.app/docs/llms.txt · api.coverr.co/docs · mixkit.co · api.polyhaven.com ·
developers.google.com/fonts · fontshare.com · fontsource.org · undraw.co · storyset.com · haikei.app.
