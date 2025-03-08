---
sidebar_position: 3
---

# ResNet50 完整示例

此文档介绍如何使用 CIX P1 NPU SDK 将 [ResNet50](https://github.com/onnx/models/blob/main/validated/vision/classification/resnet/model/resnet50-v1-12.onnx) 转换为 CIX SOC NPU 上可以运行的模型。

整体来讲有四个步骤：
:::tip
步骤1~3 在 x86 主机 Linux 环境下执行
:::
1. 下载 NPU SDK 并安装 NOE Compiler
2. 下载模型文件 (代码和脚本)
3. 编译模型
4. 部署模型到 Orion O6

## 下载 NPU SDK 并安装 NOE Compiler

请参考 [安装 NPU SDK](./npu-introduction#npu-sdk-安装) 进行 NPU SDK 和 NOE Compiler 的安装.

## 下载模型文件

在 CIX AI Model Hub 中包含了 ResNet50 的所需文件， 请用户按照 [下载 CIX AI Model Hub](./ai-hub#下载-cix-ai-model-hub) 下载，然后到对应的目录下查看

```bash
cd ai_model_hub/models/ComputeVision/Image_Classification/onnx_resnet_v1_50
```
请确认目录结构是否同下图所示。

```bash
.
├── cfg
│   └── onnx_resnet_v1_50build.cfg
├── datasets
│   └── calib_data.npy
├── graph.json
├── inference_npu.py
├── inference_onnx.py
├── ReadMe.md
├── test_data
│   ├── ILSVRC2012_val_00002899.JPEG
│   ├── ILSVRC2012_val_00004704.JPEG
│   ├── ILSVRC2012_val_00021564.JPEG
│   ├── ILSVRC2012_val_00024154.JPEG
│   ├── ILSVRC2012_val_00037133.JPEG
│   └── ILSVRC2012_val_00045790.JPEG
└── Tutorials.ipynb
```

## 编译模型

:::tip
用户可无需从头编译模型，radxa 提供预编译好的 resnet_v1_50.cix 模型（可用下面步骤下载），如果使用预编译好的模型，可以跳过“编译模型” 这一步
```bash
wget https://modelscope.cn/models/cix/ai_model_hub_24_Q4/resolve/master/models/ComputeVision/Image_Classification/onnx_resnet_v1_50/resnet_v1_50.cix
```
:::

### 准备 onnx 模型

- 下载 onnx 模型

  [resnet50-v1-12.onnx](https://github.com/onnx/models/blob/main/validated/vision/classification/resnet/model/resnet50-v1-12.onnx)

- 简化模型

  这里使用 onnxsim 进行模型输入固化和模型简化

  ```bash
  pip3 install onnxsim onnxruntime
  onnxsim resnet50-v1-12.onnx resnet50-v1-12-sim.onnx --overwrite-input-shape 1,3,224,224
  ```

### 编译模型

CIX SOC NPU 支持 INT8 计算，在编译模型前，我们需要使用 NOE Compiler 对模型进行 INT8 量化

- 准备校准集

  - 自行准备校准集

    在 `test_data` 目录下已经包含多张校准集的图片文件

    ```bash
    ├── test_data
    │   ├── ILSVRC2012_val_00002899.JPEG
    │   ├── ILSVRC2012_val_00004704.JPEG
    │   ├── ILSVRC2012_val_00021564.JPEG
    │   ├── ILSVRC2012_val_00024154.JPEG
    │   ├── ILSVRC2012_val_00037133.JPEG
    │   └── ILSVRC2012_val_00045790.JPEG
    ```

    参考以下脚本生成校准文件

    ```python
    import sys
    import os
    import numpy as np
    _abs_path = os.path.join(os.getcwd(), "../../../../")
    sys.path.append(_abs_path)
    from utils.image_process import imagenet_preprocess_method1

    from utils.tools import get_file_list
    # Get a list of images from the provided path
    images_path = "test_data"
    images_list = get_file_list(images_path)
    data = []
    for image_path in images_list:
        input = imagenet_preprocess_method1(image_path)
        data.append(input)
    # concat the data and save calib dataset
    data = np.concatenate(data, axis=0)
    print(data.shape)
    np.save("datasets/calib_data_tmp.npy", data)
    print("Generate calib dataset success.")
    ```

- 使用 NOE Compiler 量化与编译模型

  - 制作量化 cfg 配置文件, 请参考以下配置

    ```bash
    [Common]
    mode = build

    [Parser]
    model_type = onnx
    model_name = resnet_v1_50
    detection_postprocess =
    model_domain = image_classification
    input_model = ./resnet50-v1-12-sim.onnx
    output_dir = ./
    input_shape = [1, 3, 224, 224]
    input = data

    [Optimizer]
    output_dir = ./
    calibration_data = datasets/calib_data_tmp.npy
    calibration_batch_size = 16
    dataset = numpydataset
    save_statistic_info = True
    cast_dtypes_for_lib = True
    global_calibration = adaround[10, 10, 32, 0.01]

    [GBuilder]
    target = X2_1204MP3
    outputs = resnet_v1_50.cix
    tiling = fps
    profile = True
    ```

  - 量化模型
    :::tip
    如果遇到 cixbuild 报错 `[E] Optimizing model failed! CUDA error: no kernel image is available for execution on the device ...`
    这意味着当前版本的 torch 不支持此 GPU，请完全卸载当前版本的 torch, 然后在 torch 官网下载最新版本。
    :::
    ```bash
    cixbuild ./onnx_resnet_v1_50build.cfg
    ```

## 模型部署

### NPU 推理

将使用 NOE Compiler 编译好的 .cix 格式的模型复制到 Orion O6 开发板上进行模型验证

```bash
python3 inference_npu.py --images test_data --model_path ./resnet_v1_50.cix
```

```bash
(.venv) radxa@orion-o6:~/NOE/ai_model_hub/models/ComputeVision/Image_Classification/onnx_resnet_v1_50$ time python3 inference_npu.py --images test_data --model_path ./resnet_v1_50.cix
npu: noe_init_context success
npu: noe_load_graph success
Input tensor count is 1.
Output tensor count is 1.
npu: noe_create_job success
image path : test_data/ILSVRC2012_val_00004704.JPEG
plunger, plumber's helper
image path : test_data/ILSVRC2012_val_00021564.JPEG
coucal
image path : test_data/ILSVRC2012_val_00024154.JPEG
Ibizan hound, Ibizan Podenco
image path : test_data/ILSVRC2012_val_00037133.JPEG
ice bear, polar bear, Ursus Maritimus, Thalarctos maritimus
image path : test_data/ILSVRC2012_val_00002899.JPEG
rock python, rock snake, Python sebae
image path : test_data/ILSVRC2012_val_00045790.JPEG
Yorkshire terrier
npu: noe_clean_job success
npu: noe_unload_graph success
npu: noe_deinit_context success

real	0m2.963s
user	0m3.266s
sys	0m0.414s
```

### CPU 推理

使用 CPU 对 onnx 模型进行推理验证正确性，可在 X86 主机上或 Orion O6 上运行

```bash
python3 inference_onnx.py --images test_data --onnx_path ./resnet50-v1-12-sim.onnx
```

```bash
(.venv) radxa@orion-o6:~/NOE/ai_model_hub/models/ComputeVision/Image_Classification/onnx_resnet_v1_50$ time python3 inference_onnx.py --images test_data --onnx_path ./resnet50-v1-12-sim.onnx
image path : test_data/ILSVRC2012_val_00004704.JPEG
plunger, plumber's helper
image path : test_data/ILSVRC2012_val_00021564.JPEG
coucal
image path : test_data/ILSVRC2012_val_00024154.JPEG
Ibizan hound, Ibizan Podenco
image path : test_data/ILSVRC2012_val_00037133.JPEG
ice bear, polar bear, Ursus Maritimus, Thalarctos maritimus
image path : test_data/ILSVRC2012_val_00002899.JPEG
rock python, rock snake, Python sebae
image path : test_data/ILSVRC2012_val_00045790.JPEG
Yorkshire terrier

real	0m3.757s
user	0m11.789s
sys	0m0.396s
```

可以看到 NPU 和 CPU 上推理的结果一致,但运行速度缩短

## 参考文档

论文链接： [Deep Residual Learning for Image Recognition](https://arxiv.org/abs/1512.03385)
