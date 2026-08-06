## 概述

本文记录在安卓手机上通过 Termux 终端环境安装 Git，并配合 Obsidian 实现笔记同步的完整流程。

## 一、安装 Termux

### 1.1 下载 Termux

- Google Play 搜索 Termux 安装（部分地区可能找不到）
- 备选方案：从 F-Droid 下载 APK 安装：https://f-droid.org/packages/com.termux/

### 1.2 初始化 Termux

打开 Termux 后，先更新软件源：

```bash
pkg update && pkg upgrade
```

### 1.3 授权存储访问

让 Termux 访问手机存储（包括 Documents、Download 等目录）：

```bash
termux-setup-storage
```

执行后会弹出权限请求，点击允许。授权后的存储映射关系：

| Termux 路径 | 对应手机目录 |
|-------------|-------------|
| `~/storage/shared` | 手机内部存储根目录 |
| `~/storage/shared/Documents` | Documents 文件夹 |
| `~/storage/shared/Download` | Download 文件夹 |
| `~/storage/shared/DCIM` | 相册 |
| `~/storage/downloads` | 下载目录 |

## 二、安装 Git

### 2.1 安装 Git

```bash
pkg install git
```

### 2.2 配置 Git 用户信息

```bash
git config --global user.name "你的用户名"
git config --global user.email "你的邮箱"
```

## 三、配置 SSH 密钥

### 3.1 生成个人 SSH 密钥

```bash
ssh-keygen -t ed25519 -C "你的邮箱"
```

一路回车使用默认路径。生成后会在 `~/.ssh/` 下创建：

- `id_ed25519` — 私钥（切勿泄露）
- `id_ed25519.pub` — 公钥

### 3.2 安装 ssh-agent 服务

```bash
pkg install termux-services
sv-enable ssh-agent
```

### 3.3 查看公钥

```bash
cat ~/.ssh/id_ed25519.pub
```

输出格式类似：

```
ssh-ed25519 AAAAC3... 你的邮箱
```

### 3.4 添加公钥到 GitHub

1. 复制上面输出的公钥内容（从 `ssh-ed25519` 开始到邮箱结束）
2. 打开 GitHub SSH 设置页面：https://github.com/settings/ssh/new
3. 填写表单：
   - **Title**：填个名字，如「Termux-Android」
   - **Key type**：选「Authentication Key」
   - **Key**：粘贴公钥
4. 点击「Add SSH Key」保存

### 3.5 测试连接

```bash
ssh -T git@github.com
```

看到 `Hi 用户名! You've successfully authenticated` 即表示配置成功。

## 四、克隆仓库

### 4.1 SSH 方式克隆（推荐）

```bash
git clone git@github.com:用户名/仓库名.git
```

### 4.2 HTTPS 方式克隆

```bash
git clone https://github.com/用户名/仓库名.git
```

HTTPS 方式需要输入 GitHub 密码或 Personal Access Token。Token 生成地址：https://github.com/settings/tokens

## 五、修改 Termux 字体

### 5.1 通过配置文件调整大小

```bash
mkdir -p ~/.termux
echo "font-size=18" >> ~/.termux/termux.properties
termux-reload-settings
```

将 `18` 改为合适的字号。

### 5.2 通过 Termux:Style 插件（推荐）

在应用商店或 F-Droid 安装 **Termux:Style**，打开后可以选择字体样式、大小和主题颜色。

### 5.3 安装标准等宽字体（解决字体粗细不均）

Termux 默认字体会出现粗细不一致的问题，推荐安装 **JetBrains Mono** 等宽字体，渲染均匀清晰。

**方式一：在 Termux 中直接下载安装（推荐）**

```bash
pkg install wget unzip
wget https://github.com/JetBrains/JetBrainsMono/releases/download/v2.304/JetBrainsMono-2.304.zip
unzip JetBrainsMono-2.304.zip
cp fonts/ttf/JetBrainsMono-Regular.ttf ~/.termux/font.ttf
termux-reload-settings
rm -rf fonts/ JetBrainsMono-2.304.zip
```

**方式二：从手机存储安装**

如果下载速度太慢，可以用手机浏览器下载字体文件，然后从存储复制：

```bash
cp ~/storage/shared/Download/JetBrainsMono-Regular.ttf ~/.termux/font.ttf
termux-reload-settings
```

**其他推荐字体：**

| 字体名称 | 特点 | 下载地址 |
|---------|------|---------|
| JetBrains Mono | 编程专用，连字符支持好 | https://github.com/JetBrains/JetBrainsMono/releases |
| Fira Code | 经典编程字体，连字符丰富 | https://github.com/tonsky/FiraCode/releases |
| Cascadia Code | 微软出品，Windows Terminal 默认 | https://github.com/microsoft/cascadia-code/releases |

下载任意字体的 `.ttf` 文件，复制到 `~/.termux/font.ttf` 并执行 `termux-reload-settings` 即可生效。

## 六、Obsidian 配合 Git 同步

### 6.1 安装 Obsidian

在应用商店搜索 Obsidian 安装。

### 6.2 安装 Git 插件

在 Obsidian 中：设置 → 第三方插件 → 关闭安全模式 → 浏览社区插件 → 搜索「Git」→ 安装并启用。

### 6.3 配置 Git 插件

进入 Git 插件设置：

- **Remote URL**：填入 HTTPS 地址（Obsidian Git 插件不支持 SSH 协议）
  ```
  https://github.com/用户名/仓库名.git
  ```
- **Author name**：填你的 Git 用户名
- **Author email**：填你的 Git 邮箱

> **注意**：Obsidian 的 Git 插件不支持 SSH 协议，如果填入 SSH 地址会报错 `uses an unrecognized transport protocol: "ssh"`。必须使用 HTTPS 地址。

### 6.4 HTTPS 认证

HTTPS 方式需要密码认证，建议使用 Personal Access Token 代替密码：

1. 访问 https://github.com/settings/tokens
2. 点击「Generate new token」
3. 勾选 `repo` 权限
4. 生成后复制 Token
5. 在 Obsidian Git 推送时，密码栏填入 Token

## 七、常用命令速查

### Termux 相关

| 命令 | 说明 |
|------|------|
| `pkg update` | 更新软件源 |
| `pkg install <包名>` | 安装软件包 |
| `termux-setup-storage` | 授权存储访问 |
| `termux-reload-settings` | 重载 Termux 设置 |

### Git 相关

| 命令 | 说明 |
|------|------|
| `git clone <地址>` | 克隆仓库 |
| `git add .` | 添加所有变更 |
| `git commit -m "说明"` | 提交变更 |
| `git push origin main` | 推送到远程 |
| `git pull` | 拉取远程更新 |
| `git status` | 查看变更状态 |

### SSH 相关

| 命令 | 说明 |
|------|------|
| `ssh-keygen -t ed25519 -C "邮箱"` | 生成 SSH 密钥 |
| `cat ~/.ssh/id_ed25519.pub` | 查看公钥 |
| `ssh -T git@github.com` | 测试 GitHub 连接 |
| `sv-enable ssh-agent` | 启用 ssh-agent 服务 |

## 八、常见问题

### Q: Obsidian Git 插件报 SSH 协议错误？

Obsidian 的 Git 插件不支持 SSH，必须使用 HTTPS 地址。在插件设置中将 Remote URL 改为 `https://github.com/用户名/仓库名.git`。

### Q: Termux 字体太小？

编辑 `~/.termux/termux.properties`，设置 `font-size=18`（或其他数值），然后执行 `termux-reload-settings`。或者安装 Termux:Style 插件。

### Q: 如何找到 Termux 的设置？

三种方式：
1. 从手机顶部下拉通知栏，点击 Termux 通知条
2. 从屏幕左边缘向右滑动，在侧边菜单中选择「Settings」
3. 安装 Termux:Style 插件获得更多设置选项

### Q: 克隆私有仓库提示认证失败？

使用 Personal Access Token 代替密码。访问 https://github.com/settings/tokens 生成 Token，克隆时密码填 Token。

### Q: 如何进入手机的 Documents 目录？

先执行 `termux-setup-storage` 授权存储，然后 `cd ~/storage/shared/Documents`。
