---
sidebar_label: '基于YOLOv5的目标检测'
sidebar_position: 30
---

# 基于 YOLOv5 的目标检测

## 1. 在 windows 下准备原始模型文件

### 准备 YOLOv5 开发工具包和 yolov5n.pt 文件

下载 [YOLOv5开发工具包](https://codeload.github.com/ultralytics/yolov5/zip/refs/heads/master) 以及 [yolov5n.pt](https://github.com/ultralytics/yolov5/releases/download/v6.2/yolov5n.pt) 文件，下载完成后将工具包解压，并将 `yolov5n.pt` 文件放在 `yolov5-master` 目录下

### 配置 conda 环境

需要提前安装 Anaconda [https://www.anaconda.com/](https://www.anaconda.com/)

新建 `Anaconda Prompt` 终端, 执行 `conda env list` 查看当前环境
```
(base) C:\Users\Carbon> conda env list
# conda environments:
#
base                  *  C:\Users\Carbon\anaconda3
```

新建 conda 虚拟环境并安装 3.9.0 版本的 python，`duotpu` 是自己取的名字
```
(base) C:\Users\Carbon> conda create --name duotpu python=3.9.0
```

成功后再次查看当前环境
```
(base) C:\Users\Carbon> conda env list
# conda environments:
#
base                  *  C:\Users\Carbon\anaconda3
duotpu                   C:\Users\Carbon\anaconda3\envs\duotpu
```

激活刚安装的 3.9.0 的环境
```
(base) C:\Users\Carbon> conda activate duotpu
```

确认已激活
```
(duotpu) C:\Users\Carbon> conda env list
# conda environments:
#
base                     C:\Users\Carbon\anaconda3
duotpu                *  C:\Users\Carbon\anaconda3\envs\duotpu
```

然后可使用如下指令安装 1.12.1 版本的 pytorch，具体安装指令可根据需求选择，后续过程只需要用到 CPU 即可
```
# CUDA 10.2
conda install pytorch==1.12.1 torchvision==0.13.1 torchaudio==0.12.1 cudatoolkit=10.2 -c pytorch

# CUDA 11.3
conda install pytorch==1.12.1 torchvision==0.13.1 torchaudio==0.12.1 cudatoolkit=11.3 -c pytorch

# CUDA 11.6
conda install pytorch==1.12.1 torchvision==0.13.1 torchaudio==0.12.1 cudatoolkit=11.6 -c pytorch

# CPU Only
conda install pytorch==1.12.1 torchvision==0.13.1 torchaudio==0.12.1 cpuonly -c pytorch
```

然后将终端路径`cd`到开发工具包的`yolov5-master`路径下，输入 `pip install -r requirements.txt` 安装其他依赖项
```
(duotpu) C:\Users\Carbon> cd Duo-TPU\yolov5-master

(duotpu) C:\Users\Carbon\Duo-TPU\yolov5-master> pip install -r requirements.txt
```

### 生成原始模型文件

在 `yolov5-master` 目录下新建一个 `main.py` 文件，并在文件中写入如下代码
```
import torch
from models.experimental import attempt_download
model = torch.load(attempt_download("./yolov5n.pt"),
map_location=torch.device('cpu'))['model'].float()
model.eval()
model.model[-1].export = True
torch.jit.trace(model, torch.rand(1, 3, 640, 640), strict=False).save('./yolov5n_jit.pt')
```

然后找到 `yolov5-master/models/yolo.py` 文件，将第99行到第115行的代码注释，并在第116行添加代码 return x`，如下图所示

![duo](/docs/duo/tpu/duo-tpu-yolo5_01.png)

另外这个文件也需要修改一下
```
C:\Users\Carbon\anaconda3\envs\duotpu\lib\python3.9\site-packages\torch\nn\modules\upsampling.py
```
第153行左右做如下修改

![duo](/docs/duo/tpu/duo-tpu-yolo5_02.png)


修改完成后，运行 `python main.py` 文件，就会在 `yolov5-master` 目录下生成 `yolov5n_jit.pt` 文件，该文件即为所需的原始模型文件
```
(duotpu) C:\Users\Carbon\Duo-TPU\yolov5-master> python main.py
```

### 退出 conda 环境 (可选)

上面已经生成了我们需要的模型文件，可以用`conda deactivate`命令退出 conda 环境
```
(duotpu) C:\Users\Carbon\Duo-TPU\yolov5-master> conda deactivate
```

如果不再需要这个 conda 虚拟环境了（duotpu），可以用如下命令删除
```
conda env remove --name <envname>
```

## 2. 配置 Docker 开发环境

参考 [这里](https://milkv.io/zh/docs/duo/application-development/tpu/tpu-docker) 配置好 Docker 开发环境后，再回到这里继续下一步

:::warning
若您使用已经配置好的 Docker 开发环境，请您启动 Docker 后务必按照 Docker 配置教程执行 `source ./tpu-mlir/envsetup.sh` 命令，否则后续步骤可能报错。
:::

## 3. 在 Docker 中准备工作目录

创建并进入 `yolov5n_torch` 工作目录，注意是与 `tpu-mlir` 同级的目录，并将模型文件和图片文件都放入该目录下
```
# mkdir yolov5n_torch && cd yolov5n_torch
```

新建一个 Windows 终端，将`yolov5n_jit.pt`从 windows 拷贝到 Docker 中
```
docker cp <path>/yolov5-master/yolov5n_jit.pt <container_name>:/workspace/yolov5n_torch/yolov5n_jit.pt
```
其中，`<path>`为 windows 系统中 yolov5 开发工具包所在的文件目录，`<container_name>`为容器名，比如
```
docker cp C:\Users\Carbon\Duo-TPU\yolov5-master\yolov5n_jit.pt DuoTPU:/workspace/yolov5n_torch/yolov5n_jit.pt
```

再回到 Docker 终端下，将图片文件放入当前目录(`yolov5n_torch`)下
```
# cp -rf ${TPUC_ROOT}/regression/dataset/COCO2017 .
# cp -rf ${TPUC_ROOT}/regression/image .
```
这里的 `${TPUC_ROOT}` 是环境变量，对应 `tpu-mlir` 目录，是在前面配置 Docker 开发环境中 `source ./tpu-mlir/envsetup.sh` 这一步加载的

创建并进入 `work` 工作目录，用于存放编译生成的 `MLIR`、`cvimodel` 等文件
```
# mkdir work && cd work
```

## 4. YOLOv5n-TORCH 模型转换

:::tip
Duo 开发板搭载的是 CV1800B 芯片，该芯片支持 **ONNX 系列** 和 **Caffe 模型**，目前不支持 TFLite 模型。在量化数据类型方面，支持 **BF16 格式的量化** 和 **INT8 格式的非对称量化**
:::

模型转换步骤如下：
- TORCH 模型转换成 MLIR
- 生成量化需要的校准表
- MLIR 量化成 INT8 非对称 cvimodel

### TORCH 模型转换成 MLIR

本例中，模型是 RGB 输入，`mean`和`scale`分别为 `0,0,0` 和 `0.0039216`,`0.0039216`,`0.0039216`
将 TORCH 模型转换为 MLIR 模型的命令如下
 ```
# model_transform.py \
 --model_name yolov5n \
 --model_def ../yolov5n_jit.pt \
 --input_shapes [[1,3,640,640]] \
 --pixel_format "rgb" \
 --keep_aspect_ratio \
 --mean 0,0,0 \
 --scale 0.0039216,0.0039216,0.0039216 \
 --test_input ../image/dog.jpg \
 --test_result yolov5n_top_outputs.npz \
 --output_names 1219,1234,1249 \
 --mlir yolov5n.mlir
 ```
运行成功效果示例

![duo](/docs/duo/tpu/duo-tpu-yolo5_06.png)

转成 MLIR 模型后，会生成一个 `yolov5n.mlir` 文件，该文件即为 MLIR 模型文件，还会生成一个 `yolov5n_in_f32.npz` 文件和一个 `yolov5n_top_outputs.npz` 文件，是后续转模型的输入文件

![duo](/docs/duo/tpu/duo-tpu-yolo5_07.png)

### MLIR 转 INT8 模型

#### 生成量化需要的校准表

在转 INT8 模型之前需要先生成校准表，这里用现有的 100 张来自 COCO2017 的图片举例，执行 calibration
```
# run_calibration.py yolov5n.mlir \
 --dataset ../COCO2017 \
 --input_num 100 \
 -o ./yolov5n_cali_table
 ```

![duo](/docs/duo/tpu/duo-tpu-yolo5_08.png)

运行完成后，会生成 `yolov5n_cali_table` 文件，该文件用于后续编译 INT8 模型

![duo](/docs/duo/tpu/duo-tpu-yolo5_09.png)

#### MLIR 量化成 INT8 非对称 cvimodel

将 MLIR 模型转换为 INT8 模型的命令如下
```
# model_deploy.py \
 --mlir yolov5n.mlir \
 --quantize INT8 \
 --calibration_table ./yolov5n_cali_table \
 --chip cv180x \
 --test_input ../image/dog.jpg \
 --test_reference yolov5n_top_outputs.npz \
 --compare_all \
 --tolerance 0.96,0.72 \
 --fuse_preprocess \
 --debug \
 --model yolov5n_int8_fuse.cvimodel
 ```

:::tip
如果您使用的开发板不是 Duo ，请将上述命令中第 5 行 `--chip cv180x` 更换为对应的芯片型号。
使用 Duo 256M 时应更改为 `--chip cv181x`。
:::

编译成功效果示例

![duo](/docs/duo/tpu/duo-tpu-yolo5_10.png)

编译完成后，会生成 `yolov5n_int8_fuse.cvimodel` 文件

![duo](/docs/duo/tpu/duo-tpu-yolo5_11.png)

## 5. 在 Duo 开发板上进行验证

### 连接 Duo 开发板

根据前面的教程完成 Duo 开发板与电脑的连接，并使用 `mobaxterm` 或 `Xshell` 等工具开启终端操作 Duo 开发板

### 获取 tpu-sdk

在 Docker 终端下切换到 `/workspace` 目录
```
cd /workspace
```

下载 tpu-sdk，如果您使用的是 Duo ，则执行
```
git clone https://github.com/milkv-duo/tpu-sdk-cv180x.git
mv ./tpu-sdk-cv180x ./tpu-sdk
```

如果您使用的是 Duo 256M ,则执行
```
git clone https://github.com/milkv-duo/tpu-sdk-sg200x.git
mv ./tpu-sdk-sg200x ./tpu-sdk
```

### 将开发工具包和模型文件拷贝到 Duo 开发板上

在 duo 开发板的终端中，新建文件目录 `/mnt/tpu/`
```
# mkdir -p /mnt/tpu && cd /mnt/tpu
```

在 Docker 的终端中，将 `tpu-sdk` 和模型文件拷贝到 Duo 开发板上
```
# scp -r /workspace/tpu-sdk root@192.168.42.1:/mnt/tpu/
# scp /workspace/yolov5n_torch/work/yolov5n_int8_fuse.cvimodel root@192.168.42.1:/mnt/tpu/tpu-sdk
```

### 设置环境变量

在 Duo 开发板的终端中，进行环境变量的设置
```
# cd /mnt/tpu/tpu-sdk
# source ./envs_tpu_sdk.sh
```

### 进行目标检测

在 Duo 开发板上，对该图像进行目标检测

![duo](/docs/duo/tpu/duo-tpu-dog.jpg)

在 Duo 开发板的终端中，使用 `yolov5n_int8_fuse.cvimodel` 模型进行目标检测
```
# ./samples/samples_extra/bin/cvi_sample_detector_yolo_v5_fused_preprocess \
 ./yolov5n_int8_fuse.cvimodel \
 ./samples/samples_extra/data/dog.jpg \
 yolov5n_out.jpg
 ```

检测成功结果示例

![duo](/docs/duo/tpu/duo-tpu-yolo5_12.png)

运行成功后，会生成检测结果文件`yolov5n_out.jpg`，可以在 Windows 终端中通过 scp 命令拉取到 PC 上
```
scp root@192.168.42.1:/mnt/tpu/cvitek_tpu_sdk/yolov5n_out.jpg .
```

![duo](/docs/duo/tpu/duo-tpu-yolo5_13.png)

在 Windows PC 中查看检测结果文件

![duo](/docs/duo/tpu/duo-tpu-yolo5_14.jpg)
