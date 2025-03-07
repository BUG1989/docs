---
sidebar_position: 6
---

# DeepLabv3 完整示例

此文档使用 CIX P1 NPU SDK 将 [deeplabv3](https://pytorch.org/vision/main/models/generated/torchvision.models.segmentation.deeplabv3_resnet50.html) 移植到 CIX SOC 内部的硬件加速模块实现使用 NPU 推理神经网络模型，
请在参考此文档前请先在 X86 工作站按照 [安装 NPU SDK](./npu-introduction#安装-npu-sdk) 安装 NOE Compiler，在 X86 工作站与 Orion O6 按照 [下载 CIX AI Model Hub](./ai-hub#下载-cix-ai-model-hub) 文档配置所需环境。

论文链接： [Rethinking Atrous Convolution for Semantic Image Segmentation](https://arxiv.org/abs/1706.05587)

## DeepLabv3 工程目录列表

在 CIX AI Model Hub 中包含了 DeepLabv3 的所需文件， 请用户按照 [下载 CIX AI Model Hub](./ai-hub#下载-cix-ai-model-hub) 下载

```bash
cd ai_model_hub/models/ComputeVision/Semantic_Segmentation/onnx_deeplab_v3
```

```bash
.
├── cfg
│   └── onnx_deeplab_v3_build.cfg
├── datasets
│   └── calibration_data.npy
├── graph.json
├── inference_npu.py
├── inference_onnx.py
├── ReadMe.md
├── test_data
│   └── ILSVRC2012_val_00004704.JPEG
└── Tutorials.ipynb
```

## （可选）下载 CIX 模型

用户可无需从头编译模型，radxa 提供下载预编译好的 deeplab_v3.cix 模型方法

- 下载模型
  ```bash
  wget https://modelscope.cn/models/cix/ai_model_hub_24_Q4/resolve/master/models/ComputeVision/Semantic_Segmentation/onnx_deeplab_v3/deeplab_v3.cix
  ```

## 编译模型

### 准备 onnx 模型

- 下载 onnx 模型

  [deeplabv3_resnet50.onnx](https://modelscope.cn/models/cix/ai_model_hub_24_Q4/resolve/master/models/ComputeVision/Semantic_Segmentation/onnx_deeplab_v3/model/deeplabv3_resnet50.onnx)

- 简化模型

  这里使用 onnxsim 进行模型输入固化和模型简化

  ```bash
  pip3 install onnxsim onnxruntime
  onnxsim deeplabv3_resnet50.onnx deeplabv3_resnet50-sim.onnx --overwrite-input-shape 1,3,520,520
  ```

### 编译模型

CIX SOC NPU 支持 INT8 计算，在编译模型前，我们需要使用 NOE Compiler 对模型进行 INT8 量化

- 准备校准集

  - 使用 `datasets` 现有校准集

    ```bash
    .
    └── calibration_data.npy
    ```

  - 自行准备校准集

    在 `test_data` 目录下已经包含多张校准集的图片文件

    ```bash
    .
    ├── 1.jpeg
    └── 2.jpeg
    ```

    参考以下脚本生成校准文件

    ```python
    import sys
    import os
    import numpy as np
    _abs_path = os.path.join(os.getcwd(), "../../../../")
    sys.path.append(_abs_path)
    from utils.image_process import preprocess_image_deeplabv3
    from utils.tools import get_file_list
    # Get a list of images from the provided path
    images_path = "test_data"
    images_list = get_file_list(images_path)
    data = []
    for image_path in images_list:
        input = preprocess_image_deeplabv3(image_path)
        data.append(input)
    # concat the data and save calib dataset
    data = np.concatenate(data, axis=0)
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
    model_name = deeplab_v3
    detection_postprocess =
    model_domain = image_segmentation
    input_model = ./deeplabv3_resnet50-sim.onnx
    input = input
    input_shape = [1, 3, 520, 520]
    output = output
    output_dir = ./

    [Optimizer]
    output_dir = ./
    calibration_data = ./datasets/calib_data_tmp.npy
    calibration_batch_size = 1
    metric_batch_size = 1
    dataset = NumpyDataset
    quantize_method_for_weight = per_channel_symmetric_restricted_range
    quantize_method_for_activation = per_tensor_asymmetric
    save_statistic_info = True

    [GBuilder]
    outputs = deeplab_v3.cix
    target = X2_1204MP3
    profile = True
    tiling = fps
    ```

  - 量化模型
    :::tip
    如果遇到 cixbuild 报错 `[E] Optimizing model failed! CUDA error: no kernel image is available for execution on the device ...`
    这意味着当前版本的 torch 不支持此 GPU，请完全卸载当前版本的 torch, 然后在 torch 官网下载最新版本。
    :::
    ```bash
    cixbuild ./onnx_deeplab_v3_build.cfg
    ```

## 板端部署

### NPU 推理

将使用 NOE Compiler 编译好的 .cix 格式的模型复制到 Orion O6 开发板上进行模型验证

```bash
python3 inference_npu.py --images ./test_data/ --model_path ./deeplab_v3.ci
```

```bash
(.venv) radxa@orion-o6:~/NOE/ai_model_hub/models/ComputeVision/Semantic_Segmentation/onnx_deeplab_v3$ time python3 inference_npu.py --images ./test_data/ --model_path ./deeplab_v3.cix
npu: noe_init_context success
npu: noe_load_graph success
Input tensor count is 1.
Output tensor count is 1.
npu: noe_create_job success
save output: noe_ILSVRC2012_val_00004704.JPEG
npu: noe_clean_job success
npu: noe_unload_graph success
npu: noe_deinit_context success

real	0m9.047s
user	0m4.314s
sys	0m0.478s
```

结果保存在 `output` 文件夹中

![deeplab1.webp](/img/o6/deeplab1.webp)

### CPU 推理

使用 CPU 对 onnx 模型进行推理验证正确性，可在 X86 工作站或 O6 上运行

```bash
python3 inference_onnx.py --images ./test_data/ --onnx_path ./deeplabv3_resnet50-sim.onnx
```

```bash
(.venv) radxa@orion-o6:~/NOE/ai_model_hub/models/ComputeVision/Semantic_Segmentation/onnx_deeplab_v3$ time python3 inference_onnx.py --images ./test_data/ --onnx_path ./deeplabv3_resnet50-sim.onnx
save output: onnx_ILSVRC2012_val_00004704.JPEG

real	0m7.605s
user	0m33.235s
sys	0m0.558s

```

结果保存在 `output` 文件夹中
![deeplab2.webp](/img/o6/deeplab2.webp)

可以看到 NPU 和 CPU 上推理的结果一致,但运行速度大幅缩短
