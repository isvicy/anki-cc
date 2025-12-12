# Anki macOS 桌面版视频无法播放问题

## 问题描述

在 macOS 桌面版 Anki 中，卡片内嵌的 MP4 视频无法播放。症状表现为：
- 视频显示 loading 圆圈后静止
- 点击播放按钮无反应
- 同样的卡片在 iOS、Android、Linux 上都能正常播放

## 环境信息

- macOS: Darwin 25.1.0
- Anki: 25.09
- 视频格式: MP4 (H.264 + AAC)

## 根本原因

**Qt WebEngine 不支持 H.264 编解码器**

Anki 桌面版使用 Qt WebEngine 作为渲染引擎。由于 H.264 是专有编解码器，Qt WebEngine 默认不包含它（需要额外的许可证）。

参考资料：
- [Qt Forum: Can't play video using QtWebEngine](https://forum.qt.io/topic/152191/can-t-play-video-using-qtwebengine)
- [Qt WebEngine Features](https://qt.developpez.com/doc/6.4/qtwebengine-features/)

## 诊断过程

### 1. 确认视频格式

```bash
mdls -name kMDItemCodecs "/path/to/video.mp4"
# 输出: H.264, MPEG-4 AAC
```

### 2. 确认 Anki 版本

```bash
defaults read /Applications/Anki.app/Contents/Info.plist CFBundleShortVersionString
# 输出: 25.09
```

### 3. 调试 JavaScript 环境

在卡片模板中添加调试代码：

```javascript
(function(){
    var d = document.createElement('div');
    d.style.cssText = 'background:yellow;padding:10px;';
    d.innerHTML = 'Platform: ' + navigator.platform + '<br>' +
                  'pycmd: ' + (typeof pycmd) + '<br>' +
                  'bridgeCommand: ' + (typeof bridgeCommand);
    document.body.insertBefore(d, document.body.firstChild);
})();
```

### 4. 尝试过的方案（失败）

1. **修改 JavaScript 自动播放逻辑** - 无效，因为 H.264 根本不支持
2. **使用 `pycmd('ankiplay...')` 调用外部播放器** - `ankiplay` 前缀在新版 Anki 中不被识别
3. **使用 `pycmd('play:q:0')` 调用 Anki 内置播放器** - 可以工作，但视频用 mpv 悬浮窗播放，不是内嵌

## 解决方案

### 方案: 将视频转换为 WebM (VP9) 格式

VP9 是开放编解码器，Qt WebEngine 原生支持。

#### 步骤 1: 备份原始文件

```bash
mkdir -p ~/Desktop/anki_video_backup
cd "/Users/aaron/Library/Application Support/Anki2/账户 1/collection.media"
cp _family_*.mp4 ~/Desktop/anki_video_backup/
```

#### 步骤 2: 批量转换

创建转换脚本 `/tmp/convert_videos.sh`:

```bash
#!/bin/bash
cd "/Users/aaron/Library/Application Support/Anki2/账户 1/collection.media" || exit 1

count=0
total=$(ls _family_*.mp4 2>/dev/null | wc -l | tr -d ' ')
echo "Total files to convert: $total"

for f in _family_*.mp4; do
    if [ ! -f "$f" ]; then
        continue
    fi
    count=$((count + 1))
    echo "[$count/$total] Converting: $f"

    # 临时文件用 .webm 扩展名
    tempfile="${f%.mp4}_temp.webm"

    # Convert to webm with VP9
    ffmpeg -y -i "$f" -c:v libvpx-vp9 -crf 35 -b:v 0 -c:a libopus -b:a 64k "$tempfile" 2>/dev/null

    if [ -f "$tempfile" ]; then
        mv "$tempfile" "$f"
        echo "  OK"
    else
        echo "  FAILED - keeping original"
    fi
done

echo "Conversion completed!"
```

执行转换：

```bash
chmod +x /tmp/convert_videos.sh
/bin/bash /tmp/convert_videos.sh
```

#### 步骤 3: 验证转换结果

```bash
file _family_s01e01_0000292.mp4
# 输出应该是: WebM
```

#### 步骤 4: 更新卡片模板

简化后的模板（所有平台统一使用 HTML5 video）：

```html
<div id="stage-face" class="section" >
    <img src="_mj_modern_family.jpg" width="100%" height="auto">
</div>
<div class="section">
    <div id="tv-src" style="display:none;">{{Video}}</div>
    <div id="tv" class="item">
        <video id="myvideo" width="100%" height="auto" src="{{Video}}" controls="controls" webkit-playsinline="true" playsinline="true" poster="_mj_modern_family.jpg" preload="auto"></video>
    </div>
</div>

<div class="section2">
    <div id="voice" class="item">
        <span>播放音频：</span>
        <img src="_mj_play_button.png" onClick="playAudio(1);">
        <img src="_mj_pause_button.png" class="mobile" onClick="playAudio(0);">
        <audio id="audiotag" src="{{Audio}}" style="display:none"></audio>
        <label class="mobile"><input id="loop" name="loop" type="checkbox" onclick="loopAudio(this);"/>循环播放</label>
    </div>
</div>

<div class="footer">
    <p>{{Timeline}}</p>
</div>

<script type="text/javascript">
    window.playAudio = function(bPlay) {
        var player = document.getElementById('audiotag');
        var v = document.getElementById('myvideo');
        if (player) {
            if (v && !v.paused) v.pause();
            if (bPlay) {
                player.currentTime = 0;
                player.play().catch(console.error);
            } else {
                player.pause();
            }
        }
    };

    window.loopAudio = function(cb) {
        var player = document.getElementById('audiotag');
        if (player) cb.checked ? player.setAttribute("loop", "true") : player.removeAttribute("loop");
    };

    (function(){
        var sf = document.getElementById("stage-face");
        if (sf) sf.style.display = "none";
    })();
</script>
```

## 结果

| 平台 | 转换前 | 转换后 |
|------|--------|--------|
| macOS 桌面 | 无法播放 | 点击播放（带声音） |
| Windows | 需要测试 | 点击播放（带声音） |
| Linux | 正常 | 点击播放（带声音） |
| iOS | 正常 | 点击播放（带声音） |
| Android | 正常 | 点击播放（带声音） |

文件大小变化: 70MB → 59MB (减小 16%)

## 其他可选方案

### 方案 B: 使用 Anki 内置 mpv 播放器

如果不想转换视频格式，可以使用 Anki 的 `[sound:...]` 语法配合 `pycmd('play:q:0')` 调用内置 mpv 播放器：

```html
<div style="display:none;">[sound:{{Video}}]</div>
<img src="_mj_play_button.png" onclick="pycmd('play:q:0');">
```

缺点：mpv 以悬浮窗形式播放，不是内嵌在卡片中。

## 注意事项

1. **备份原始文件** - 转换前务必备份
2. **ffmpeg 参数** - `-crf 35` 控制质量，数值越小质量越高但文件越大
3. **Anki 同步** - 转换后的视频文件会在下次同步时上传到 AnkiWeb
4. **移动端兼容性** - VP9 在现代移动设备上都支持

## 相关命令参考

```bash
# 检查视频编码
mdls -name kMDItemCodecs "/path/to/video.mp4"

# 检查 Anki 内置 mpv
"/Users/aaron/Library/Application Support/AnkiProgramFiles/cache/archive-v0/*/anki_audio/mpv" --version

# 单个文件转换测试
ffmpeg -i input.mp4 -c:v libvpx-vp9 -crf 35 -b:v 0 -c:a libopus -b:a 64k output.webm
```
