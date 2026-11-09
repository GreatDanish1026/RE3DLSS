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

## 1.2.0: DLSS Ray Reconstruction (2026-11-09)

Ported from RE2DLSS 1.2.0 (its `docs/FINDINGS.md` "Ray tracing chain" names the passes). RE3's
chain is RE2's with two differences (captures in the game folder, dumps in `docs/shaders/rt`):

| Step | RE2 | RE3 |
|---|---|---|
| shade specular (rayTracingShading) | cs2f1da6b8 | **csde6f8914** (also reads the VFX pass's vfxSpecular, t18) |
| VFX denoise (median, spatial, temporal) | cs6b883e38, cs04bec84e, csc67ec41e | none |
| rayTracingUpscale | cse120127f, t5-t8 | **cs4e928452**: t0 depth, t2 BaseColorMetallic, t3 NormalXNormalYRoughnessMisc, t4 PreGF, t5 historyShY, t6 historyCoCg, t7 historySpecular -> u0 Result |

The rest (diffuse shading csfc23ed5b, VFX csddef9834, spatial csd6792963/cs25501ade, mean/variance,
temporal, deflicker, bilateral) are RE2's shaders. So the skip modes hand over three signals.

Test (2026-11-09, 4K HDR, RT reflections on): RR at DLAA and Quality (2561x1440 -> 3840x2160),
camera matrix check ok, G-buffer formats 29/24 as RE2, models D/E/F created, "Skip all but the
spatial passes" handed over 3 of 3 signals, switching RR on and off went back and forth to Super
Resolution cleanly. Default RrDenoise 0 (the game's denoiser, RR on top) as RE2.
