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

运行日志全部输出到 log 文件夹，同时保持双击启动的易用性。
最终结构如下：

```text
D:\Tools\AiLLM\
├── LLama\
│   ├── llama-server.exe
│   ├── models\
│       └── Qwen3-VL-2B-Instruct-1M-BF16.gguf   
│       
└── Auto\
    ├── 启动大模型对话.bat     ← 更新为上述内容
    └── log\                   ← 日志自动产生
```



1. 粘贴以下代码，保存为`启动大模型对话.bat`（注意扩展名是 .bat 不是 .txt）


```bat
@echo off
title Llama 视觉大模型服务

:: 设置路径
set "LOG_DIR=D:\Tools\AiLLM\Auto\log"
set "LLAMA_DIR=D:\Tools\AiLLM\LLama"
set "MODEL=models\Qwen3-VL-2B-Instruct-1M-BF16.gguf"
set "MMPROJ=models\mmproj-BF16.gguf"

:: 创建日志文件夹
if not exist "%LOG_DIR%" mkdir "%LOG_DIR%"

:: 生成时间戳日志文件名（纯数字）
for /f "tokens=2 delims==" %%i in ('wmic os get localdatetime /value') do set datetime=%%i
set "LOGFILE=%LOG_DIR%\llama_%datetime:~0,8%_%datetime:~8,6%.log"

echo Starting llama-server with vision support, log: %LOGFILE%

:: 启动 llama-server（后台最小化运行，带视觉投影）
cd /d "%LLAMA_DIR%"
start "LlamaServer" /min cmd /c "llama-server.exe -m %MODEL% --mmproj %MMPROJ% --host 127.0.0.1 --port 8081 -ngl 999 -c 12288 --threads 8 > "%LOGFILE%" 2>&1"

:: 1. 等待端口监听（最多 15 秒）
set n=0
:wait_port
timeout /t 1 >nul
netstat -an | findstr ":8081.*LISTENING" >nul
if %errorlevel%==0 goto wait_http
set /a n+=1
if %n% lss 15 goto wait_port
echo Error: Port 8081 not listening, check log.
pause
exit /b 1

:: 2. 等待 HTTP 服务真正就绪（通过访问根路径检测，最多 15 秒）
:wait_http
set m=0
:http_loop
timeout /t 1 >nul
powershell -Command "try { (Invoke-WebRequest -Uri 'http://127.0.0.1:8081' -Method Head -TimeoutSec 1).StatusCode -eq 200 } catch { $false }" >nul 2>&1
if %errorlevel%==0 (
    set /a m+=1
    if %m% lss 15 goto http_loop
    echo Warning: HTTP service not ready after port listening, but will open browser anyway.
    goto open
)

:open
echo Service ready, opening web page...
start http://127.0.0.1:8081
echo All done. Closing this window won't stop the background service.
pause
```


















