# Fork 仓库发布 Release 与 Homebrew Tap 指南

本指南说明如何在你的 fork 上发布 Release，并通过自建 Homebrew tap 提供 `brew install` 安装。

## 一、整体流程

1. **GitHub Actions**：在 fork 里推送 `v*` tag 时，自动构建多平台二进制、打 GitHub Release、推 GHCR 镜像，并**更新你的 Homebrew tap 仓库**。
2. **Homebrew Tap**：一个独立的 GitHub 仓库（如 `你的用户名/homebrew-tap`），存放 Formula；用户通过 `brew tap 你的用户名/tap` 和 `brew install 你的用户名/tap/kustomizer` 安装。

配置已按「当前仓库 owner」写好，fork 后无需改仓库名，只需准备好 Secrets 和 tap 仓库即可。

---

## 二、GitHub 配置

### 2.1 所需 Secrets

在 **fork 的仓库** Settings → Secrets and variables → Actions 中配置：

| Secret | 说明 | 必填 |
|--------|------|------|
| `PUSH_GITHUB_TOKEN` | GitHub PAT（Personal Access Token），需勾选 `repo`、`write:packages`；用于推送到 **tap 仓库** 和登录 ghcr.io | 是 |
| `COSIGN_PASSWORD` | cosign 私钥密码（详见下方 [2.3 COSIGN 如何配置](#23-cosign-如何配置)；不做签名可跳过） | 做签名时必填 |
| `COSIGN_KEY` | cosign 私钥**完整文件内容**（同上；不做签名可跳过） | 做签名时必填 |

- **GITHUB_TOKEN** 无需配置，由 Actions 自动注入，且**只能操作当前仓库**（即运行 workflow 的 kustomizer fork）。用它创建 Release、上传当前仓库的资产没问题。
- **推送到 tap 不能用 GITHUB_TOKEN**：tap 是**另一个仓库**（如 `你的用户名/homebrew-tap`）。GitHub 限制 GITHUB_TOKEN 只能访问「当前仓库」，不能对其它 repo 做 push。因此必须用你有 `repo` 权限的 PAT（`PUSH_GITHUB_TOKEN`）作为 `HOMEBREW_TAP_GITHUB_TOKEN`，GoReleaser 才能把 Formula 推送到 homebrew-tap。

### 2.2 创建 PAT（PUSH_GITHUB_TOKEN）

1. GitHub → Settings → Developer settings → Personal access tokens → Generate new token (classic)。
2. 勾选：`repo`（完整）、`write:packages`、`read:packages`（如需拉取私有镜像）。
3. 将生成的 token 存为 fork 仓库的 Secret：`PUSH_GITHUB_TOKEN`。

### 2.3 COSIGN 如何配置

Release 流程会用 [Cosign](https://github.com/sigstore/cosign) 对**校验和文件**和**容器镜像**做签名，便于用户验证下载未被篡改。按下面步骤生成密钥并填入 Secrets 即可。

#### 步骤一：安装 Cosign

```bash
brew install cosign
```

#### 步骤二：生成密钥对

在本地执行（会提示你输入私钥密码，请牢记）：

```bash
cosign generate-key-pair
```

完成后当前目录会多出两个文件：

- **cosign.key**：私钥（**绝不要提交到 Git**，只放在本机和 GitHub Secret 里）
- **cosign.pub**：公钥（可公开，用户用它验证签名）

#### 步骤三：填入 GitHub Secrets

1. **COSIGN_PASSWORD**  
   填你在「步骤二」里为私钥设置的密码。

2. **COSIGN_KEY**  
   填**私钥文件的 Base64 编码**（推荐，可避免 GitHub 多行 Secret 破坏换行导致 `invalid pem block`）。在终端执行：
   ```bash
   base64 -i cosign.key | tr -d '\n' | pbcopy   # macOS 复制到剪贴板
   # 或直接输出到终端，再粘贴到 Secret：
   base64 -i cosign.key | tr -d '\n'
   ```
   将输出的**一整行**粘贴到仓库 Secret `COSIGN_KEY` 的值里。  
   （若你已用原始 PEM 内容作为 Secret 且报错 `invalid pem block`，请改为上述 Base64 方式重新保存 Secret。）

#### 步骤四：公钥给用户验证（可选）

把 `cosign.pub` 放到用户能访问的地方（例如仓库的 Release 附件、或单独一个 gist/文档），用户验证时用：

```bash
# 验证某次 Release 的 checksums
cosign verify-blob --key cosign.pub --signature checksums.txt.sig checksums.txt

# 验证容器镜像
cosign verify --key cosign.pub ghcr.io/你的用户名/kustomizer:v1.0.0
```

#### 若不想做签名（跳过 COSIGN）

如果暂时不需要签名，可以关掉 GoReleaser 的签名步骤，并让 workflow 不再依赖 COSIGN 密钥：

1. 在 **`.goreleaser.yml`** 里注释或删掉 `signs:` 整块和 `docker_signs:` 整块（或参考仓库内「跳过签名」的说明）。
2. 在 **`.github/workflows/release.yaml`** 里注释或删掉「Write signing key to tmp」这一步，以及 `env` 里的 `COSIGN_PASSWORD`。

这样就不需要配置 `COSIGN_KEY` 和 `COSIGN_PASSWORD`，Release 和 tap 仍会照常发布，只是没有签名可验证。

---

## 三、创建 Homebrew Tap 仓库

Tap 是一个「只放 Formula 的 GitHub 仓库」，命名约定为 `homebrew-tap`，对应 brew 命令中的 `tap` 名为 `你的用户名/tap`。

### 3.1 创建仓库

1. 在 GitHub 新建仓库，名称：**homebrew-tap**（必须叫这个，这样 `brew tap 你的用户名/tap` 才会找到）。
2. 可见性选 Public，不勾选「Add a README」也可以（GoReleaser 会往里面推 Formula）。

### 3.2 可选：先手动建一个 Formula（便于测试 tap）

若你想先有一个可用的 tap，可以手动建一个 Formula，之后 GoReleaser 会在每次 release 时覆盖更新该文件。

在 **homebrew-tap** 仓库中创建目录和文件：

- 仓库根目录下创建：**Formula/kustomizer.rb**

内容示例（把 `YOUR_GITHUB_USERNAME` 换成你的 GitHub 用户名，`x.x.x` 换成当前版本号）：

```ruby
# Formula/kustomizer.rb
class Kustomizer < Formula
  desc "Kustomizer CLI"
  homepage "https://github.com/YOUR_GITHUB_USERNAME/kustomizer"
  version "x.x.x"

  on_macos do
    on_intel do
      url "https://github.com/YOUR_GITHUB_USERNAME/kustomizer/releases/download/v#{version}/kustomizer_#{version}_darwin_amd64.tar.gz"
      sha256 "REPLACE_WITH_REAL_SHA256"
    end
    on_arm do
      url "https://github.com/YOUR_GITHUB_USERNAME/kustomizer/releases/download/v#{version}/kustomizer_#{version}_darwin_arm64.tar.gz"
      sha256 "REPLACE_WITH_REAL_SHA256"
    end
  end

  on_linux do
    on_intel do
      url "https://github.com/YOUR_GITHUB_USERNAME/kustomizer/releases/download/v#{version}/kustomizer_#{version}_linux_amd64.tar.gz"
      sha256 "REPLACE_WITH_REAL_SHA256"
    end
    on_arm do
      url "https://github.com/YOUR_GITHUB_USERNAME/kustomizer/releases/download/v#{version}/kustomizer_#{version}_linux_arm64.tar.gz"
      sha256 "REPLACE_WITH_REAL_SHA256"
    end
  end

  depends_on "cosign"
  depends_on "diffutils" => :optional

  def install
    bin.install "kustomizer"
    bash_completion.install Utils.safe_popen_read(bin/"kustomizer", "completion", "bash") => "kustomizer"
    zsh_completion.install Utils.safe_popen_read(bin/"kustomizer", "completion", "zsh") => "_kustomizer"
    fish_completion.install Utils.safe_popen_read(bin/"kustomizer", "completion", "fish") => "kustomizer.fish"
  end

  test do
    system "#{bin}/kustomizer", "--version"
  end
end
```

首次发布后，从 GitHub Release 页的 asset 链接下载对应 tar.gz，在本地执行 `shasum -a 256 文件名` 得到 sha256，替换上面 `REPLACE_WITH_REAL_SHA256`。**之后每次用 tag 发布，GoReleaser 会自动更新该 Formula，无需再改。**

---

## 四、发布 Release 的步骤

1. 在 fork 仓库中确认代码已提交并推送到默认分支（如 `main`）。
2. 打 tag 并推送（版本号按语义化版本，例如）：
   ```bash
   git tag v1.0.0
   git push origin v1.0.0
   ```
3. 打开 fork 仓库的 **Actions** 页，会看到 **release** workflow 运行。
4. 完成后：
   - **Releases** 页会出现该版本的 Release 和二进制/源码等资产；
   - **Packages**（或 ghcr.io）会出现对应容器镜像；
   - 你的 **homebrew-tap** 仓库中 `Formula/kustomizer.rb` 会被 GoReleaser 自动更新（若已存在则覆盖）。

---

## 五、用户安装方式（你的 tap）

用户安装你 fork 的 kustomizer：

```bash
brew tap 你的用户名/tap
brew install 你的用户名/tap/kustomizer
```

例如你的 GitHub 用户名为 `seanly`：

```bash
brew tap seanly/tap
brew install seanly/tap/kustomizer
```

---

## 六、本仓库已做的配置说明

- **`.github/workflows/release.yaml`**
  - 触发器：`push` 到 tag `v*`。
  - 使用 `github.repository_owner` 登录 ghcr.io，镜像推送到你的 GHCR（如 `ghcr.io/你的用户名/kustomizer`）。
  - 将 `PUSH_GITHUB_TOKEN` 传给 GoReleaser 作为 `HOMEBREW_TAP_GITHUB_TOKEN`，用于推送到你的 homebrew-tap。

- **`.goreleaser.yml`**
  - `brews.tap.owner`：使用 `GITHUB_REPOSITORY_OWNER`（Actions 自动注入），即当前仓库 owner，fork 后即为你的用户名。
  - `brews.tap.name`：`homebrew-tap`（对应 `brew tap 用户名/tap`）。
  - Docker 相关镜像前缀：同样使用 `GITHUB_REPOSITORY_OWNER`，推到你自己的 ghcr.io。

因此你**不需要**在仓库里改任何用户名，只要：

1. 在 fork 里配置好 `PUSH_GITHUB_TOKEN`、`COSIGN_PASSWORD`（及可选的 `COSIGN_KEY`），  
2. 创建并准备好 **homebrew-tap** 仓库（可先空仓库，或按上面 3.2 先建一个 Formula），  
3. 推送 `v*` tag，

即可在 fork 的 Release 中发布，并通过 `brew tap 你的用户名/tap && brew install 你的用户名/tap/kustomizer` 安装。
