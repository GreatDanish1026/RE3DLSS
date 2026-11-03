# Resident Evil 3 frame analysis

Game: Steam, E:\SteamLibrary\steamapps\common\RE3, re3.exe (DirectX 12 version), re3_config.ini
(PC* keys as RE2), ReShade 6.8.0, REFramework, RenoDX. 3840x2160; the captures came out SDR
(rgba8 back buffer, colour space 1). See RE2-FINDINGS.md and VILLAGE-FINDINGS.md.

## Post chain (captures: frame-1 = 100%, frame-2 = 50%, 1920x1080 -> 3840x2160)

RE3 shares RE2's TAA (PreTonemap2_PS ps89b84994: t1 depth, t2 colour, t3 history, t4 velocity) and
bloom composite (NewFinal psdd975721): RE2DLSS 0.0.1 ran DLSS straight away. After that the chain
is Village's:

| Pass | Shader | Reads | Writes |
|---|---|---|---|
| tonemap | psd127432d (vs00b71385) | t0 colour | #d6399 rgba16f |
| sharpen | ps6fbe0dd2 (FullScreenTriangleTexVS, like Village's Sharpness_PS) | t0 tonemapped | #d61ce |
| stretch | pse254af62 (FullScreenTriangleVS, like CopyImageWithDepthPS) | t0 depth, t1 tonemapped, t2 sharpened | 3840x2160 rgba16f + depth |
| output | ps4c385ccd | | back buffer |

The RE2 fork redirected the bloom draw to output size and then didn't recognise RE3's tonemap, so
the output-size viewport stayed for the game's render-size tonemap: a zoomed-in quarter of the
picture (user: "resized in an odd way").

0.1.0: RE3's ids (tonemap, sharpen in place, stretch as Village's OnCopy: the game's draw for its
depth, then the final image drawn over its colour). New safety net: after a redirected draw, the
game's own targets, viewport and bindings come back before the next draw the add-on doesn't
handle (draw, indexed and indirect draws). Shader names not confirmed by bytecode (structure and
vertex shaders match Village's).

Confirmed by the user (2026-11-03): looks right. Log: every DLSS mode via the dropdown
(RE3DLSS.lua works): DLAA 3840x2160, 2561x1440, 2227x1252, 1920x1080 -> 3840x2160, no fallbacks.
