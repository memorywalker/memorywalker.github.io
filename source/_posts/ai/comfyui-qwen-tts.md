---
title: ComfyUI使用Qwen-TTS
date: 2026-02-01T15:55:00
categories:
  - AI
tags:
  - AI
  - tts
---

## ComfyUI使用Qwen-TTS


### Qwen-tts

项目主页[Qwen-tts](https://github.com/QwenLM/Qwen3-TTS)

#### 相关模型

* **Qwen3-TTS-Tokenizer-12Hz** 分词模型，把语音编码和解码
* **Qwen3-TTS-12Hz-1.7B-Base** 能够根据用户音频输入实现3秒快速语音克隆的基座模型；可用于微调其他模型。
* **Qwen3-TTS-12Hz-1.7B-CustomVoice** 通过用户指令对目标音色进行风格控制；支持9种覆盖性别、年龄、语言及方言等维度的优质音色。
* **Qwen3-TTS-12Hz-1.7B-VoiceDesign** 根据用户提供的描述设计语音


### ComfyUI使用Qwen-TTS

1. **下载插件**， 项目地址 https://github.com/flybirdxx/ComfyUI-Qwen-TTS/  在custom_nodes目录中执行 `git clone https://github.com/flybirdxx/ComfyUI-Qwen-TTS.git`
2. **安装插件依赖**，进入下载的插件目录后，激活ComfyUI的虚拟环境，执行`pip install -r requirements.txt`下载项目依赖。（可以删除项目依赖中的huggingface的库，因为我直接从魔搭下载模型文件）
3. **下载模型**，进入comfyui的models目录下，新建`qwen-tts`目录，在`qwen-tts`中执行以下命令下载模型，每个模型都有自己的目录，名称要保持官方的一致。由于不需要通过文本描述设计声音，base就行。

```bash
modelscope download --model Qwen/Qwen3-TTS-Tokenizer-12Hz  --local_dir ./Qwen3-TTS-Tokenizer-12Hz
modelscope download --model Qwen/Qwen3-TTS-12Hz-1.7B-Base --local_dir ./Qwen3-TTS-12Hz-1.7B-Base
modelscope download --model Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice --local_dir ./Qwen3-TTS-12Hz-1.7B-CustomVoice
```

4. **运行comfyui**，执行comfuyi.bat

#### 测试使用

最简单的用法：克隆声音，输入参考音频和音频的提示词，克隆声音节点输入要输出的文本，最后连接一个保存音频节点。

这个插件里面还有一些其他节点例如使用模型内置的声音，或者设计声音，以及多人对话节点。

![](uploads/ai/qwen-tts-clone.png.png)

参考使用林志玲声音![林志玲声音](uploads/ai/林志玲.wav)
参考声音的文本：
我希望我在20岁的时候能够好好的去挑战一些事情，然后，让自己多一点能量存在心中，那到30岁的时候，我觉得慢慢更能够沉淀和有所选择 ，之后我觉得生命就是要开始回馈了

使用《重庆森林》中的经典台词

不知道从什么时候开始，在什么东西上面都有个日期，秋刀鱼会过期，肉罐头会过期，连保鲜纸都会过期，我开始怀疑，在这个世界上，还有什么东西是不会过期的?

![](uploads/ai/ComfyUI_00029_.flac)

* 使用过程中**显存使用5.8G** 
* 目标文本输入日语或其他支持的语言，也可以使用克隆的声音进行输出

### 遇到问题

1. `Qwen3TTSTalkerConfig` object has no attribute `pad_token_id` 降低transformers库的版本[#21](https://github.com/flybirdxx/ComfyUI-Qwen-TTS/issues/21)，我当时使用的5.0，使用`pip install transformers==4.57.3` 在虚拟环境中安装这个版本
2. `z_stft() got multiple values for argument 'window'`，需要修改`\custom_nodes\ComfyUI-Qwen-TTS\qwen_tts\core\models\modeling_qwen3_tts.py`中调用`torch.stft()`的方法参数，把每一个参数都指定形参名称
```python
spec = torch.stft(
        input=y,
        n_fft=n_fft,
        hop_length=hop_size,
        win_length=win_size,
        window=hann_window,
        center=center,
        pad_mode="reflect",
        normalized=False,
        onesided=True,
        return_complex=True,
    )
```

