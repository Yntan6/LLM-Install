## 一、从GitHub拉取安装文件

1. 从[releases](https://github.com/ggml-org/llama.cpp/releases)下载最新版的Vulkan版(兼容性最好)，也可以其他版本比如AMD的`HIP`版本(有点麻烦，需要Rcom)
## 二、导入大模型
1. 从huggingface下载模型(注意需要llama.cpp需要gguf后缀)
2. 在llama.cpp安装目录下创建models文件夹(该文件夹与llama-server.exe处于同一层级)
3. 随后将xxx.gguf的模型文件放入models里
## 三、启动模型
1. 在 llama-server.exe 所在的文件夹地址栏输入 `cmd` 回车。或者在资源管理器左上角点击`文件`用`PowerShell`
2. 输入`.\llama-server.exe -m "D:\Tools\AiLLM\LLama\models\Qwen3-0.6B-BF16.gguf" --host 0.0.0.0 --port 8080 -ngl 999 -c 4096`
    - `.\llama-server.exe` 执行程序的名称 `.\` 表示在当前目录
    - `-m`（或 --model）加载的模型
    - `"D:\Tools\AiLLM\LLama\models\Qwen3-0.6B-BF16.gguf"`绝对路径，要与`-m`配合使用
    - `--host 0.0.0.0`监听 IP 地址
    - `--port 8080`设置 HTTP 服务监听的端口号
    - `-ngl 999`（或 --n-gpu-layers）控制将模型卸载到 GPU 显存中运行的层数
    - `-c 4096`（或 --ctx-size）一次对话中能“记住”和处理的最大 Token 数量

## 附加
制作一个启动脚本