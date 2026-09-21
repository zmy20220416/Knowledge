# npm allow-scripts 安装脚本安全拦截

## 摘要

`npm` 在全局安装某些包（如 `@nubjs/nub`）时，可能会提示使用 `--allow-scripts=@nubjs/nub` 或将其加入用户配置。这是 npm 对**包安装阶段生命周期脚本**的安全拦截机制。

## 核心内容

### 什么是安装脚本

npm 包可以在 `package.json` 中定义以下生命周期脚本，这些脚本会在 `npm install` 时自动执行：

- `preinstall`
- `install`
- `postinstall`

`@nubjs/nub` 这类包通常需要这些脚本完成编译、下载平台资源或权限设置。

### 安全拦截机制

npm 默认拦截安装脚本，要求用户显式允许某个包运行脚本，防止恶意包在 `npm i` 时执行任意命令（如窃取环境变量、植入后门）。

### 两种允许方式

仅本次安装允许：

```bash
npm install -g --allow-scripts=@nubjs/nub @nubjs/nub
```

永久允许（写入用户配置）：

```bash
npm config set allow-scripts=@nubjs/nub --location=user
npm i -g @nubjs/nub
```

## 示例

安装 `@nubjs/nub` 时被拦截：

```bash
npm i -g @nubjs/nub
# 提示：Run `npm install -g --allow-scripts=@nubjs/nub` to allow these scripts once,
# or `npm config set allow-scripts=@nubjs/nub --location=user` to allow them for all global installs.
```

## 注意事项

- `allow-scripts` 检查的是**正在被安装的包本身**，例如这次检查的是 `nub` 这个包。
- 安装完成后，运行 `nub install` 这类命令时，如果 nub 内部继续安装其他 npm 包，而那些包也包含安装脚本，npm 可能再次弹出 allow-scripts 提示。
- 只有在确认包来源可信时，才应将其加入允许列表。

## 相关知识

- npm lifecycle scripts
- 供应链攻击（Supply Chain Attack）
- `npm config`

## 外部来源

- 本次记录来自与 npm 安装 `@nubjs/nub` 时的实际交互。

## 重要限制

- `allow-scripts` 只控制安装阶段的脚本执行，不影响包安装后的运行时行为。
