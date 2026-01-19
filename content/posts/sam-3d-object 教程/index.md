---
title: "SAM 3D Objects 教程"
date: 2026-01-15
draft: false
topics: ["三维重建"]
---

## 安装

参考[官方安装教程](https://github.com/facebookresearch/sam-3d-objects/blob/main/doc/setup.md)在服务器上配置环境，服务器全程使用 clash 代理，使用 conda 安装

### 环境配置

```shell
# 克隆项目仓库
git clone https://github.com/facebookresearch/sam-3d-objects.git
cd sam-3d-objects

# 创建 conda 环境
conda env create -f environments/default.yml
conda activate sam3d-objects

# 设置 pytorch 等安装的 URL
export PIP_EXTRA_INDEX_URL="https://pypi.ngc.nvidia.com https://download.pytorch.org/whl/cu121"

# 安装 sam 3d 核心依赖
pip install -e '.[dev]'
pip install -e '.[p3d]' # pytorch3d dependency on pytorch is broken, this 2-step approach solves it

# 安装 sam 3d 推理
export PIP_FIND_LINKS="https://nvidia-kaolin.s3.us-east-2.amazonaws.com/torch-2.5.1_cu121.html"
pip install -e '.[inference]'
```

### 下载官方权重文件

首先需要在 huggingface 上申请许可，笔者使用新邮箱创建了一个新的 huggingface 账户，在申请许可需要填入的个人信息中填写美国地址、学校等等，否则容易被拒。获得许可后，执行：

```shell
pip install 'huggingface-hub[cli]<1.0'

TAG=hf
hf download \
  --repo-type model \
  --local-dir checkpoints/${TAG}-download \
  --max-workers 1 \
  facebook/sam-3d-objects
mv checkpoints/${TAG}-download/checkpoints checkpoints/${TAG}
rm -rf checkpoints/${TAG}-download
```

`hf download` 执行后，会要求填入 huggingface 账号的 token，点击右上角个人头像，进入 [access tokens](https://huggingface.co/settings/tokens) 中新建 token，全部打勾提交，即可获得账号 token。复制 token，在服务器命令行中鼠标右键粘贴回车后，就可以开始下载模型文件了。成功下载后，checkpoints 文件保存在 sam-3d-objects/checkpoints/hf 目录。

### 测试 demo

执行 `pythonn demo.py` 成功执行后会生成 `splat.ply` 物体三维重建的高斯点云结果

## mesh

demo 中输出的三维是点云，保存在输出的 `gs` 属性中。下述是[issue 31](https://github.com/facebookresearch/sam-3d-objects/issues/23)中作者给出的模型输出中的各个属性

> Those are all the outputs from our model.
>
> - "scale", "translation", "rotation" are informations about object pose.
> - "pointmap" is the MoGe pointmap used internally by the layout model.
> - "gs", "glb" are the 3D object (gaussian splatting and Trimesh mesh).
>
> To get the object dimensions in the scene, you would need to move the object in the scene reference frame and calculate the 3D bbox of the object using the gaussians xyz coordinates. You can have a look at the [make_scene](https://github.com/facebookresearch/sam-3d-objects/blob/cf066761706cd02b07e2fc6274570ec8cdafb683/notebook/inference.py#L255) function to do the first step.

可见，`glb` 属性是重建的 mesh，[导出代码](https://github.com/facebookresearch/sam-3d-objects/issues/12) 如下：

```py
output = inference(image, mask, seed=42)
mesh = output["glb"]
mesh.export("modle.glb")
```

## texture

在 `notebook/inference.py` 文件中的 `Inference` 类可以更改模型运行的相关设置，如 `with_mesh_postprocess` `with_texture_postprocess` 等（等价于在 `sam3d_objects/pipeline/inference_pipeline.py` 中 `Class InferencePipeline` 的 `def postprocess_slat_output` 中做更详细的修改，[参考](https://gist.github.com/luffy-yu/3c9708aaf446d3640ef843c927ad9952)）。其中，`with_texture_postprocess` 可以产生 texture。

使用 `with_texture_postprocess = True` 时产生了报错 `TypeError: GaussianRasterizationSettings.__new__() got an unexpected keyword argument 'kernel_size'`，[安装相应的 python 包](https://gist.github.com/luffy-yu/3c9708aaf446d3640ef843c927ad9952) 解决了报错

设置 `with_mesh_postprocess` 和 `with_texture_postprocess` 均为 `True`，就可以生成带有 texture 的 mesh 了。（导入 Blender 后要把左上角的 Object Mode 改为 Texture Paint）

## 多视角 sam-3d-object

[项目地址](https://github.com/devinli123/MV-SAM3D)

克隆该项目后发现，`sam3d_objects` 目录并非原版，在 `MV-SAM3D/sam3d_objects/utils` 实现了 `attetion` 机制等进行多视角融合。所以，一开始将原版 `sam3d_objects` 拷贝到这里是不对的，而是应该根据执行 `demo` 产生的报错，向该目录中补齐缺少的部分（如 checkpoints 等等） 。

## 三维重建自己的图片

### 单视角

sam-3d-object 的输入是原始图片和经过 mask 的目标物体图片，所以第一步需要利用某些方法对原图像目标物体做 mask，之后按照 sam-3d-object 的 demo 过程执行即可。[示例代码](https://github.com/facebookresearch/sam-3d-objects/issues/127) 如下：

```py
# 1. Load SAM3 and segment the image
print(f"Loading SAM3 Model and segmenting: '{prompt}'...")
model = Sam3Model.from_pretrained("facebook/sam3").to(device)
processor = Sam3Processor.from_pretrained("facebook/sam3")

image_pil = Image.open(image_path).convert("RGB")
inputs = processor(images=image_pil, text=prompt, return_tensors="pt").to(device)

with torch.no_grad():
    outputs = model(**inputs)

results = processor.post_process_instance_segmentation(
    outputs,
    threshold=0.5,
    mask_threshold=0.5,
    target_sizes=inputs.get("original_sizes").tolist()
)[0]

if len(results['masks']) == 0:
    print(f"No objects matching '{prompt}' were found.")
    return
    
mask_input = results['masks'][0].cpu().numpy()

if mask_input.ndim == 3:
    mask_input = np.squeeze(mask_input)

# Prepare image for SAM3D
image_np = np.array(image_pil).astype(np.uint8)

# 2. Run SAM3D Inference
print("Initializing SAM3D reconstruction...")
config_path = "checkpoints/hf/pipeline.yaml"
inference = Inference(config_path, compile=False)

print("Generating 3D object...")
output = inference(image_np, mask_input, seed=42)

# 3. Save output
output["gs"].save_ply(output_path)
print(f"Success! Reconstruction saved to: {output_path}")
```

（SAM3使用SAM3DObjests的环境就可以运行起来）

### 多视角

1. 多视角图片要经过 [SAM3 批处理](https://huggingface.co/facebook/sam3) 生成对应的mask

2. 使用方法：[参考](https://github.com/devinli123/multi-view-sam-3d-objects)，将该项目中的 [sam3d_objects](https://github.com/devinli123/multi-view-sam-3d-objects/tree/main/sam3d_objects) 替换原目录（其中 [inference_pipeline.py](https://github.com/devinli123/multi-view-sam-3d-objects/blob/main/sam3d_objects/pipeline/inference_pipeline.py)）实现了多视角重建功能

3. 具体代码：

   ```py
   """
   使用 SAM3 和 SAM-3D-Objects 从单张单物品图像和文本提示进行 3D 重建。
   
   需要使用代理。
   """
   
   import argparse
   import os
   from pathlib import Path
   import torch
   from PIL import Image
   import numpy as np
   
   
   device = "cuda" if torch.cuda.is_available() else "cpu"
   
   
   def get_images_and_masks(images_path, prompt):
       """
       使用 SAM3 批处理，根据文本提示生成掩码，
       
       :param images_path: 图像路径
       :param prompt: 文本提示
       """
       from transformers import Sam3Processor, Sam3Model # 可不可以不用 tansformers
       
       print(f"Loading images ...")
       images = load_images(images_path)
       # 1. Load SAM3 and segment the image
       print(f"Loading SAM3 Model and segmenting: '{prompt}'...")
    
       model = Sam3Model.from_pretrained("facebook/sam3").to(device)
       processor = Sam3Processor.from_pretrained("facebook/sam3")
   
       text_prompts = [prompt] * len(images)
       inputs = processor(images=images, text=text_prompts, return_tensors="pt").to(device)
       with torch.no_grad():
           outputs = model(**inputs)
       
       results = processor.post_process_instance_segmentation(
           outputs,
           threshold=0.5,
           mask_threshold=0.5,
           target_sizes=inputs.get("original_sizes").tolist()
       )
   
       masks = []
       for i, result in enumerate(results):
           result = result["masks"]
           if len(result) == 0:
               print(f"No objects matching {prompt} were found in image {i}.")
               exit(1)
           mask = result[0].cpu().numpy()
           if mask.ndim == 3:
               mask = np.squeeze(mask)
           masks.append(mask)
       
       return images, masks
   
   
   def recon(images, masks, output_path):
       """
       使用 SAM-3D-Objects 进行 3D 重建。
       
       :param images: 使用 PIL.Image 打开的图片列表
       :param masks: 掩码列表
       :param output_path: 输出路径
       """
   
       from inference import Inference
   
       config_path = "checkpoints/hf/pipeline.yaml"
       inference = Inference(config_path, compile=False)
   
       result = inference._pipeline.run_multi_view(
               view_images=images,
               view_masks=masks,
               seed=42,
               mode="multidiffusion",
               stage1_inference_steps=50,
               stage2_inference_steps=25,
               decode_formats="gaussian,mesh",
               with_mesh_postprocess=False,
               with_texture_baking=False,
               use_vertex_color=True,
           )
       
   
       print(f"\n{'='*60}")
       print(f"Inference completed!")
       print(f"Generated coordinates: {result['coords'].shape[0] if 'coords' in result else 'N/A'}")
       print(f"{'='*60}")
       saved_files = []
       if 'glb' in result and result['glb'] is not None:
           result['glb'].export(str(output_path / "result.glb"))
           saved_files.append("result.glb")
           print(f"✓ GLB file saved to: {output_path / 'result.glb'}")
   
       if 'gs' in result:
           result['gs'].save_ply(str(output_path / "result.ply"))
           saved_files.append("result.ply")
           print(f"✓ Gaussian Splatting (PLY) saved to: {output_path / 'result.ply'}")
       elif 'gaussian' in result:
           if isinstance(result['gaussian'], list) and len(result['gaussian']) > 0:
               result['gaussian'][0].save_ply(str(output_path / "result.ply"))
               saved_files.append("result.ply")
               print(f"✓ Gaussian Splatting (PLY) saved to: {output_path / 'result.ply'}")
       
       if 'mesh' in result:
           print(f"✓ Mesh information generated (included in GLB)")
       
       print(f"\n{'='*60}")
       print(f"All output files saved to: {output_path}")
       print(f"Saved files: {', '.join(saved_files)}")
       print(f"{'='*60}")
   
       
   def load_images(images_path):
       """
       加载图像。
       
       :param images_path: 图像路径
       :return: 使用 PIL.Image 打开的图片
       """
       images = []
       files = sorted(images_path.glob("*.jpg"))
       if len(files) == 0:
           raise FileNotFoundError(f"No images found in path: {images_path}")
       for file in files:
           image = Image.open(file).convert("RGB")
           images.append(image)
       return images
   
   
   
   def main():
       parser = argparse.ArgumentParser(
           description="Run SAM3D reconstruction from a single image and text prompt."
       )
       parser.add_argument(
           "--images_path",
           type=str,
           default="images/bottle",
           help="Path to the input image directory.",
       )
       parser.add_argument(
           "--prompt",
           type=str,
           default="a bottle",
           help="Text prompt to segment the object of interest.",
       )
   
       args = parser.parse_args()
       
       images_path = Path(args.images_path)
       if not images_path.exists():
           raise FileNotFoundError(f"Input path does not exist: {images_path}")
       if not images_path.is_dir():
           raise ValueError("For multiview reconstruction, images_path should be a directory.")
       # images = load_images(images_path)    
       
       prompt = args.prompt
       output_path = Path(f"outputs/{images_path.stem}/")
       os.makedirs(output_path, exist_ok=True)
       images, masks = get_images_and_masks(images_path, prompt)
   
       recon(images, masks, output_path)
   
   
   if __name__ == "__main__":
       main()
   ```

   



