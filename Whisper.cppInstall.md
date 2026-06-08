## 一、从GitHub拉取安装文件

1. 从[releases](https://github.com/ggerganov/whisper.cpp/releases)下载最新版whisper-bin-x64.zip(纯 CPU 版本)；whisper-cublas-12.4.0-bin-x64.zip是 NVIDIA 显卡版本
2. 从官方下载处理大模型[Files and versions](https://huggingface.co/ggerganov/whisper.cpp/tree/main)，一般选择ggml-tiny.bin（中文可用，仅 75MB），也可以ggml-small.bin（准确度更高，约 466MB）

## 二、导入大模型
1. 从huggingface下载模型(注意需要虽然是.bin后缀，实际上是gguf，不影响使用)
2. 在whisper.cpp安装目录下创建models文件夹(该文件夹与whisper-server.exe处于同一层级)
3. 随后将xxx.bin的模型文件放入models里

## 三、启动模型
1. 在 **whisper-server.exe** 所在的文件夹地址栏输入 `cmd` 回车。或者在资源管理器左上角点击`文件`用`PowerShell`
2. 输入`.\whisper-server.exe -m models\ggml-small.bin --host 127.0.0.1 --port 8082 -l zh`
    - `.\whisper-server.exe` 执行程序的名称 `.\` 表示在当前目录
    - `-m`（或 --model）加载的模型
    - `models\ggml-small.bin`路径，要与`-m`配合使用
    - `--host 127.0.0.1`监听 IP 地址
    - `--port 8082`设置 HTTP 服务监听的端口号
    - `-l zh`指定中文，提高识别准确率
3. 在浏览器输入`127.0.0.1:8082`即可进入web交互界面

## 附加
制作一个启动脚本

运行日志全部输出到 log 文件夹，同时保持双击启动的易用性。
最终结构如下：

```text
D:\Tools\AiLLM\
├── ASR\
│   ├── whisper-server.exe
│   ├── models\
│   │   └── ggml-small.bin
│   └── public\
│       └── index.html        ← 从 Auto\web\ 复制过来
└── Auto\
    ├── 启动语音转文字.bat     ← 更新为上述内容
    └── log\                   ← 日志自动产生
```


1. 粘贴以下代码，保存为`启动语音转文字.bat`（注意扩展名是 .bat 不是 .txt）
```bat
@echo off
chcp 65001 >nul
title Whisper 语音转文字服务

:: 设置路径
set "LOG_DIR=D:\Tools\AiLLM\Auto\log"
set "WHISPER_DIR=D:\Tools\AiLLM\ASR"
set "MODEL=models\ggml-small.bin"

:: 创建日志文件夹
if not exist "%LOG_DIR%" mkdir "%LOG_DIR%"

:: 生成时间戳日志文件名（纯数字，避免中文）
for /f "tokens=2 delims==" %%i in ('wmic os get localdatetime /value') do set datetime=%%i
set "LOGFILE=%LOG_DIR%\whisper_%datetime:~0,8%_%datetime:~8,6%.log"

echo 服务启动中，日志：%LOGFILE%

:: 启动 whisper-server，使用 --public 托管网页，无需 --cors
cd /d "%WHISPER_DIR%"
start "WhisperServer" /min cmd /c "whisper-server.exe -m %MODEL% -l zh --host 127.0.0.1 --port 8082 --public public > "%LOGFILE%" 2>&1"

:: 等待端口监听（最多 15 秒）
set n=0
:wait
timeout /t 1 >nul
netstat -an | findstr ":8082.*LISTENING" >nul
if %errorlevel%==0 goto open
set /a n+=1
if %n% lss 15 goto wait
echo 错误：服务启动超时，请检查日志文件。
pause
exit /b 1

:open
echo 服务已就绪，正在打开网页...
start http://127.0.0.1:8082
echo 全部完成，关闭本窗口不会影响后台服务。
pause
```
    - 解释：
            - `cd /d D:\Tools\AiLLM\ASR` 自动切换到 ASR 目录，自行修改到自己的路径




2. 在 D:\Tools\AiLLM\Auto\ 下创建`Start-Whisper.ps1`内容如下

```powershell
$logDir = "D:\Tools\AiLLM\Auto\log"
if (-not (Test-Path $logDir)) { New-Item -ItemType Directory -Path $logDir | Out-Null }

$timestamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"
$logFile = "$logDir\whisper_$timestamp.log"

# 启动 whisper-server，同时将 stdout/stderr 写入日志文件，并在控制台显示
$process = Start-Process -FilePath "D:\Tools\AiLLM\ASR\whisper-server.exe" `
    -ArgumentList '-m models\ggml-small.bin --host 127.0.0.1 --port 8082 -l zh --cors "*"' `
    -RedirectStandardOutput $logFile `
    -RedirectStandardError "$logDir\whisper_error_$timestamp.log" `
    -NoNewWindow -PassThru

# 等待服务器启动（检查端口是否监听）
Write-Host "等待 Whisper 服务启动..." -ForegroundColor Cyan
$timeout = 10
do {
    Start-Sleep -Seconds 1
    $timeout--
    $listening = (Test-NetConnection -ComputerName 127.0.0.1 -Port 8082 -InformationLevel Quiet) -eq $true
} while (-not $listening -and $timeout -gt 0)

if ($listening) {
    Write-Host "Whisper 服务已启动，日志: $logFile" -ForegroundColor Green
    Start-Process "http://127.0.0.1:8082"
    Write-Host "网页已打开，关闭本窗口将不会停止服务。"
    # 保持脚本运行，防止窗口关闭（但服务已脱离）
    Pause
} else {
    Write-Host "服务启动超时，请检查日志: $logFile" -ForegroundColor Red
    Pause
}
```



3. 制作静态网页

将下方代码复制粘贴到文本文件中，并将其命名为`index.html`；保存到`D:\Tools\AiLLM\Auto\web`下，同时也要在`D:\Tools\AiLLM\ASR` 下新建文件夹 `public`，把我们的`index.html` 复制进去

```html
<!DOCTYPE html>
<html lang="zh">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Whisper 语音转文字 · 本地版</title>
<style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: 'Segoe UI', system-ui, sans-serif; background: #f5f7fb; display: flex; height: 100vh; color: #333; }
    .main { flex: 1; display: flex; flex-direction: column; align-items: center; justify-content: center; padding: 40px; }
    .card { background: white; border-radius: 20px; box-shadow: 0 10px 30px rgba(0,0,0,0.05); padding: 40px; width: 100%; max-width: 600px; }
    h1 { font-size: 28px; margin-bottom: 8px; color: #1a1a2e; }
    .sub { color: #888; margin-bottom: 30px; font-size: 15px; }
    .upload-area { border: 2px dashed #ccd4e0; border-radius: 16px; padding: 40px 20px; text-align: center; transition: border-color 0.3s; background: #fafbfd; margin-bottom: 24px; }
    .upload-area.drag-over { border-color: #4f6ef7; background: #eef2ff; }
    .upload-area input { display: none; }
    .upload-icon { font-size: 40px; margin-bottom: 12px; }
    .upload-text { color: #555; font-size: 15px; }
    .upload-hint { color: #aaa; font-size: 13px; margin-top: 6px; }
    .row { display: flex; gap: 16px; margin-bottom: 20px; flex-wrap: wrap; }
    .form-group { flex: 1; min-width: 140px; }
    label { display: block; font-size: 13px; font-weight: 500; color: #555; margin-bottom: 6px; }
    select, button { width: 100%; padding: 12px 16px; border-radius: 10px; border: 1px solid #dde1e8; font-size: 15px; background: white; cursor: pointer; }
    button { background: #4f6ef7; color: white; border: none; font-weight: 600; transition: background 0.2s; }
    button:hover { background: #3b54d4; }
    button:disabled { background: #b0bae0; cursor: not-allowed; }
    .status { margin-top: 20px; padding: 16px; border-radius: 12px; background: #f0f4ff; color: #2c3e90; font-size: 14px; text-align: center; }
    .status.error { background: #ffeaea; color: #b33; }
    .result-link { margin-top: 12px; display: block; color: #4f6ef7; text-decoration: underline; cursor: pointer; }

    /* 右侧历史栏 */
    .sidebar { width: 280px; background: white; border-left: 1px solid #e0e4e9; display: flex; flex-direction: column; }
    .sidebar-header { padding: 24px 20px 12px; font-size: 18px; font-weight: 600; color: #1a1a2e; display: flex; justify-content: space-between; align-items: center; }
    .sidebar-header .export-btn { background: none; border: 1px solid #ccd4e0; color: #555; padding: 6px 12px; border-radius: 8px; font-size: 13px; cursor: pointer; width: auto; }
    .history-list { flex: 1; overflow-y: auto; padding: 0 16px 16px; }
    .history-item { padding: 12px; border-radius: 10px; margin-bottom: 8px; background: #f8f9fc; font-size: 14px; }
    .history-item .name { font-weight: 500; margin-bottom: 4px; }
    .history-item .meta { font-size: 12px; color: #888; display: flex; justify-content: space-between; }
    .clear-btn { background: none; border: 1px solid #ddd; color: #666; margin: 12px 16px 16px; padding: 10px; border-radius: 10px; width: auto; cursor: pointer; }
    .footer-note { padding: 12px 20px; font-size: 12px; color: #aaa; border-top: 1px solid #e0e4e9; text-align: center; }
</style>
</head>
<body>

<!-- 主体 -->
<div class="main">
    <div class="card">
        <h1>🎤 语音转文字</h1>
        <p class="sub">完全本地 · 私密安全 · 带时间戳</p>
        
        <div class="upload-area" id="dropZone">
            <div class="upload-icon">📂</div>
            <div class="upload-text">点击或拖拽音频文件到此处</div>
            <div class="upload-hint">支持 MP3, WAV, FLAC, OGG, M4A 等</div>
            <input type="file" id="fileInput" accept="audio/*">
        </div>

        <div class="row">
            <div class="form-group">
                <label>输出格式</label>
                <select id="formatSelect">
                    <option value="srt">SRT 字幕</option>
                    <option value="vtt">WebVTT 字幕</option>
                    <option value="json">JSON (详细)</option>
                    <option value="txt">纯文本 (无时间戳)</option>
                </select>
            </div>
            <div class="form-group">
                <label>语言</label>
                <select id="langSelect">
                    <option value="zh">中文</option>
                    <option value="auto">自动检测</option>
                    <option value="en">英文</option>
                </select>
            </div>
        </div>

        <button id="submitBtn" disabled>开始转录</button>
        <div class="status" id="statusBox" style="display:none;"></div>
        <a class="result-link" id="downloadLink" style="display:none;">⬇️ 下载结果文件</a>
    </div>
</div>

<!-- 右侧历史记录 -->
<div class="sidebar">
    <div class="sidebar-header">
        📋 历史记录
        <button class="export-btn" onclick="exportHistory()">导出</button>
    </div>
    <div class="history-list" id="historyList"></div>
    <button class="clear-btn" onclick="clearHistory()">清空记录</button>
    <div class="footer-note">服务端日志：<br>D:\Tools\AiLLM\Auto\log</div>
</div>

<script>
const API_BASE = 'http://127.0.0.1:8082';
const dropZone = document.getElementById('dropZone');
const fileInput = document.getElementById('fileInput');
const submitBtn = document.getElementById('submitBtn');
const formatSelect = document.getElementById('formatSelect');
const langSelect = document.getElementById('langSelect');
const statusBox = document.getElementById('statusBox');
const downloadLink = document.getElementById('downloadLink');
const historyList = document.getElementById('historyList');

let selectedFile = null;

dropZone.addEventListener('click', () => fileInput.click());
fileInput.addEventListener('change', handleFileSelect);

dropZone.addEventListener('dragover', e => {
    e.preventDefault();
    dropZone.classList.add('drag-over');
});
dropZone.addEventListener('dragleave', () => dropZone.classList.remove('drag-over'));
dropZone.addEventListener('drop', e => {
    e.preventDefault();
    dropZone.classList.remove('drag-over');
    if (e.dataTransfer.files.length) {
        fileInput.files = e.dataTransfer.files;
        handleFileSelect();
    }
});

function handleFileSelect() {
    const file = fileInput.files[0];
    if (file) {
        selectedFile = file;
        dropZone.querySelector('.upload-text').textContent = file.name;
        submitBtn.disabled = false;
    }
}

submitBtn.addEventListener('click', async () => {
    if (!selectedFile) return;
    submitBtn.disabled = true;
    statusBox.style.display = 'block';
    statusBox.textContent = '正在转录，请稍候...';
    statusBox.className = 'status';
    downloadLink.style.display = 'none';

    const formData = new FormData();
    formData.append('file', selectedFile);
    formData.append('temperature', '0.0');
    formData.append('response_format', formatSelect.value);
    formData.append('language', langSelect.value);

    try {
        const response = await fetch(`${API_BASE}/inference`, {
            method: 'POST',
            body: formData
        });

        if (!response.ok) {
            throw new Error(`服务器错误 (${response.status})`);
        }

        const ext = formatSelect.value === 'json' ? 'json' : formatSelect.value;
        const fileName = selectedFile.name.replace(/\.[^/.]+$/, "") + '.' + ext;

        const blob = await response.blob();
        const url = URL.createObjectURL(blob);
        downloadLink.href = url;
        downloadLink.download = fileName;
        downloadLink.style.display = 'block';
        downloadLink.textContent = `⬇️ 下载 ${fileName}`;

        statusBox.textContent = '✅ 转录完成！';
        statusBox.className = 'status';

        // 保存历史记录
        saveHistory({
            name: selectedFile.name,
            format: formatSelect.value,
            time: new Date().toLocaleString()
        });
        renderHistory();

    } catch (err) {
        statusBox.textContent = `❌ 错误: ${err.message}`;
        statusBox.className = 'status error';
    }
    submitBtn.disabled = false;
});

// ========= 历史记录 =========
function getHistory() {
    try {
        return JSON.parse(localStorage.getItem('whisper_history') || '[]');
    } catch { return []; }
}
function saveHistory(entry) {
    let history = getHistory();
    history.unshift(entry);
    if (history.length > 50) history.pop();
    localStorage.setItem('whisper_history', JSON.stringify(history));
}
function clearHistory() {
    localStorage.removeItem('whisper_history');
    renderHistory();
}
function renderHistory() {
    const history = getHistory();
    historyList.innerHTML = history.map(item => `
        <div class="history-item">
            <div class="name">${item.name}</div>
            <div class="meta"><span>${item.format.toUpperCase()}</span><span>${item.time}</span></div>
        </div>
    `).join('');
}

// 导出历史为 JSON 文件（自动下载）
function exportHistory() {
    const history = getHistory();
    const blob = new Blob([JSON.stringify(history, null, 2)], {type: 'application/json'});
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-').slice(0,19);
    a.href = url;
    a.download = `web_history_${timestamp}.json`;
    a.click();
    URL.revokeObjectURL(url);
}

renderHistory();
</script>
</body>
</html>
```
































