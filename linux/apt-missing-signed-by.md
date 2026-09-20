# apt update 提示 Missing Signed-By

## 摘要

在 Ubuntu 26.04（apt 3.x）执行 `sudo apt update` 时，若 deb822 格式的软件源缺少 `Signed-By` 字段，会输出
`Notice: Missing Signed-By in the sources.list(5) entry for '...'`。这只是提示，不是错误，但可通过显式指定密钥消除。

## 核心内容

- 该 Notice 由 apt 3.x 引入，用于提醒：源没有通过 `Signed-By` 指定专属密钥，只能依赖全局信任的
  `/etc/apt/trusted.gpg` 与 `/etc/apt/trusted.gpg.d/`。后者是已弃用的旧方式。
- 出现该提示时 `apt update` 仍会正常完成，包也正常校验，通常无需处理；介意的话按下文修复。
- 修复思路：给每个源显式绑定密钥。
  - deb822 格式（`.sources`）使用 `Signed-By:` 字段。
  - 单行格式（`.list`）使用 `[signed-by=/path/to/key.gpg]` 选项。
- Ubuntu 官方归档密钥就在 `/usr/share/keyrings/ubuntu-archive-keyring.gpg`，无需额外导入。

## 示例

以 `/etc/apt/sources.list.d/ubuntu.sources` 为例，原始内容缺少 `Signed-By`：

```text
Types: deb
URIs: http://archive.ubuntu.com/ubuntu/
Suites: resolute resolute-updates resolute-security
Components: main universe restricted multiverse
Architectures: amd64
```

备份并追加 `Signed-By`（备份放到 `sources.list.d/` 之外，见注意事项）：

```bash
sudo cp -a /etc/apt/sources.list.d/ubuntu.sources /etc/apt/ubuntu.sources.bak.$(date +%F)
sudo sed -i '/^Components:/a Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg' \
  /etc/apt/sources.list.d/ubuntu.sources
sudo apt update
```

修改后内容：

```text
Types: deb
URIs: http://archive.ubuntu.com/ubuntu/
Suites: resolute resolute-updates resolute-security
Components: main universe restricted multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
Architectures: amd64
```

参考：`.list` 格式的写法（`/etc/apt/sources.list.d/yazi.list`）：

```text
deb [signed-by=/usr/share/keyrings/yazi-keyring.gpg] https://yazi-rs.github.io/builds/ stable main
```

## 注意事项

- 只影响有提示的那一个源文件，逐个检查，不要给所有源盲目追加。
- 不要在 `/etc/apt/sources.list.d/` 内保留备份文件。apt 只识别 `.list` 和 `.sources` 后缀，
  遇到其他后缀（如 `ubuntu.sources.bak.2026-09-20`）会输出
  `Notice: Ignoring file '...' in directory '/etc/apt/sources.list.d/' as it has an invalid filename extension`。
  虽被忽略但会碍眼，因此备份放到 `/etc/apt/` 或 `/root/` 等目录下。
- 提示本身不是错误；若源使用的密钥不在官方 keyring 中，应指向该源自己的密钥文件。
- 本机环境：Ubuntu 26.04.1 LTS，apt 3.2.0。

## 相关知识

暂无

## 外部来源

- apt 官方文档：`man 5 sources.list`（`Signed-By` / `signed-by`）
