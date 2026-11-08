# Minecraft Theme for NarraFork

Three dimension themes for [NarraFork](https://github.com/Jyno100), built from
vanilla Minecraft textures. Each theme follows the system light/dark setting and
picks a different scene per scheme.

| Theme | Light | Dark |
| --- | --- | --- |
| **Minecraft — Overworld** | daytime surface: scrolling clouds, a small village, mobs | night sky: stars, moon, circling phantoms |
| **Minecraft — Nether** | lit nether: netherrack, soul sand, lava falls, ghasts | same scene at void darkness |
| **Minecraft — The End** | violet-lit end island | the void-lit end island |

## Features

- **Declarative tokens only** — the theme is a `manifest.json` of colours,
  backgrounds, frames and fonts. No JavaScript, no CSS injection.
- **Vanilla textures** — every block texture is the real 16x16 asset, restored
  pixel-for-pixel from the 1072x1072 source and averaged down, never redrawn.
- **Animated backgrounds** — 512x300 GIFs, 20 frames, seamless loops, drawn as
  pixel art from those same textures.
- **Pixel font** — Fusion Pixel 10px subset at `fontSize: 20px` for an exact 2x
  integer scale (no blurry gridfitting).

## Install

1. Download `dist/minecraft-theme-<version>.nfplugin`.
2. In NarraFork: **Plugins → Install from file**, then enable it.
3. Pick a theme, then hard-refresh (Ctrl+F5).

Themes must be selected **once each** before the host compiles them — the theme
list only serves enabled themes.

## Layout

```
manifest.json    theme tokens (all three themes)
assets/          block textures, backgrounds, fonts
dist/*.nfplugin  packaged plugin
```

## Notable implementation details

The scenes are generated, not hand-drawn, so a few things are worth recording:

- **Cloud cell scale.** `clouds.png` has no long runs (median horizontal run: 2
  texels); it only reads as "big clouds" in game because one texel is rendered
  as a 12x12 block puff. Drawing it 1:1 made a salt-and-pepper sky, so the mask
  is quantised to 64x64 cells of 8px and closed (dilate then erode) to join
  specks into slabs. 64 cells x 8px == 512px keeps the 20-frame scroll looping
  seamlessly.
- **`opacity` is scrim strength, not image opacity.** It compiles to
  `color-mix(in srgb, var(--mantine-color-body) N%, transparent)`, so a larger
  value mutes the pattern more. Values above ~0.3 make a texture unreadable.
- **`--mantine-color-body` tints every surface at once.** A pale body colour
  (e.g. `rgba(255,236,226,.1)`) washes the whole UI — sidebar included — into a
  flat haze. It has to be a colour that can act as a scrim.
- **Light schemes need light-scene art.** The nether and the end are dark by
  nature; for a light scheme the terrain tones are lifted with a linear gain and
  the sky is designed bright, rather than compositing a translucent wash over
  the finished dark image (which reads as fog).

## Licence

Theme code and generated art: MIT. Minecraft is a trademark of Mojang; the
block textures remain Mojang's assets and are used here for a personal theme.
