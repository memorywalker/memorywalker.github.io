---
title: AMD GPU使用ComfyUI-Zluda简单图像生成
date: 2025-06-08 09:07:49
categories:
- AI
tags:
- AI
- SD
- Comfyui
- AMD
---

## AMD GPU使用ComfyUI-Zluda简单图像生成 

使用ComfyUI进行间的的文本图像生成，AMD显卡运行pytorch需要额外的配置

### AMD显卡Rocm HIP SDK

以我的电脑AMD 6650 XT 8G显卡为例：

1. 从  https://rocm.docs.amd.com/projects/install-on-windows/en/develop/reference/system-requirements.html 查看AMD Radeon表格中可以看到6650XTLLVM的目标环境为gfx1032，默认支持Runtime，但是没有SDK支持

2. 下载AMD的HIP SDK https://www.amd.com/en/developer/resources/rocm-hub/hip-sdk.html ，目前最新版本是6.2.4，HIP SDK可以简单理解为AMD的CUDA平替

3. 在 https://github.com/likelovewant/ROCmLibs-for-gfx1103-AMD780M-APU/releases 下载适用于gfx1032的6.2.4版本[rocm.gfx1032.for.hip.sdk.6.2.4.navi21.logic.7z](https://github.com/likelovewant/ROCmLibs-for-gfx1103-AMD780M-APU/releases/download/v0.6.2.4/rocm.gfx1032.for.hip.sdk.6.2.4.navi21.logic.7z)

    预编译好的库文件。ROCm是AMD的开源GPU计算软件堆栈，旨在提供一个可移植、高性能的GPU计算平台。

4. 安装HIP SDK后，把下载的rocm.gfx1032.for.hip.sdk.6.2.4.navi21.logic.7z中的文件覆盖 `C:\Program Files\AMD\ROCm\6.2\bin`目录中的`rocblas.dll`和`C:\Program Files\AMD\ROCm\6.2\bin\rocblas\library`目录

5. 系统环境变量path中添加 `C:\Program Files\AMD\ROCm\6.2\bin`目录


### 安装ComfyUI-Zluda

ComfyUI-Zluda项目的网址为https://github.com/patientx/ComfyUI-Zluda

 1.  参考项目主页的[说明]( https://github.com/patientx/ComfyUI-Zluda?tab=readme-ov-file#dependencies )  ，确认安装依赖环境，包括git，python，VC运行时以及AMD HIP这个说明文件很详细的说明了依赖需要的版本和注意事项；python的版本我本机之前安装的是3.12就保持不变，VC运行时重新安装了一遍；AMD HIP 安装的6.2版本

 2.  在E:\ai目录下执行`git clone https://github.com/patientx/ComfyUI-Zluda`，可以把项目下载到ComfyUI-Zluda目录中

 3.  进入到ComfyUI-Zluda目录中执行install.bat进行安装，这个过程需要**外网**连接，同时安装过程中也会提示下载torch文件很大，需要很长时间 。安装过程中会在当前目录中创建venv的目录作为python虚拟环境，安装完成后虚拟环境目录大小为6G。详细安装的内容可以查看install.bat文件。由于安装过程会自动安装ZLUDA补丁，所以不用自己单独下载ZLUDA补丁了。

     ![comfyui_zluda_insall](../../uploads/ai/comfyui_zluda_insall.png)
     ![comfyui_zluda_insall](/uploads/ai/comfyui_zluda_insall.png)

 4.  第一次安装完成后，Comfyui会自动运行，并打开浏览器的http://127.0.0.1:8188/
     浏览器显示如下：

    ![Comfyui_webui](../../uploads/ai/Comfyui_webui.png)
     ![Comfyui_webui](/uploads/ai/Comfyui_webui.png)

  后台显示：

     ![start_comfyui_zluda](../../uploads/ai/start_comfyui_zluda.png)
     ![start_comfyui_zluda](/uploads/ai/start_comfyui_zluda.png)
### ComfyUI文本生成图像

ComfyUI的使用方法可以到https://comfyui-wiki.com/zh 这个网站学习。

ComfyUI使用工作流的方式来执行生成图像的各个步骤。每个步骤都是一个节点，每个节点有自己的输入和输出，通过输入和输出可以把这些节点连接起来，所有配置好后，执行运行就可以生成图像了。

最简单的方法是用默认提供的基础模板第一个，它包含了最基本文生图的流程。

#### 1. 加载模型

选了基础模板后，会提示没有对应的模型，关掉对话，我们自己下载想要的模型。基本的文生图只需要安装checkpoint对应的模型。

模型的下载可以到https://civitai.com/ 这个网站，这个网站可以按模型类型(checkpoint. lora等)，版本(SD的版本)，分类排序。例如我下载了这个月排名第一名为[Real Dream](https://civitai.com/models/153568/real-dream?modelVersionId=712448) 的模型[realDream_15SD15.safetensors](https://civitai-delivery-worker-prod.5ac0637cfd0766c97916cefa3764fbdf.r2.cloudflarestorage.com/model/58980/realDream15.0zrr.safetensors?X-Amz-Expires=86400&response-content-disposition=attachment%3B%20filename%3D%22realDream_15SD15.safetensors%22&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=e01358d793ad6966166af8b3064953ad/20250607/us-east-1/s3/aws4_request&X-Amz-Date=20250607T115116Z&X-Amz-SignedHeaders=host&X-Amz-Signature=207c7d569d48d1de9683b01eea208d3bfb5f5c9f1cd22c9e5b0d372761fd52e4) ，模型下载下来的大小为2G

ComfyUI的模型都存放在安装目录的models目录下，这个目录里面又根据模型的类型分别有不同的子目录。

因为下载的是Checkpoint模型，所以把模型文件放在`E:\ai\ComfyUI-Zluda\models\checkpoints\SD1.5`目录中，SD1.5目录是自己手动创建用来区分SD的版本，以后可能需要下载很多不同的模型。例如我下载了官方的SD1.5模型 `v1-5-pruned-emaonly.ckpt`文件[下载地址](https://hf-mirror.com/stable-diffusion-v1-5/stable-diffusion-v1-5/tree/main)，也是放在了`checkpoints\SD1.5`目录中。

在ComfyUI的Load Checkpoint节点就可以切换不同的checkpoint模型，这个节点的输出是model，clip和vae。

#### 2. 输入提示词

提示词分为正向和负向两种，正向就是图中需要包含的信息，负向就是图像中没有的信息。提示词节点Clip Text Encode(Prompt) 以Checkpoint的Clip作为输入，输出Contidioning。

例如正向提示词可以输入"A japanese girl, full body, long leg, short hair"，负向提示词输入"text, watermark"

#### 3. 设置图片大小

Latent节点可以设置图片的大小，默认是512*512

#### 4. 图像采样KSampler

这个节点把前面所有的输入进行处理生成图像数据，它的输入model为checkpoint的输出，positive和negative分别对应正向和负向提示词，latent_image和设置图像大小的latent连接

#### 5. 合成图像

VAE Decode节点把生成的采样数据生成图片，它的vae和checkpoint的vae连接，最终把图片输出到最后一个节点Save Image。在Save Image节点中可以保存生成的图像。

### 试用总结

官方模型4G多，网友分享的模型2G，二者比较居然是后者生成的图像质量高很多。官方的1.5模型生成的人物脸都变形了。第一次加载模型使用的时间比较长，后面修改提示词再生成图像就只需要几秒时间。

第一次使用stable diffusion和ComfyUI，很多名词和概念都不明白，但整个过程还是很简单，就像小时候玩积木游戏，一步一步操作，查看输出，满满成就感。

![Comfyui_make_image](../../uploads/ai/Comfyui_make_image.png)
![Comfyui_make_image](/uploads/ai/Comfyui_make_image.png)




