---
title: "Turning Pre-Rendered Game Backgrounds into 3D Scenes with AI"
date: 2026-11-11
draft: true
description: "How to automatically turn the pre-rendered backgrounds of games like Onimusha: Warlords and Final Fantasy IX into 3D models with AI: depth and geometry estimation (Depth Anything 3, MoGe-2), single-image Gaussian splatting (Apple SHARP), generative world models (World Labs Marble, HunyuanWorld) and object reconstruction (SAM 3D), plus a practical pipeline and its limits."
summary: "Games like Onimusha: Warlords and Final Fantasy IX hide their worlds inside flat, pre-rendered images. In 2026, a handful of open AI models can lift those images back into 3D: depth maps, point clouds, Gaussian splats and even full meshes. Here's how the pieces fit together, what a practical automatic pipeline looks like, and where it still falls apart."
tags: ["gamedev", "ai", "3d", "gaussian splatting", "depth estimation", "final fantasy", "onimusha", "retro games", "blender", "modding"]
categories: ["Programming"]
---

Final Fantasy IX's Alexandria, the Mist-covered towns, Onimusha's burning castle corridors: for many of us these places feel like real, solid spaces. They aren't. Each screen is a single image, rendered once on an expensive workstation in the late 1990s, with a few real-time 3D characters walking on top.

For decades, that was the end of the story. You could upscale the images, but you couldn't walk *into* them. In 2026 that's changed. A new generation of AI models can take one flat image and estimate the 3D scene behind it, sometimes in under a second. This post looks at how to use them to turn pre-rendered backgrounds into 3D models, automatically, and what you realistically get out the other end.

## How pre-rendered backgrounds work

Pre-rendered backgrounds were a trick to get better-looking scenes than the hardware could draw in real time. Artists built and rendered the environments offline, and the game drew the real-time characters and objects on top of the resulting still image[^1].

The hard part was making the 3D characters fit into a 2D picture. The PlayStation had no Z-buffer, so games faked depth and occlusion with ordering tables, layered depth values or separate pre-rendered depth maps[^2]. In practice, each screen came with a few pieces of hidden 3D data:

- **A camera**, so the real-time characters are drawn with the same perspective as the background. Final Fantasy VII's field files, for example, have a dedicated section that defines the camera matrix[^3].
- **A walkmesh**, a simple set of triangles describing where characters can walk. FF7's field format stores the walkmesh triangles in their own section too[^3].
- **Layers**, so a character can walk *behind* a pillar. FF7's backgrounds are split into several layers, with lower layers covered by higher ones[^3].

That's worth remembering, because it means we're not starting from nothing. The game already knows where the camera was and roughly where the floor is. The AI only has to fill in everything else.

## Why do this at all?

The obvious use is remakes and fan projects: free camera, VR walkthroughs, new lighting, or a real 3D version of a place you love. But there are good technical reasons too:

- **The source art is gone.** Square Enix's own remaster of FF9 was held back by the lack of backups of the original high-resolution background artwork, apart from what a few artists had kept themselves[^4]. If the 3D source scenes are lost, reconstruction from the final images is the only way back.
- **Official remasters stop at upscaling.** Capcom's 2019 Onimusha: Warlords remaster simply blew up the old backdrops to fit modern resolutions instead of redoing them[^5], and reviewers noted that sharper characters make the blurry backgrounds look even worse by contrast[^6]. Fans had to step in with a 4K texture pack[^7]. The Onimusha 2 remaster likewise keeps upscaled pre-rendered backgrounds[^8].
- **Even AI upscaling works better with depth.** Parallax, depth-of-field, dynamic lighting and widescreen extension all need to know what's near and what's far.

## The starting point: AI upscaling

Fans have been feeding these backgrounds through neural networks since 2019. The **Moguri Mod** for Final Fantasy IX is the best-known example: it upscales the field backgrounds with ESRGAN and, since version 9.0, Stable Diffusion, aiming to make them sharper and less noisy while respecting the original art[^9]. Version 9.0 re-rendered every background with Stable Diffusion's help, mixed with the earlier result, and fully reworked the battle backgrounds[^10]. Resident Evil got the same treatment, with modders combining AI tools with thousands of manual edits[^11].

Upscaling is step one of our pipeline too. Every 3D method below works better with a clean, high-resolution input than with a 320×240 dithered image.

## The AI building blocks

There are four families of models worth knowing about. They give you increasingly "complete" 3D, at the cost of increasingly invented detail.

### 1. Monocular depth: Depth Anything

The simplest step: estimate how far away every pixel is. **Depth Anything V2** (NeurIPS 2024) was trained on synthetic images with perfect depth labels and comes in sizes from 25M to 1.3B parameters[^12]. Its successor, **Depth Anything 3** from ByteDance Seed, came out on 14 November 2025[^13]. It handles single images, multiple views and video, and its authors report that it beats DA2 at monocular depth[^14]. The code is on GitHub, and the main models are Apache 2.0[^15].

A depth map alone gives you a **2.5D relief**: displace a grid of vertices by depth and you have a mesh that looks right from the original camera and gets stretchy as you move away.

### 2. Geometry and camera: MoGe-2

Depth isn't enough to build real geometry. To turn a depth map into 3D points, you also need the camera's field of view, and guessing that from one image is hard[^16]. Microsoft's **MoGe** sidesteps the problem by predicting a 3D point for every pixel directly, and deriving the depth map and focal length from that[^17]. **MoGe-2** adds metric scale, sharper detail and normal maps, all from one forward pass[^18]. It also masks out regions with no defined geometry, such as sky[^19].

For our use case MoGe-2 is the workhorse: one image in, a point cloud, a normal map and a camera estimate out.

### 3. Single-image Gaussian splats: Apple SHARP

Instead of a mesh, you can produce a **3D Gaussian splat**: millions of small, soft, coloured blobs that render very convincingly. Apple's **SHARP**, released in December 2025, predicts a Gaussian splat from a single photo in one neural network pass, in under a second on a standard GPU[^20]. It's open source, with code and weights, and writes standard `.ply` splat files that common viewers can open[^21].

The catch is in the paper's own framing: SHARP renders *nearby* views. Move too far from the original viewpoint and artefacts and distortions break the illusion[^22]. For a fixed-camera game, that may be fine: a little parallax and head movement in VR, not a free-roaming camera.

### 4. Generative worlds and objects: Marble, HunyuanWorld, SAM 3D

The last family doesn't just lift what's visible; it *invents* what isn't.

- **World Labs Marble** turns a single image (or text, panorama, several photos or a video) into a navigable Gaussian splat world, and can export it as a splat, a triangle mesh or a collider mesh[^23]. It invents consistent geometry and off-screen surroundings[^23]. It's a paid service: high-quality mesh export takes about an hour per world[^24].
- **HunyuanWorld 1.0** from Tencent is a source-available alternative. Image-to-world is a two-step process: it first generates a 360° panorama from your image, then builds a layered 3D mesh world from that panorama[^25]. Note that its license is more "source available" than truly open, with use restrictions[^26].
- **SAM 3D Objects** from Meta reconstructs individual objects (geometry, texture and layout) from a single image and a mask, including the hidden back sides[^27]. It's released under the SAM license, so check the terms before commercial use[^28].

World Labs' own **IMAGE-BLASTER** toolkit shows how these combine: it uses Marble for the environment, cuts dynamic objects out of the image, turns them into separate meshes with Hunyuan 3D, and cleans up the remaining background with image-editing models, all orchestrated through Claude Code skills[^29]. That's close to what we want for game backgrounds.

## A practical automatic pipeline

Here's how I'd put it together for a fixed-camera game like Onimusha or FF9. Everything except step 7 can run unattended over a whole folder of backgrounds.

**1. Extract the background and its metadata.** Pull out the background layers, plus the camera and walkmesh if the game's format is documented. FF7's field format is well documented[^3], and FF9's PC version has a mature modding tool, Hades Workshop, that can read and rewrite field data[^30]. Even if you can't decode the camera, the background images alone are enough to start.

**2. Upscale.** Run the image through an upscaler (or use an existing pack like Moguri). Higher resolution means sharper depth edges later.

**3. Estimate geometry.** Run MoGe-2 to get a point map, normals and a field-of-view estimate. If you have the game's real camera, use it to correct the estimated focal length and scale. This is the single biggest quality win, because a wrong field of view warps the whole scene.

**4. Build a mesh.** Turn the point map into a grid mesh, and **cut triangles at depth discontinuities**, or the edge of a pillar will be stretched into a wall reaching back to whatever is behind it. Project the background image onto the mesh as its texture.

**5. Fill what the camera never saw.** Behind every pillar and around every corner is a hole. Options, from cheapest to most ambitious:
- Inpaint the hidden regions in the image and re-project.
- Use the game's own layers: a foreground layer peeled off reveals part of what's behind it.
- Send the image to a world model (Marble or HunyuanWorld) and use its output for the occluded parts.

**6. Separate objects.** Run SAM 3D Objects on prominent props (statues, barrels, furniture) so they become full, closed meshes instead of thin shells.

**7. Clean up by hand.** Use the walkmesh as the floor and collision, merge screens that share a location, and fix the worst artefacts in Blender.

A minimal version of steps 3 and 4 fits in a short Python script. This is a sketch, not production code, but it shows how little glue is needed:

```python
# Sketch: one background image -> textured mesh with MoGe-2.
# pip install git+https://github.com/microsoft/MoGe.git trimesh pillow
import numpy as np, torch, trimesh
from PIL import Image
from moge.model.v2 import MoGeModel

device = "cuda"
model = MoGeModel.from_pretrained("Ruicheng/moge-2-vitl-normal").to(device).eval()

img = np.asarray(Image.open("bg_upscaled.png").convert("RGB"))
x = torch.tensor(img / 255.0, dtype=torch.float32, device=device).permute(2, 0, 1)

with torch.no_grad():
    out = model.infer(x)

pts = out["points"].cpu().numpy()   # (H, W, 3), OpenCV camera coordinates
mask = out["mask"].cpu().numpy()    # valid pixels (no sky / undefined)
H, W = mask.shape

# One vertex per pixel, two triangles per pixel quad.
idx = np.arange(H * W).reshape(H, W)
a, b = idx[:-1, :-1], idx[:-1, 1:]
c, d = idx[1:, :-1], idx[1:, 1:]
faces = np.concatenate([np.stack([a, c, b], -1).reshape(-1, 3),
                        np.stack([b, c, d], -1).reshape(-1, 3)])

verts = pts.reshape(-1, 3)
z = verts[:, 2]
valid = mask.reshape(-1)

# Drop faces touching invalid pixels or spanning a big depth jump.
fz = z[faces]
jump = fz.max(1) / np.maximum(fz.min(1), 1e-6) > 1.05
keep = valid[faces].all(1) & ~jump
faces = faces[keep]

uv = np.stack(np.meshgrid(np.linspace(0, 1, W), np.linspace(1, 0, H)), -1).reshape(-1, 2)
mesh = trimesh.Trimesh(verts * [1, -1, -1], faces,  # OpenCV -> OpenGL axes
                       visual=trimesh.visual.TextureVisuals(uv=uv, image=Image.fromarray(img)))
mesh.export("bg_mesh.glb")
```

Loop that over every background, and you have a first-pass 3D version of the whole game in an afternoon. Then the real work starts.

## Where it breaks

It's worth being honest about the limits, because a single image simply doesn't contain the whole scene.

- **Everything off-camera is invented.** Depth models only see the visible surface. World models fill the gaps with *plausible* content, not the *original* content. The back of FF9's Alexandria Castle will be the model's guess, not the artist's.
- **Painterly art confuses models trained on photos.** Depth and geometry models learn from real and photorealistic synthetic images. Onimusha's backgrounds are photorealistic CG renders, which suits them well. FF9's stylised, hand-touched art is harder: researchers have noted that Gaussian splatting in particular struggles with stylised scenes such as cartoons and games[^31].
- **Lighting is baked in.** Shadows, glows and fog are painted into the texture. Move the camera or add real-time lights, and the baked lighting stays put. Removing it reliably is still an open problem.
- **Low-resolution, dithered sources.** PS1-era images are small and noisy. Upscaling helps, but the upscaler also invents detail, and the 3D model then faithfully reconstructs that invented detail.
- **Screens don't line up.** Each background was rendered from its own camera. Stitching neighbouring screens into one continuous level means aligning separate reconstructions with different scale errors. The game's walkmeshes and Depth Anything 3's multi-view mode help, but this remains manual work.

## What the result is good for

With those limits in mind, here's what's realistic in late 2026:

| Goal | Best approach | Quality |
| --- | --- | --- |
| Subtle parallax, VR "look around" from the original spot | SHARP or a MoGe-2 depth mesh | Very good |
| Dynamic lighting and depth effects in an upscale mod | Depth Anything 3 / MoGe-2 depth and normals | Good |
| Short camera moves (dolly, orbit) | MoGe-2 mesh + inpainting | Good with cleanup |
| Free-roaming 3D level | World model (Marble/HunyuanWorld) + manual stitching | Convincing but not faithful |
| Faithful 3D remake | AI output as a reference and blockout for artists | A big head start, not a replacement |

That last row matters most. Even Capcom, when it remastered *Resident Evil*, recreated some rooms as real 3D scenes rather than relying on the flat images[^32]. AI won't hand you that remake, but it will hand an artist a correctly scaled blockout of every room in the game, which used to be weeks of work.

## A note on rights

The backgrounds belong to Square Enix and Capcom. Reconstructing them for personal experiments, or for mods that require you to own the game (the way Moguri works on the Steam version), is one thing. Distributing extracted or reconstructed assets is another. Keep the scripts public and the assets private.

## Conclusion

The pieces are all here. An upscaler cleans the image, MoGe-2 or Depth Anything 3 recovers the geometry and the camera, SHARP turns it into a splat in a second, SAM 3D rebuilds the props, and world models like Marble and HunyuanWorld guess what the camera never saw. Strung together, they turn an entire game's worth of flat backgrounds into rough 3D scenes automatically.

What you get is a faithful reconstruction of what was *visible* and an informed guess about everything else. For a little VR parallax in Alexandria, that's already magic. For a full 3D remake, it's the best starting point fans have ever had.

## References

[^1]: [An Adventure in Pre-Rendered Backgrounds](https://jmeiners.com/pre-rendered-backgrounds), Justin Meiners and Hunter Rasmussen.
[^2]: [How Pre-Rendered Backgrounds Defined Early 3D Gaming](https://www.pulse.bot/gaming/video/how-pre-rendered-backgrounds-defined-early-3d-gaming-40b260b9-5da3-402b-9dc2-8b3930e68832), Modern Vintage Gamer via Pulse, April 2026.
[^3]: [FF7 Field](https://ff7-mods.github.io/ff7-flat-wiki/FF7/Field), FF7 Flat Wiki; and [Field: Camera Matrix](https://ff7-mods.github.io/ff7-flat-wiki/FF7/Field/Camera_Matrix).
[^4]: [Moguri Mod](https://en.wikipedia.org/wiki/Moguri_Mod), Wikipedia.
[^5]: [Onimusha: Warlords Xbox One Review](https://darkzero.co.uk/?p=100610), DarkZero.
[^6]: [Onimusha: Warlords Remastered Review](https://primagames.com/featured/onimusha-warlords-remastered-review), Prima Games.
[^7]: [A fan has remastered the first Onimusha into 4K](https://www.gamepressure.com/newsroom/a-fan-has-remastered-the-first-onimusha-into-4k/zf7bb4), Gamepressure.
[^8]: [Onimusha 2: Samurai's Destiny Remaster Brings HD Upgrades](https://finalboss.io/onimusha-2-samurais-destiny-remaster-brings-hd-upgrades), Final Boss.
[^9]: [Moguri Mod](https://www.sites.google.com/view/moguri-mod), official site.
[^10]: [The Final Fantasy 9 upscale mod just got a surprise update after 3 years](https://pcgamer.com/games/rpg/the-final-fantasy-9-upscale-mod-just-got-a-surprise-update-after-3-years-including-a-full-rework-of-its-backgrounds-120-fps-ultrawide-and-more), PC Gamer.
[^11]: [Resident Evil HD Texture Pack](https://www.dsogaming.com/?p=187168), DSOGaming.
[^12]: [Depth Anything V2](https://neurips.cc/virtual/2024/poster/94431), NeurIPS 2024.
[^13]: [Depth Anything 3 for Depth Estimation](https://blog.roboflow.com/depth-anything-3/), Roboflow.
[^14]: [depth-anything-3](https://pypi.org/project/depth-anything-3/0.1.0/), PyPI.
[^15]: [ByteDance-Seed/Depth-Anything-3](https://awesome.ecosyste.ms/projects/github.com%2FByteDance-Seed%2FDepth-Anything-3), repository metadata (Apache 2.0); see also [the project page](https://depth-anything-3.github.io/).
[^16]: [MoGe paper](https://arxiv.org/pdf/2410.19115), Wang et al., arXiv 2410.19115.
[^17]: [MoGe paper (HTML)](https://arxiv.org/html/2410.19115v1), arXiv.
[^18]: [MoGe README](https://huggingface.co/spaces/microsoft/VITRA/blob/main/thirdparty/MoGe/README.md), Microsoft, mirrored on Hugging Face.
[^19]: [microsoft/moge](https://awesome.ecosyste.ms/projects/github.com%2Fmicrosoft%2Fmoge), repository README.
[^20]: [Apple releases impressive open-source model that instantly turns 2D photos into 3D views](https://9to5mac.com/2025/12/17/apple-sharp-ai-model-turns-2d-photos-into-3d-views/), 9to5Mac; paper: [Sharp Monocular View Synthesis in Less Than a Second](https://arxiv.org/abs/2512.10685).
[^21]: [apple/ml-sharp README](https://cdn.jsdelivr.net/gh/apple/ml-sharp@main/README.md), Apple on GitHub; and [Apple Unveils AI Tool That Converts 2D Images Into 3D](https://www.digitalcitizen.life/apple-unveils-ai-tool-that-converts-2d-images-into-3d/), Digital Citizen.
[^22]: [Photos become 3D environments: Try out Apple's new AI model with iPhone](https://heise.de/-11141396), heise online.
[^23]: [Marble by World Labs: The Essentials](https://help.scenario.com/articles/4774055514-marble-by-world-labs-the-essentials), Scenario.
[^24]: [Marble documentation](https://docs.worldlabs.ai/), World Labs; pricing summarised in [World Labs Marble, Explained](https://invideo.io/blog/world-labs-marble-3d-worlds/), invideo.
[^25]: [HunyuanWorld 1.0](https://github.com/tencent-hunyuan/hunyuanworld-1.0), Tencent on GitHub.
[^26]: [Tencent Just Open-Sourced a 3D World Generator](https://www.theneuron.ai/explainer-articles/tencent-just-open-sourced-a-3d-world-generator), The Neuron.
[^27]: [SAM 3D: 3Dfy Anything in Images](https://arxiv.org/pdf/2511.16624), Meta, arXiv 2511.16624; see also [Meta SAM 3D](https://blog.roboflow.com/sam-3d/), Roboflow.
[^28]: [3D Reconstruction Models](https://playground.roboflow.com/models/task/3d-reconstruction), Roboflow.
[^29]: [IMAGE-BLASTER](https://www.worldlabs.ai/image-blaster), World Labs.
[^30]: [Final Fantasy IX](https://datacrystal.tcrf.net/wiki/Final_Fantasy_IX), Data Crystal; and [Steam community discussion on Hades Workshop](https://steamcommunity.com/app/377840/discussions/0/2291716608424014374).
[^31]: [3D Gaussian Splatting papers](https://huggingface.co/papers?q=3D%20Gaussian), Hugging Face Papers.
[^32]: [REupscale Project Cleans up REmake HD's Backgrounds](https://www.relyonhorror.com/latest-news/reupscale-project-cleans-up-remake-hds-backgrounds), Rely on Horror; and [Ray Tracing shader uncovers hidden background detail in Resident Evil Remastered](https://gamesradar.com/ray-tracing-shader-uncovers-hidden-background-detail-in-resident-evil-remastered), GamesRadar.
