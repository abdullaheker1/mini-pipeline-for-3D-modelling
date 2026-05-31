🌐 **Dil / Language:** [Türkçe](README.tr.md) · [English](README.md)

---

# 🧊 Görsel → 3D Model Pipeline'ı
### Nano Banana 2 → Hunyuan3D 2.x → Kullanıma Hazır Mesh

Tek bir metin prompt'undan oyun/simülasyon hazır 3D asset üretmek için tekrarlanabilir bir mini pipeline.  
**Hunyuan3D 2.0** (arxiv:2501.12202) ve **2.1** (arxiv:2506.15442) makalelerindeki bulgulara dayanmaktadır.

---

## Pipeline Genel Bakış

```
[Metin Prompt'u]
     │
     ▼
┌─────────────────────┐
│  Nano Banana 2      │  ← Görsel üretimi
│  (image gen)        │     (Geometri- veya Texture-Odaklı prompt)
└────────┬────────────┘
         │  .PNG / .WEBP (beyaz arka plan, ortalı, kare)
         ▼
┌─────────────────────┐
│  Hunyuan3D 2.x      │  ← Tek görsel → 3D
│  DiT + Paint        │     Önce Shape (mesh), sonra Texture (PBR haritaları)
└────────┬────────────┘
         │  .GLB / .OBJ + albedo/metallic/roughness haritaları
         ▼
┌─────────────────────┐
│  Kullan / Dışa Aktar│  ← Blender, Unity, ROS, simülasyon
└─────────────────────┘
```

---

## Adım 1 — Girdi Görselini Üret

Hunyuan3D makaleleri, hangi girdi özelliklerinin en iyi sonucu verdiğini açıkça belgeliyor.  
Bu iki prompt şablonu, modelin eğitim dağılımıyla örtüşecek şekilde tasarlanmıştır.

### Görsel kalitesi neden önemli?

Model, koşul encoder'ı olarak **DINOv2 Giant** kullanır (518×518 px).  
Eğitim koşul görselleri 3D veri setlerinden şu parametrelerle render edilmiştir:
- **Beyaz arka plan**, nesne ortalanmış ve birim küpe normalize edilmiş
- **Rastgele kamera yükseklik açısı: -30° ile +70°** (örneklerin büyük çoğunluğu 0°–45° aralığında)
- **Stokastik aydınlatma**: %70 HDR haritaları, %30 nokta ışıkları — difüz, sert specular yok
- Texture sentezinden önce aydınlatmayı gideren özel bir **image delighting modülü**

> Temiz, gölgesiz, beyaz arka planlı bir görsel sağlamak; modelin kendi preprocessing adımını neredeyse atlatmak ve dağılımla örtüşen bir girdi vermek anlamına gelir.

---

### Prompt Şablonu A — Geometri Odaklı
*En iyi sonuç: keskin geometri, detaylı kenarlar, doğru topoloji*

```
A [NESNE] photographed as a product hero shot.
Pure white background (#FFFFFF), seamlessly clean.
Object centered, filling 75% of the square frame.
Slight three-quarter perspective view, camera elevated 20-30 degrees above horizon.
Soft even studio lighting, no harsh shadows, no specular highlights, no reflections.
All surface details and edges clearly visible.
Physically accurate materials, not stylized or illustrated.
No depth of field blur. Sharp focus across entire object.
Square aspect ratio, 1:1.
```

### Prompt Şablonu B — Texture Odaklı
*En iyi sonuç: doğru PBR malzeme, albedo renkleri, metallic/roughness haritaları*

```
A [NESNE] on a pure white background, product photography style.
Centered composition, object occupying 70-80% of the frame.
Flat diffuse lighting from front-top (45 degrees), no cast shadows,
no environment reflections, no colored light spill.
True material colors visible: [MALZEME — örn. mat seramik, fırçalanmış alüminyum, eskimiş ahşap].
No post-processing, no vignette, no color grading.
All visible surfaces evenly illuminated.
Sharp edges, clean silhouette. Square crop.
Photorealistic render style.
```

**Değiştir:** `[NESNE]` yerine nesne adını, `[MALZEME]` yerine yüzey özelliğini yaz.  
*Not: Prompt İngilizce kalmalı — Nano Banana 2 İngilizce prompt'larla eğitilmiştir.*

---

### Girdi Görseli Kontrol Listesi

| Parametre | Hedef Değer | Neden? |
|-----------|-------------|--------|
| Arka plan | Düz beyaz `#FFFFFF` | Modelin iç normalizer'ı ve eğitim render'larıyla birebir örtüşür |
| Nesne ortalama | Ortalı, çerçevenin ~%75'ini kaplayan | Normalizer nesneyi birim küpe eşler — yüksek doluluk = detay token'larına daha fazla alan |
| Kamera yüksekliği | Ufuk çizgisinin 15–35° üstü | Eğitim dağılımının tepe bölgesi; tepeden bakış açıklığını ortadan kaldırır |
| Aydınlatma | Flat, yumuşak, difüz | Delighting modülü düşük kontrastlı girişlerde en verimli çalışır |
| FoV / perspektif | Hafif 3/4 perspektif | Eğitim FoV aralığı 10°–70°; tam ortografik perspektiften kaçın |
| Odak | Tüm nesne keskin | DoF bulanıklığı DINOv2 feature extraction'ı bozar |
| En-boy oranı | 1:1 kare kırpma | Model dahili olarak 518×518'e kırpar |

---

## Adım 2 — Hunyuan3D 2.x'i Çalıştır

### Kurulum

```bash
# Repoyu klonla
git clone https://github.com/Tencent-Hunyuan/Hunyuan3D-2.1
cd Hunyuan3D-2.1

# Bağımlılıkları yükle
pip install -r requirements.txt

# Model ağırlıkları ilk çalıştırmada HuggingFace üzerinden otomatik indirilir
# Ya da manuel: tencent/Hunyuan3D-2.1 (HuggingFace)
```

### Çalıştır (Gradio Arayüzü)

```bash
python3 gradio_app.py
# Tarayıcıda aç: http://localhost:7860
```

### Çalıştır (Python API)

```python
from hy3dgen.shapegen import Hunyuan3DDiTFlowMatchingPipeline
from hy3dgen.texgen import Hunyuan3DPaintPipeline
from PIL import Image

# Pipeline'ları yükle
shape_pipeline = Hunyuan3DDiTFlowMatchingPipeline.from_pretrained(
    'tencent/Hunyuan3D-2.1'
)
tex_pipeline = Hunyuan3DPaintPipeline.from_pretrained(
    'tencent/Hunyuan3D-2.1'
)

# Beyaz arka planlı görselini yükle
image = Image.open('nesne.png').convert('RGBA')

# Aşama 1: Shape (geometri) üretimi
mesh = shape_pipeline(image=image, num_inference_steps=50)[0]

# Aşama 2: Texture sentezi (PBR)
textured_mesh = tex_pipeline(mesh, image=image)[0]

# Dışa aktar
textured_mesh.export('cikti.glb')
```

### Temel Parametreler

| Parametre | Varsayılan | Açıklama |
|-----------|------------|----------|
| `num_inference_steps` | 50 | Hızlı iterasyon için 25, final çıktı için 50+ |
| `guidance_scale` | 5.0 | Yüksekse girdi görseline daha sadık kalır |
| `octree_resolution` | 380 | Mesh detay seviyesi — karmaşık nesneler için artır |

---

## Adım 3 — Modeli Kullan

### Dışa aktarma formatları
- **`.glb`** — web, Three.js, ROS görselleştirme
- **`.obj` + `.mtl`** — Blender, oyun motorları
- **PBR haritaları**: albedo, metallic, roughness — her PBR renderer'a aktarılabilir

### Blender
```python
# Blender Python — üretilen GLB'yi içe aktar
import bpy
bpy.ops.import_scene.gltf(filepath='cikti.glb')
```

### ROS / Gazebo
```bash
# URDF için GLB → DAE dönüşümü
blender cikti.glb --background --python convert_to_dae.py
```

### Three.js
```javascript
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
const loader = new GLTFLoader();
loader.load('cikti.glb', (gltf) => scene.add(gltf.scene));
```

---

## Örnek: Mekanik Klavye Tuşu

### Kullanılan prompt (Şablon A)
```
A single mechanical keyboard keycap photographed as a product hero shot.
Pure white background (#FFFFFF), seamlessly clean.
Object centered, filling 75% of the square frame.
Slight three-quarter perspective view, camera elevated 25 degrees above horizon.
Soft even studio lighting, no harsh shadows, no specular highlights, no reflections.
Matte black ABS plastic surface, PBT legend in white, sculpted SA profile.
Sharp focus across entire object. Square aspect ratio, 1:1.
```

### Sonuçlar
| Aşama | Çıktı |
|-------|-------|
| Girdi görseli | Nano Banana 2'den 1024×1024 beyaz arka planlı render |
| Shape (DiT) | ~8k üçgen mesh, doğru SA eğrisi |
| Texture (Paint) | albedo + metallic(0) + roughness(0.85) haritaları |
| Dışa aktarma | `.glb` ~2.4 MB |

---

## Notlar ve Sınırlılıklar

- **İç boşluklu nesneler** (içi boş şekiller) dolu olarak rekonstrükte edilebilir — model tek görüşten çıkarım yapar.
- **Şeffaf/cam malzemeler** geometri belirsizliği nedeniyle zayıf sonuç verir.
- En iyi sonuçlar **net siluete sahip ürün benzeri nesnelerde**: alet, mobilya, araç, prop.
- Karakter/yaratık için: Şablon A'yı nötr A-pose ön görünümüyle kullan.

---

## Kaynaklar

- Hunyuan3D 2.0: [arxiv:2501.12202](https://arxiv.org/abs/2501.12202)
- Hunyuan3D 2.1: [arxiv:2506.15442](https://arxiv.org/abs/2506.15442)
- GitHub: [Tencent-Hunyuan/Hunyuan3D-2.1](https://github.com/Tencent-Hunyuan/Hunyuan3D-2.1)

---

*Pipeline: [@shekerFND](https://github.com/shekerFND) · ASABEES / ESTÜ Robotik & Yapay Zeka*
