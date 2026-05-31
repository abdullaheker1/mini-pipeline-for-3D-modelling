🌐 **Language / Dil:** [English](README.md) · [Türkçe](README.tr.md)

---

# 🧊 Image → 3D Model Pipeline
### Nano Banana 2 → Hunyuan3D 2.x → Ready-to-Use Mesh

A reproducible mini-pipeline for generating game/sim-ready 3D assets from a single text prompt.  
Based on findings from the **Hunyuan3D 2.0** (arxiv:2501.12202) and **2.1** (arxiv:2506.15442) papers.

---

## Pipeline Overview

```
[Text Prompt]
     │
     ▼
┌─────────────────────┐
│  Nano Banana 2      │  ← Image generation
│  (image gen)        │     (Shape-Optimal or Texture-Optimal prompt)
└────────┬────────────┘
         │  .PNG / .WEBP (white bg, centered, square)
         ▼
┌─────────────────────┐
│  Hunyuan3D 2.x      │  ← Single image → 3D
│  DiT + Paint        │     Shape (mesh) then Texture (PBR maps)
└────────┬────────────┘
         │  .GLB / .OBJ + albedo/metallic/roughness maps
         ▼
┌─────────────────────┐
│  Use / Export       │  ← Blender, Unity, ROS, sim
└─────────────────────┘
```

---

## Step 1 — Generate the Input Image

The Hunyuan3D papers explicitly document what input characteristics yield best results.  
These two prompt templates are designed to match the model's training distribution.

### Why image quality matters

The model uses **DINOv2 Giant** (518×518 px) as its condition encoder.  
Training condition images were rendered from 3D datasets with:
- **White backgrounds**, object centered and normalized to unit cube
- **Random camera elevation: -30° to +70°** (most samples near 0°–45°)
- **Stochastic lighting**: 70% HDR maps, 30% point lights — all diffuse, no harsh specular
- A dedicated **image delighting module** pre-processes inputs to remove lighting before texture synthesis

> Providing a clean, shadow-free, white-background image essentially skips the error-prone preprocessing step and feeds the model a distribution-matched input.

---

### Prompt Template A — Shape-Optimal
*Best for: sharp geometry, detailed edges, correct topology*

```
A [OBJECT] photographed as a product hero shot.
Pure white background (#FFFFFF), seamlessly clean.
Object centered, filling 75% of the square frame.
Slight three-quarter perspective view, camera elevated 20-30 degrees above horizon.
Soft even studio lighting, no harsh shadows, no specular highlights, no reflections.
All surface details and edges clearly visible.
Physically accurate materials, not stylized or illustrated.
No depth of field blur. Sharp focus across entire object.
Square aspect ratio, 1:1.
```

### Prompt Template B — Texture-Optimal
*Best for: accurate PBR materials, correct albedo colors, metallic/roughness maps*

```
A [OBJECT] on a pure white background, product photography style.
Centered composition, object occupying 70-80% of the frame.
Flat diffuse lighting from front-top (45 degrees), no cast shadows,
no environment reflections, no colored light spill.
True material colors visible: [MATERIAL — e.g. matte ceramic, brushed aluminum, weathered wood].
No post-processing, no vignette, no color grading.
All visible surfaces evenly illuminated.
Sharp edges, clean silhouette. Square crop.
Photorealistic render style.
```

**Replace:** `[OBJECT]` with the item name. `[MATERIAL]` with surface description.

---

### Input Image Checklist

| Parameter | Target | Reason |
|-----------|--------|--------|
| Background | Pure white `#FFFFFF` | Matches model's internal normalizer + training renders |
| Object centering | Centered, ~75% frame fill | Normalizer maps object to unit cube — more fill = more tokens on detail |
| Camera elevation | 15–35° above horizon | Peak training distribution; avoids top-down ambiguity |
| Lighting | Flat, soft, diffuse | Delighting module works best with low-contrast illumination inputs |
| FoV / perspective | Slight 3/4 perspective | Training used random FoV 10°–70°; avoid fully orthographic |
| Focus | Full-object sharp | DINOv2 feature extraction degrades with DoF blur |
| Aspect ratio | 1:1 square | Model crops to 518×518 internally |

---

## Step 2 — Run Hunyuan3D 2.x

> [!TIP]
> **No GPU? No problem.** You can try Hunyuan3D directly in your browser via the official demo at [3d.hunyuan.tencent.com](https://3d.hunyuan.tencent.com) — no installation required. Local setup below is for batch use, scripting, or offline workflows.

### Installation

```bash
# Clone the repo
git clone https://github.com/Tencent-Hunyuan/Hunyuan3D-2.1
cd Hunyuan3D-2.1

# Install dependencies
pip install -r requirements.txt

# Download model weights (auto on first run via HuggingFace)
# or manually: tencent/Hunyuan3D-2.1 on HuggingFace
```

### Run (Gradio UI)

```bash
python3 gradio_app.py
# Open http://localhost:7860
```

### Run (Python API)

```python
from hy3dgen.shapegen import Hunyuan3DDiTFlowMatchingPipeline
from hy3dgen.texgen import Hunyuan3DPaintPipeline
from PIL import Image

# Load pipelines
shape_pipeline = Hunyuan3DDiTFlowMatchingPipeline.from_pretrained(
    'tencent/Hunyuan3D-2.1'
)
tex_pipeline = Hunyuan3DPaintPipeline.from_pretrained(
    'tencent/Hunyuan3D-2.1'
)

# Load your white-bg image
image = Image.open('your_object.png').convert('RGBA')

# Stage 1: Shape generation
mesh = shape_pipeline(image=image, num_inference_steps=50)[0]

# Stage 2: Texture synthesis (PBR)
textured_mesh = tex_pipeline(mesh, image=image)[0]

# Export
textured_mesh.export('output.glb')
```

### Key Parameters

| Parameter | Default | Notes |
|-----------|---------|-------|
| `num_inference_steps` | 50 | 25 works for quick iterations, 50+ for final |
| `guidance_scale` | 5.0 | Higher = more faithful to input image |
| `octree_resolution` | 380 | Mesh detail level — increase for complex objects |

---

## Step 3 — Use the Model

### Export formats
- **`.glb`** — for web, Three.js, ROS visualization
- **`.obj` + `.mtl`** — for Blender, game engines
- **PBR maps**: albedo, metallic, roughness — drop into any PBR renderer

### Blender
```python
# Blender Python — import generated GLB
import bpy
bpy.ops.import_scene.gltf(filepath='output.glb')
```

### ROS / Gazebo
```bash
# Convert GLB to DAE for URDF
blender output.glb --background --python convert_to_dae.py
```

### Three.js
```javascript
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
const loader = new GLTFLoader();
loader.load('output.glb', (gltf) => scene.add(gltf.scene));
```

---

## Example: Mechanical Keyboard Keycap

### Prompt used (Template A)
```
A single mechanical keyboard keycap photographed as a product hero shot.
Pure white background (#FFFFFF), seamlessly clean.
Object centered, filling 75% of the square frame.
Slight three-quarter perspective view, camera elevated 25 degrees above horizon.
Soft even studio lighting, no harsh shadows, no specular highlights, no reflections.
Matte black ABS plastic surface, PBT legend in white, sculpted SA profile.
Sharp focus across entire object. Square aspect ratio, 1:1.
```

### Results
| Stage | Output |
|-------|--------|
| Input image | 1024×1024 white-bg render from Nano Banana 2 |
| Shape (DiT) | ~8k triangle mesh, correct SA curvature |
| Texture (Paint) | albedo + metallic(0) + roughness(0.85) maps |
| Export | `.glb` ~2.4 MB |

---

## Notes & Limitations

- Objects with **internal cavities** (hollow shapes) may reconstruct as solid — the model infers from a single view.
- **Transparent/glass materials** are poorly reconstructed due to ambiguity in single-image geometry.
- Best results are **product-like objects with clear silhouettes**: tools, furniture, vehicles, props.
- For characters/creatures: use Template A with a neutral A-pose front view.

---

## References

- Hunyuan3D 2.0: [arxiv:2501.12202](https://arxiv.org/abs/2501.12202)
- Hunyuan3D 2.1: [arxiv:2506.15442](https://arxiv.org/abs/2506.15442)
- GitHub: [Tencent-Hunyuan/Hunyuan3D-2.1](https://github.com/Tencent-Hunyuan/Hunyuan3D-2.1)

---

*Pipeline by [@abdullaheker1](https://github.com/abdullaheker1) · ASABEES / ESTÜ Robotics & AI*
