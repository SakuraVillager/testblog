---
title: 将Fuwari的代码仓库与文章仓库分离
published: 2026-01-13
description: ''
image: ''
tags: ["博客", "Fuwari", "教程"]
category: 'Fuwari'
draft: true 
lang: ''
---
# 前言
当

# 基本思路

废话不说，开始实践
# 构建GitHub仓库
这里需要构建3个仓库，分别储存博客源码、博客文章、博客源码和文章的合体。我这边建立了`Fuwari-Code`、`Fuwari-Posts`，后文分别用这两个仓库代表博客源码仓库和博客文章仓库。
1. 在本地拉取`Fuwari`仓库并重命名为`Fuwari-Code`
2. 删除`.git`文件以方便管理(可选)
3. 复制仓库中的`src\content\posts`文件夹到一个新的文件夹并重命名为`Fuwari-Posts`
4. 对`Fuwari-Posts`文件夹进行git初始化并推送到Github上的`Fuwari-Posts`仓库
5. 删除`Fuwari-Code`中的`src\content\posts`文件夹
6. 对`Fuwari-Code`文件夹进行git初始化并推送到`Fuwari-Code`仓库
# 在Fuwari-Code中子模块挂载Fuwari-Posts
## 配置 SSH 密钥
### 创建 SSH 密钥
在 Git Bash 中执行：
```bash
ssh-keygen -t ed25519 -C "你的GitHub邮箱地址"
```

执行后会出现以下提示，全部按回车即可，也可以按需自定义：
1. `Enter file in which to save the key` : 保存到默认路径
2. `Enter passphrase (empty for no passphrase)`  : 不设置密码。设置则后每次推送都要输入
3. `Enter same passphrase again` : 确认无密码
执行完成后，再执行 `ls -al ~/.ssh`，会看到新增两个文件：
- `id_ed25519`：SSH 私钥，保存在本地
- `id_ed25519.pub`：SSH 公钥，需要添加到 GitHub
:::[!attention]
SSH私钥千万不能泄露！！
:::
### 连接至GitHub
在 Git Bash 中执行以下代码，将公钥复制到剪贴板：
```bash
cat ~/.ssh/id_ed25519.pub | clip
```

:::[!note]
如果上述命令无效，手动打开 `C:\Users\你的用户名\.ssh\id_ed25519.pub` 文件（用记事本），全选复制内容。
:::
登录 GitHub，点击右上角`头像`-`Settings`
![](./assets/将Fuwari的代码仓库与文章仓库分离/file-20260113224931287.png)
选择 `SSH and GPG keys`-`New SSH key`
![](./assets/将Fuwari的代码仓库与文章仓库分离/file-20260113225014643.png)
`Title` 随便填，`Key type` 选择 `Authentication Key`，`Key` 框粘贴刚才复制的公钥内容，点击 `Add SSH key`，输入 GitHub 密码验证即可
![](./assets/将Fuwari的代码仓库与文章仓库分离/file-20260113225333699.png)
### 验证 SSH 连接是否成功
在 Git Bash 中执行：
```bash
ssh -T git@github.com
```
:::[!tip]
首次执行会提示 `Are you sure you want to continue connecting`，输入 `yes` 并回车。
:::
如果输出 `Hi 你的GitHub用户名! You've successfully authenticated, but GitHub does not provide shell access.`，说明 SSH 认证成功。
### 将私钥添加到 Windows SSH 代理(可选)
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_xxx
```

## 添加并提交 Git 子模块
回到你的`Fuwari-Code`文件夹，**用 Git Bash 执行**：
```bash
cd /xx/xx/Fuwari-code
git submodule add -b main git@github.com:你的GitHub账户名/Fuwari-Posts.git src/content/posts
```
正常添加后提交子模块配置
```bash
git add .gitmodules src/content/posts
git commit -m "feat: 添加文章仓库作为子模块，实现实时预览"
git push origin main
```