+++
date = '2026-01-14T02:43:43+08:00'
draft = false
title = 'Windows 11 创建 Hugo 网站'

+++

官方教程不允许使用 CMD、Windows PowerShell 执行各个命令，建议使用 PowerShell 执行命令。Anaconda Powershell Prompt 应该是基于 PowerShell 开发的，因此，本文的命令均在 Anaconda Powershell Prompt 中执行。

## 安装 Hugo

安装 Hugo 前需要安装 Git、Go、Dart Sass。

* 安装 [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

  无论是克隆 Hugo 的 GitHub 仓库，还是基于 Github Page 发布网站，都需要使用 Git。

* 安装 [Go](https://go.dev/doc/install)

  Hugo 是基于 Go 语言开发，Go 作为一种高效的编程语言，使得 Hugo 在速度上远超 Hexo 等静态网站搭建技术。

* 安装 [Dart Sass](https://gohugo.io/functions/css/sass/#dart-sass)

  Dart Sass 是 Hugo 开发的网页渲染插件，可以通过 Scoop 和 Chocolatey 两个 Windows 安装器安装，这里选择使用 Scoop 安装。

  * 安装 [Scoop](https://scoop.sh/#/)

    首先 cd 到 C:\ 路径，在 C:\ 路径下执行下述命令。

    ```shell
    Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
    Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression
    ```

    按照上述官方命令安装时出现了 PowerShell 禁止运行脚本的报错，使用 [以下命令](https://blog.csdn.net/tongxin_tongmeng/article/details/128150906) 替代上面的第一条命令解决该问题。

    ```shell
    Set-ExecutionPolicy RemoteSigned -Scope Process
    Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
    Set-ExecutionPolicy RemoteSigned -Scope LocalMachine
    ```

  安装 Scoop 后，可以执行下述命令安装 Dart Sass。

  ```shell
  scoop install sass
  ```

安装 Git、Go、Dart Sass 后

1. 在 [Hugo GitHub 发布页](https://github.com/gohugoio/hugo/releases/tag/v0.154.5) 下载并解压 [hugo_0.154.5_windows-amd64.zip](https://github.com/gohugoio/hugo/releases/download/v0.154.5/hugo_0.154.5_windows-amd64.zip) 。
2. 将解压得到的文件 hugo.exe 移动到安装路径，这里选择的安装路径是 D:\Program Files\hugo，并将该路径添加到系统路径变量中。

此时，重新打开 PowerShell 并执行 `hugo version`，若输出 Hugo 的版本号等信息，则表明 Hugo 安装成功。

## 网站创建

成功安装 Hugo 后，就可以创建第一个网站了。

```shell
hugo new site quickstart
cd quickstart
git init
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke
echo "theme = 'ananke'" >> hugo.toml
hugo server
```

* 报错 1：

  ```shell
  ERROR command error: failed to load config: "D:\Program Files\hugo\quickstart\hugo.toml:4:2": unmarshal failed: toml: expected character =
  ```

  直接将 `echo "theme = 'ananke'" >> hugo.toml` 复制到 PowerShell 中执行会出现格式错误，修改 hugo.toml 


* 报错 2：

  ```shell
  ERROR error building site: render: [en v1.0.0 guest] failed to render pages: render of "/tags" failed: "D:\Program Files\hugo\quickstart\themes\ananke\layouts\baseof.html:26:15": execute of template failed: template: taxonomy.html:26:15: executing "taxonomy.html" at <partials.Include>: error calling Include: "D:\Program Files\hugo\quickstart\themes\ananke\layouts\_partials\site-style.html:2:32": execute of template failed: template: _partials/site-style.html:2:32: executing "_partials/site-style.html" at <.RelPermalink>: error calling RelPermalink: TOCSS: failed to transform "/ananke/css/main.css" (text/css). Check your Hugo installation; you need the extended version to build SCSS/SASS with transpiler set to 'libsass'.: this feature is not available in your current Hugo version, see https://goo.gl/YMrWcn for more information
  ```

  这是因为通过 [hugo_0.154.5_windows-amd64.zip](https://github.com/gohugoio/hugo/releases/download/v0.154.5/hugo_0.154.5_windows-amd64.zip) 安装的不是 extended version，不能对 anake 主题进行渲染。执行命令 `scoop install hugo-extended` 直接从 scoop 中安装 hugo-extended 并删除系统路径中的 hugo 路径即可。

## 内容添加

向网站中添加一个新文章：

```shell
hugo new content content/posts/my-first-post.md
```

在 content/posts 目录下出现了 my-first-post.md 文件，下述是该文件中的内容。其中 draft=true 表示这个文章不会被发表到网站上。

```markdown
+++
title = 'My First Post'
date = 2024-01-14T07:07:07+01:00
draft = true
+++
```

通过下述命令查看含有 draft 文章的网站，当决定发表该文章后，将 draft 改为 false。

```shell
hugo server --buildDrafts
hugo server -D
```

## 网站配置

通过目录下的 hugo.toml 配置网站，其中的 baseURL 设置为生产网站。

```toml
baseURL = 'https://example.org/'
languageCode = 'en-us'
title = 'My New Hugo Site'
theme = 'ananke'
```

## 网站发布

网站发布指的是 Hugo 在 public 目录下创建静态网站所需的全部文件（包括 HTML 文件、CSS 文件、Javascript 文件、图片得等等），命令如下：

```shell
hugo
```

## 网站部署

TODO













