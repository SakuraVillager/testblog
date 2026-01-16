---
title: 将Fuwari的代码仓库与文章仓库分离
published: 2026-01-16
description: ""
image: ""
tags:
  - 博客
  - Fuwari
  - 教程
category: Fuwari
draft: false
lang: ""
---

# 前言

代码仓库与文章仓库在一起真的很难受。有时候改外观出错了想要回退之前的版本，结果发现写了几篇文章，回退的话文章也会没掉，就很难受。于是打算在发生这类问题之前解决掉。而最粗暴的方法就是将代码仓库和文章仓库分离。

# 基本思路

受[分离 hexo 的 posts 目录到独立仓库](https://zjlian.github.io/2022-07/987bb131dc00/)这篇文章的启发，最开始的思路是利用`Git子模块`，建立 2 个仓库，分别储存博客源码、博客文章，然后通过Git子模块将博客文章挂载到博客源码的posts目录下即可。但这样本地预览文章就会很麻烦。询问豆包后发现有更好的办法，那就是通过`GitHub Action`检测推送并合并到第三个仓库里。

由于我的托管服务时监听仓库的新提交来自动更新的，为了实现自动化，我们需要在博客源码和博客文章这两个仓库中任意一个仓库更新后都触发一次提交。我这边的思路是建立第三个仓库`Fuwari-Deploy`，利用`GitHub Actions`监测这两个仓库，当他们之中任意一个仓库有提交时，自动将两个仓库内容合并到第三个仓库`Fuwari-Deploy`，并完成一次推送，以触发边缘解析服务更新。
:::tip
由于是一边试验一边写博客的，所以最开始的git子模块方法我也写了一点，但由于失败了且没有完成，于是我放在本文的最后，如果又小伙伴有思路解决预览的问题也可以展示以下方法。
:::
废话不说，开始实践

# 构建 GitHub 仓库

这里需要构建 3 个仓库，分别储存博客源码、博客文章和合并后的文件。我这边建立了`Fuwari-Code`、`Fuwari-Posts`，后文分别用这三个仓库代表博客源码仓库、博客文章仓库和合并后的仓库。

1. 在本地拉取`Fuwari`仓库
2. 删除`.git`文件夹
3. 在`.gitignore`文件中添加一行`.\src\content\posts`，意思是忽略`posts`文件夹
4. 对`code`文件夹(根目录)进行git初始化并提交到`Fuwari-Code`仓库
5. 对`.\src\content\posts`文件夹进行git初始化并提交到`Fuwari-Posts`仓库

:::note
如果你的托管服务可以监听`Fuwari-Code`的指定分支来触发部署，那么不需要建立第三个仓库，你完全可以通过监听 code 仓库或者 posts 仓库的分支来完成合并。但如果不支持，请新建第三个仓库，用于储存合并后的文件。
:::

# 创建 token 和 GitHub App

打开GitHub-点击`头像`-`Settings`-`Developer settings`-`Personal access tokens`-`Tokens (classic)`，点击右上角`Generate new token`-`Generate new token (classic)`
![](./assets/将Fuwari的代码仓库与文章仓库分离/file-20260116190719543.png)
填写配置：

- `Note`：随便填
- `Expiration`：有效期，自己选一个比较合适的时间
- `Select scopes`：选择如下权限选项：- `repo`,用于跨仓库操作 - `workflow`，用于触发GitHub Actions
  配置完成后点击页面底部`Generate token`生成令牌，GitHub会生成一串字符串令牌。这个字符串**仅显示一次**，复制后可以保存到记事本等储存。
  回到`Developer settings`，点击`GitHub Apps`-`New GitHub App`
  ![](./assets/将Fuwari的代码仓库与文章仓库分离/file-20260116160439810.png)
  填写配置：
- `Register new GitHub App`:
  - `GitHub App name`：自定义
  - `Homepage URL`：随便填（如你的博客地址）
  - `Expire user authorization tokens`：自动过期，可选
- `Webhook`:
  - 取消勾选 “Active”
- `Repository permissions`：
  - `Contents`：选`Read & write`
  - `Actions`：选`Read & write`
- 最底部`Where can this GitHub App be installed?` : 选择`Only on this account`
  点击`Create GitHub App`，然后记录两个关键信息（后续要用）：
- 页面顶部的`App ID`
- 滚动到`Private keys` → `Generate a private key` → 下载生成的`.pem`文件
  ![](./assets/将Fuwari的代码仓库与文章仓库分离/file-20260116165635492.png)

# 安装 GitHub App 到你的仓库

回到 GitHub App 详情页 → 左侧`Install App` → 选择你的账号 → 点击`Install`
![](./assets/将Fuwari的代码仓库与文章仓库分离/file-20260116170004181.png)
勾选`Only select repositories`，点击`Select repository`选择安装范围，只选择所有私有的仓库，我这边选择`Fuwari-Posts`和`Fuwari-Deploy`，点击`Install`完成安装。
![](./assets/将Fuwari的代码仓库与文章仓库分离/file-20260116170423630.png)
安装完成后会转跳到安装详情页面，url应该类似于：

```text
https://github.com/settings/installations/xxxxxxxxx
```

记录最后的这一段数字，这就是`Installation ID`，后续会用到。

# 配置 GitHub Actions

:::attention

- 由于本人能力有限，下文中`GitHub Actions`的代码由`豆包`编写。请仔细斟酌。如有能力，建议自己编写。
- 有些人的仓库是私有仓库，有些人的仓库是公开仓库，为了适配所有仓库，我这里以`Fuwari-Deploy`私有，`Fuwari-Code`公开，`Fuwari-Posts`私有为例演示。
  :::

## 配置 Fuwari-Deploy 的 Actions

先配置App密钥。进入`Fuwari-Deploy`仓库-`Settings`-`Secrets and variables`-`Actions`，添加下面3个`Secrets`：

- `APP_ID`：之前记录的 GitHub App 的`App ID`
- `APP_PRIVATE_KEY`：打开之前下载的`.pem`文件，复制全部内容（包括`-----BEGIN RSA PRIVATE KEY-----`和`-----END RSA PRIVATE KEY-----`），粘贴到值中
- `INSTALLATION_ID`：之前记录的 GitHub App 的`Installation ID`
  ![](./assets/将Fuwari的代码仓库与文章仓库分离/file-20260116172148251.png)
  打开`Actions`-`New workflow`-`set up a workflow yourself`
  ![](./assets/将Fuwari的代码仓库与文章仓库分离/file-20260115195120925.png)
  ![](./assets/将Fuwari的代码仓库与文章仓库分离/file-20260115195204178.png)
  粘贴以下配置，命名为`sync-code-posts.yml`。**注意替换所有`你的 GitHub 用户名`为你的 GitHub 用户名**

```text
name: Auto Sync Code & Posts to Deploy

# 触发条件：接收跨仓库的repository_dispatch信号（sync_code/sync_posts）
on:
  repository_dispatch:
    types: [sync_code, sync_posts]

# 工作流权限：最小必要权限
permissions:
  contents: write
  actions: read

jobs:
  sync-and-deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 10 # 超时时间，避免卡死
    steps:
      # 步骤1：获取GitHub App临时令牌
      - name: Get GitHub App Access Token
        id: get_token
        uses: tibdex/github-app-token@v2
        with:
          app_id: ${{ secrets.APP_ID }}
          private_key: ${{ secrets.APP_PRIVATE_KEY }}
          installation_id: ${{ secrets.INSTALLATION_ID }}

      # 步骤2：拉取Deploy仓库自身
      - name: Checkout Deploy Repository
        uses: actions/checkout@v4
        with:
          repository: 你的 GitHub 用户名/Fuwari-Deploy
          ref: main
          token: ${{ steps.get_token.outputs.token }}
          path: deploy
          fetch-depth: 1

      # 步骤3：拉取公开Fuwari-Code仓库
      - name: Checkout Fuwari-Code Repository
        uses: actions/checkout@v4
        with:
          repository: 你的 GitHub 用户名/Fuwari-Code
          ref: main
          token: ${{ steps.get_token.outputs.token }} # 兼容私有场景，开源也可省略
          path: code
          fetch-depth: 1

      # 步骤4：拉取私有Fuwari-Posts仓库（需App授权）
      - name: Checkout Fuwari-Posts Repository
        uses: actions/checkout@v4
        with:
          repository: 你的 GitHub 用户名/Fuwari-Posts
          ref: main
          token: ${{ steps.get_token.outputs.token }}
          path: posts
          fetch-depth: 1

      # 步骤5：合并内容到Deploy仓库
      - name: Merge Code & Posts to Deploy
        run: |
          # 进入Deploy目录
          cd $GITHUB_WORKSPACE/deploy

          # 清空旧内容（避免冗余文件）
          rm -rf ./*

          # 复制Code仓库核心内容
          cp -r $GITHUB_WORKSPACE/code/* ./

          # 创建文章目录（防止目录不存在）
          mkdir -p ./src/content/posts

          # 复制Posts仓库内容到指定路径
          cp -r $GITHUB_WORKSPACE/posts/* ./src/content/posts/

          # 配置Git身份（Action必需）
          git config --global user.name "Fuwari-Deploy-Bot"
          git config --global user.email "fuwari-deploy-bot@github.com"

          # 检查是否有内容变化（避免空提交）
          if [ -n "$(git status --porcelain)" ]; then
            # 提交并推送
            git add .
            git commit -m "Auto Sync: Merge Code + Posts [Triggered by ${{ github.event.action }}]"
            git push origin main
            echo "✅ Deploy仓库合并推送成功！"
          else
            echo "ℹ️ 无内容变化，无需提交"
          fi

      # 步骤6：失败通知
      - name: Notify on Failure
        if: failure()
        run: |
          echo "❌ 同步失败！请检查："
          echo "1. GitHub App权限是否足够"
          echo "2. 仓库地址是否正确"
          echo "3. 文章路径是否匹配src/content/posts"
```

## 配置公开的 Fuwari-Code 的 Actions

在`Fuwari-Code`下的`Actions`，新建下面的配置并命名为`trigger-deploy.yml`。**注意替换所有`你的 GitHub 用户名`为你的 GitHub 用户名**

```text
name: 触发Deploy仓库同步
on:
  push:
    branches: [main]

jobs:
  trigger-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: 向Deploy仓库发送触发信号
        uses: peter-evans/repository-dispatch@v3
        with:
          repository: 你的 GitHub 用户名/Fuwari-Deploy
          event-type: sync_code
          client-payload: '{"ref": "${{ github.ref }}"}'
```

## 配置私有的 Fuwari-Posts 的 Actions

同样的，配置`Fuwari-Posts`下的`trigger-deploy.yml`。**注意替换所有`你的 GitHub 用户名`为你的 GitHub 用户名**。

```text
name: 触发Deploy仓库同步
on:
  push:
    branches: [main]

jobs:
  trigger-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: 向Deploy仓库发送触发信号
        uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.GH_PAT }}
          repository: 你的 GitHub 用户名/Fuwari-Deploy
          event-type: sync_posts
          client-payload: '{"ref": "${{ github.ref }}"}'
```

配置完后，更新任意一个仓库，来到`Actions`页面，可以看到推送自动触发了脚本，`Fuwari-Deploy`仓库成功更新了，从提交的名称`Triggered by sync_posts`也可以清晰的看到该合并提交是由于`Posts`仓库更新了。
![](./assets/将Fuwari的代码仓库与文章仓库分离/file-20260116195411994.png)
![](./assets/将Fuwari的代码仓库与文章仓库分离/file-20260116195459112.png)
![](./assets/将Fuwari的代码仓库与文章仓库分离/file-20260116195518145.png)

# 利用Git子模块实现同步(废弃)

:::CAUTION
本方法失败且未完成，仅提供思路借鉴。
:::

## 构建 GitHub 仓库

这里需要构建 2 个仓库，分别储存博客源码、博客文章。我这边建立了`Fuwari-Code`、`Fuwari-Posts`，后文分别用这两个仓库代表博客源码仓库和博客文章仓库。

1. 在本地拉取`Fuwari`仓库并重命名为`Fuwari-Code`
2. 删除`.git`文件以方便管理(可选)
3. 复制仓库中的`src\content\posts`文件夹到一个新的文件夹并重命名为`Fuwari-Posts`
4. 对`Fuwari-Posts`文件夹进行 git 初始化并推送到 Github 上的`Fuwari-Posts`仓库
5. 删除`Fuwari-Code`中的`src\content\posts`文件夹
6. 对`Fuwari-Code`文件夹进行 git 初始化并推送到`Fuwari-Code`仓库

## 在 Fuwari-Code 中子模块挂载 Fuwari-Posts

### 配置 SSH 密钥

#### 创建 SSH 密钥

在 Git Bash 中执行：

```bash
ssh-keygen -t ed25519 -C "你的GitHub邮箱地址"
```

执行后会出现以下提示，全部按回车即可，也可以按需自定义：

1. `Enter file in which to save the key` : 保存到默认路径
2. `Enter passphrase (empty for no passphrase)` : 不设置密码。设置则后每次推送都要输入
3. `Enter same passphrase again` : 确认无密码
   执行完成后，再执行 `ls -al ~/.ssh`，会看到新增两个文件：

- `id_ed25519`：SSH 私钥，保存在本地
- `id_ed25519.pub`：SSH 公钥，需要添加到 GitHub
  :::attention
  SSH 私钥千万不能泄露！！
  :::

#### 连接至 GitHub

在 Git Bash 中执行以下代码，将公钥复制到剪贴板：

```bash
cat ~/.ssh/id_ed25519.pub | clip
```

:::note
如果上述命令无效，手动打开 `C:\Users\你的用户名\.ssh\id_ed25519.pub` 文件（用记事本），全选复制内容。
:::
登录 GitHub，点击右上角`头像`-`Settings`
![](./assets/将Fuwari的代码仓库与文章仓库分离/file-20260113224931287.png)
选择 `SSH and GPG keys`-`New SSH key`
![](./assets/将Fuwari的代码仓库与文章仓库分离/file-20260113225014643.png)
`Title` 随便填，`Key type` 选择 `Authentication Key`，`Key` 框粘贴刚才复制的公钥内容，点击 `Add SSH key`，输入 GitHub 密码验证即可
![](./assets/将Fuwari的代码仓库与文章仓库分离/file-20260113225333699.png)

#### 验证 SSH 连接是否成功

在 Git Bash 中执行：

```bash
ssh -T git@github.com
```

:::tip
首次执行会提示 `Are you sure you want to continue connecting`，输入 `yes` 并回车。
:::
如果输出 `Hi 你的GitHub用户名! You've successfully authenticated, but GitHub does not provide shell access.`，说明 SSH 认证成功。

### 添加并提交 Git 子模块

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

到这里分离的工作就完成了。后续写文章时只需要专注于文章仓库即可，大家可以摆出五花八门的写作姿势。例如我就是利用`Obsidian`撰写文章。
