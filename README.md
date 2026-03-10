# macOS 微信双开指南 & 紫色独立图标版

本指南提供一种在 macOS 上完美实现微信双开的方法，通过克隆应用、修改 Bundle ID、替换应用图标及重新签名，生成一个完全独立运行的“微信双开版”，并带有一眼即可区分的紫色图标。

## 方案优势
1. **完全独立**：不在终端残留黑框，和原生软件体验一致。
2. **方便区分**：通过特殊脚本将其绿色的图标改成了紫色，可同时放在程序坞（Dock）上，不会因为图标相同而点击错误。
3. **稳定防检测**：绕过了微信客户端原本的单例检测机制，比使用 script 命令行打开更加稳定。

---

## 一键自动化脚本方法 (首选推荐)

如果您想一步到位搞定所有的复制、改色、重签名操作，可以直接在终端（Terminal）执行我编写的这款整合命令。

**执行前需要注意：**
1. 您的 Mac 必须安装 `python3` (系统一般自带)。
2. 为了能够对图标进行改色，该脚本会自动安装 python 的图片处理库 `Pillow`。
3. 执行最后一步重新签名时可能需要输入您的电脑开机密码。

**请打开终端，复制并执行以下内容：**

```bash
# 1. 复制一个微信的克隆版到用户的 Applications 文件夹
rm -rf ~/Applications/微信双开.app
cp -R /Applications/WeChat.app ~/Applications/微信双开.app

# 2. 修改它的系统标识 (Bundle ID)，伪装成全新软件
/usr/libexec/PlistBuddy -c "Set :CFBundleIdentifier net.maclub.wechat.dual" ~/Applications/微信双开.app/Contents/Info.plist

# 3. 提取原版图标
iconutil -c iconset ~/Applications/微信双开.app/Contents/Resources/AppIcon.icns -o /tmp/wechat_icon.iconset

# 4. 安装图片处理依赖
python3 -m pip install Pillow

# 5. 执行 Python 脚本，将图标中的所有绿色像素等比位移映射至完美的紫色（微信紫）
cat << 'EOF' > /tmp/shift_hue.py
import glob, colorsys
from PIL import Image

target_hue_shift = 150 / 360.0
for path in glob.glob("/tmp/wechat_icon.iconset/*.png"):
    img = Image.open(path).convert('RGBA')
    pixels = img.load()
    for y in range(img.height):
        for x in range(img.width):
            r, g, b, a = pixels[x, y]
            if a == 0: continue
            h, l, s = colorsys.rgb_to_hls(r/255.0, g/255.0, b/255.0)
            if s > 0.1: # 仅对彩色的像素进行色域翻转，保留白色 logo 本身
                h = (h + target_hue_shift) % 1.0
            nr, ng, nb = colorsys.hls_to_rgb(h, l, s)
            pixels[x, y] = (int(nr * 255), int(ng * 255), int(nb * 255), a)
    img.save(path)
EOF
python3 /tmp/shift_hue.py

# 6. 将改好的紫色图标打包回去
iconutil -c icns /tmp/wechat_icon.iconset -o ~/Applications/微信双开.app/Contents/Resources/AppIcon.icns

# 7. 刷新缓存并彻底重新签名
rm -rf /tmp/wechat_icon.iconset /tmp/shift_hue.py
touch ~/Applications/微信双开.app
codesign --force --deep --sign - ~/Applications/微信双开.app

# 8. 如果 macOS 图标缓存顽固，使用 Swift 原生接口强制为该 APP 烙印自定义图标
cat << 'EOF' > /tmp/set_icon.swift
import Cocoa
let iconPath = NSHomeDirectory() + "/Applications/微信双开.app/Contents/Resources/AppIcon.icns"
let targetPath = NSHomeDirectory() + "/Applications/微信双开.app"
if let image = NSImage(contentsOfFile: iconPath) {
    NSWorkspace.shared.setIcon(image, forFile: targetPath, options: [])
}
EOF
swift /tmp/set_icon.swift
rm -f /tmp/set_icon.swift
killall Dock && killall Finder

# 9. 自动打开紫色的双开版微信！
open ~/Applications/微信双开.app
```

大功告成！之后您可以在访达 (`Finder`) 左侧菜单中的「个人文件夹 (小房子图标) -> Applications」里找到带有全新**紫色**图标的【微信双开.app】，将它拖到程序坞中就可以永久快速使用了！

---

## 免责声明
本教程属于本地客户端环境配置双开，仅供技术研究交流及个人生活工作隔离等合规场景使用。请勿用于任何营销、群发黑产工具。因不可控力所导致的封号或账号异常风险请自行承担。
