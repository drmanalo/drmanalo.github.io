+++
date = "2025-02-03T18:04:43Z"
draft = false
title = "Hello Ollama"
tags = ["linux","ai"]
+++

## Prerequisites
Being an `archlinux` user means dependencies are for `arch` but this blog post should work with any `nix` distribution.
- [Linux](https://wiki.archlinux.org/title/Installation_guide)

## Installation
```declarative
❯ curl -L https://ollama.com/download/ollama-linux-amd64.tgz

❯ tar -xzf ollama-linux-amd64.tgz

❯ curl -L https://ollama.com/download/ollama-linux-amd64-rocm.tgz

❯ tar -xzf ollama-linux-amd64-rocm.tgz

❯ curl -fsSL https://cdn.useanything.com/latest/installer.sh | sh
>>> Downloading latest AnythingLLM Desktop...
>>> Extracting...
>>> AnythingLLMDesktop is ready to run with ~/AnythingLLMDesktop/start.
>>> You can rerun this installer anytime to get the latest version of AnythingLLM without effecting your existing data.
>>> Documentation: https://docs.useanything.com
>>> Issues: https://github.com/Mintplex-Labs/anything-llm
>>> Thanks for using AnythingLLM!
```

### Create ollama user
```declarative
❯ sudo useradd -r -s /bin/false -U -m -d /home/ollama ollama

❯ sudo usermod -a -G ollama $(whoami)

❯ sudo usermod -a -G video,render ollama
```

### Customisation
Create a service file in `/etc/systemd/system/ollama.service`.
```declarative
[Unit]
Description=Ollama Service
After=network-online.target

[Service]
ExecStart=/home/ollama/bin/ollama serve
User=ollama
Group=ollama
Restart=always
RestartSec=3
Environment="PATH=$PATH;OLLAMA_HOST=0.0.0.0"
```

Create an application launcher in `~/.local/share/applications/anythingllm.desktop`
```
[Desktop Entry]
Category=Artificial Intelligence
Comment=Anything LLM
Exec=~/AnythingLLMDesktop/start
Icon=~/AnythingLLMDesktop/anythingllm-desktop/anythingllm-desktop.png
Name=AnythingLLMDesktop
StartupNotify=true
Terminal=false
Type=Application
Version=1.0
```

[Install]
WantedBy=default.target
```
```declarative
❯ sudo systemctl daemon-reload

❯ sudo systemctl restart ollama

❯ sudo systemctl status ollama
● ollama.service - Ollama Service
Loaded: loaded (/etc/systemd/system/ollama.service; enabled; preset: ::disabled)
Active: active (running) since Mon 2025-02-03 22:59:28 GMT; 15min ago
Invocation: 14477e42c7a24224824a7aa91553b115
Main PID: 19129 (ollama)
Tasks: 29 (limit: 71968)
Memory: 9.4G (peak: 9.5G)
CPU: 34min 21.786s
```

## Get your model
```declarative
❯ ollama pull qwen2.5-coder:14b
pulling manifest
pulling ac9bc7a69dab... 100% ▕███████████████████████████████▏ 9.0 GB
pulling 66b9ea09bd5b... 100% ▕███████████████████████████████▏   68 B
pulling e94a8ecb9327... 100% ▕███████████████████████████████▏ 1.6 KB
pulling 832dd9e00a68... 100% ▕███████████████████████████████▏  11 KB
pulling 0578f229f23a... 100% ▕███████████████████████████████▏  488 B
verifying sha256 digest
writing manifest
success
```

## First run
```declarative
❯ ollama run --verbose qwen2.5-coder:14b
>>> I want a pinescript for double-top double-bottom.
Certainly! Below is an example of a Pine Script that identifies Double Top and Double Bottom patterns on a chart. This script will plot the support and resistance levels where these 
patterns occur.
```

```pinescript
//@version=5
indicator("Double Top/Bottom", overlay=true)

// Input for lookback period
lookback = input.int(20, title="Lookback Period")

// Function to find highs and lows in a range
findHighs(lowest, highest) =>
    array.new_float()
    
findLows(lowest, highest) =>
    array.new_float()

// Find the double top pattern
doubleTopPattern() =>
    var float top1 = na
    var float top2 = na
    var int count = 0
    
    for i = lookback to 0
        highVal = high[i]
        if (top1 == na)
            if (highVal > highest[lookback])
                top1 := highVal
        else
            if (highVal < top1 and highVal > top2)
                top2 := highVal
                count += 1
            else if (highVal > top1)
                top1 := highVal
                count = 0
    [top1, top2, count]

// Find the double bottom pattern
doubleBottomPattern() =>
    var float bot1 = na
    var float bot2 = na
    var int count = 0
    
    for i = lookback to 0
        lowVal = low[i]
        if (bot1 == na)
            if (lowVal < lowest[lookback])
                bot1 := lowVal
        else
            if (lowVal > bot1 and lowVal < bot2)
                bot2 := lowVal
                count += 1
            else if (lowVal < bot1)
                bot1 := lowVal
                count = 0
    [bot1, bot2, count]

// Get the highest high and lowest low in the lookback period
highestHigh = ta.highest(high, lookback)
lowestLow = ta.lowest(low, lookback)

// Find double top/bottom patterns
[top1, top2, topCount] = doubleTopPattern()
[bot1, bot2, botCount] = doubleBottomPattern()

// Plot the support and resistance levels
if (topCount >= 2)
    plot(top1, color=color.red, title="Double Top Support Level 1")
    plot(top2, color=color.red, title="Double Top Resistance Level")

if (botCount >= 2)
    plot(bot1, color=color.green, title="Double Bottom Support Level 1")
    plot(bot2, color=color.green, title="Double Bottom Resistance Level")
```

### How It Works:
1. **Lookback Period**: The script uses a lookback period to identify the highs and lows over that time frame.
2. **Finding Highs and Lows**: The `doubleTopPattern` and `doubleBottomPattern` functions iterate through the bars in reverse order, checking for potential double top or double bottom
   patterns.
3. **Plotting Levels**: Once a pattern is identified, the script plots the support (for double bottom) and resistance (for double top) levels on the chart.

### Customization:
- You can adjust the `lookback` period to change how far back in time the script looks for patterns.
- The script plots the levels as red lines for double tops and green lines for double bottoms. You can modify these colors or add additional plotting options as needed.

```
total duration:       2m15.702582844s
load duration:        13.041176ms
prompt eval count:    41 token(s)
prompt eval duration: 2.458s
prompt eval rate:     16.68 tokens/s
eval count:           750 token(s)
eval duration:        2m12.92s
eval rate:            5.64 tokens/s
```