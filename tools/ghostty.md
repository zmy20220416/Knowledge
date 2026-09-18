# Ghostty 背景模糊（niri WM）

> 注意：该内容可能已经过时，需要重新验证。笔记中的版本相关结论（Ghostty 1.3.1 为最新稳定版、`ext-background-effect` 支持仍在未发布的 1.4.0）具有时效性，请用 `ghostty --version` 和官方 Releases 重新确认当前版本。

## 摘要

在 niri 合成器下为 Ghostty 终端启用背景模糊，主要依靠 niri 侧的 `background-effect` 窗口规则（compositor 端模糊，不要求应用支持协议），再配合 Ghostty 的背景透明度即可。Ghostty 自带的 `background-blur` 在 niri 上暂无效。

## 核心内容

- **niri 侧模糊（推荐，当前可用）**：
  - niri ≥ 26.04 提供 `background-effect` 窗口规则与全局 `blur {}` 配置。
  - 该方式是合成器端生效，不要求应用实现 `ext-background-effect` 协议，因此对 Ghostty 1.3.1 也有效。
- **Ghostty 侧的 `background-blur`（暂不适用于 niri）**：
  - Ghostty 1.3.x 的 `background-blur` 在 Linux 上仅支持 macOS 和 KDE Plasma，niri 不在支持列表内，设置后无效果。
  - Ghostty 对 `ext-background-effect` 协议的支持加入于未发布的 1.4.0（相关 PR 里程碑为 1.4.0）；在当前最新稳定版 1.3.1 中不可用。
- 无论用哪种方式，窗口都必须半透明（`background-opacity < 1`），否则窗口不透明，看不到背后的模糊。

## 示例

### Ghostty 配置

文件：`~/.config/ghostty/config`

```ini
# 必须小于 1，否则背景不透明，看不到模糊
background-opacity = 0.85
```

1.3.1 上不需要（也无效）设置 `background-blur`。等 Ghostty 1.4.0 发布后，可加：

```ini
# Ghostty 1.4.0+ 才支持通过 ext-background-effect 向合成器请求模糊
background-blur = true
```

### niri 配置

文件：`~/.config/niri/config.kdl`

```kdl
// 为 Ghostty 开启背景模糊
window-rule {
    match app-id="com.mitchellh.ghostty"

    background-effect {
        blur true
        xray true        // 默认开启，只模糊壁纸，性能更好
        noise 0.02
        saturation 1.5
    }
}

// 全局模糊强度
blur {
    // off
    passes 3
    offset 3.0
    noise 0.02
    saturation 1.5
}
```

## 注意事项

- 当前（Ghostty 1.3.1）应使用 niri 的 `blur true` 窗口规则，而不是 Ghostty 的 `background-blur`。
- niri 窗口规则的模糊跟随 `geometry-corner-radius` 设置的圆角；Ghostty 通过协议请求的模糊（1.4.0+）则精确跟随窗口自身形状。
- `background-opacity` 必须小于 1，否则看不到背景模糊效果。
- 开启 `xray false`（非 xray 模糊，实验性）时，窗口开关动画期间模糊会消失，且移动窗口、下方内容变化时开销更大。
- niri < 26.04 时不支持窗口模糊，需升级 niri。
- 可用 `niri --version` 和 `ghostty --version` 确认版本。

## 相关知识

- 暂无

## 外部来源

- Ghostty 配置参考：https://ghostty.org/docs/config/reference
- Ghostty Releases（确认最新稳定版）：https://github.com/ghostty-org/ghostty/releases
- niri Window Rules：https://github.com/niri-wm/niri/wiki/Configuration:-Window-Rules
- niri Window Effects：https://github.com/niri-wm/niri/wiki/Window-Effects
- niri Miscellaneous（`blur {}` 全局配置）：https://github.com/niri-wm/niri/wiki/Configuration:-Miscellaneous
