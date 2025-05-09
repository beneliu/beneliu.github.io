# Hugo学习


{{&lt; admonition type=info title=&#34;内容简介&#34; open=true &gt;}}
简单介绍 **Hugo** 的一些核心概念和原理，包括核心工作流程、构建过程解析、目录结构、如何安装等。
{{&lt; /admonition &gt;}}

## Hugo简介

**Hugo**是一个用 Go 语言编写的开源静态网站生成器，具有以下核心特点：
- ⚡ **极速构建**：平均构建速度 &lt; 1秒/千页
- 📂 **内容优先**：支持 Markdown 格式内容管理
- 🎨 **主题生态**：丰富的[主题库](https://themes.gohugo.io/) 
- 🖥️ **跨平台**：支持 Windows/macOS/Linux
- 🌍 **多语言**：内置国际化（i18n）支持

典型应用场景：
- 个人/技术博客
- 项目文档网站
- 企业官网
- 作品集展示

## Hugo实现博客建站的原理

### 核心工作流程

{{&lt; mermaid &gt;}}
graph LR
    A[Markdown 内容] --&gt; B[Hugo 引擎]
    C[模板文件] --&gt; B
    D[静态资源] --&gt; B
    B --&gt; E[静态HTML网站]
{{&lt; /mermaid &gt;}}

构建过程：
1. 读取配置文件确定构建参数
2. 解析 Markdown 内容为结构化数据
3. 将内容数据注入模板文件
4. 生成静态 HTML/CSS/JS 文件
5. 输出到 /public 目录

### 目录结构

每个 Hugo 项目都是一个目录，其中的子目录贡献于站点的内容、结构、行为和呈现。在创建新站点时，Hugo 会生成一个项目骨架。例如，运行以下命令：
```bash
hugo new site my-site
```
会创建以下目录结构：
```bash
my-site/
├── archetypes/
│   └── default.md
├── assets/
├── content/
├── data/
├── i18n/
├── layouts/
├── static/
├── themes/
└── hugo.toml         &lt;-- 站点配置
```
根据需要，也可以将站点配置组织到子目录中：
```bash
my-site/
├── archetypes/
│   └── default.md
├── assets/
├── config/           &lt;-- 站点配置
│   └── _default/
│       └── hugo.toml
├── content/
├── data/
├── i18n/
├── layouts/
├── static/
└── themes/
```
本站点就是采用的这种方式，将配置文件放置在 `config` 目录下，以 `_default` 为子目录，以 `hugo.toml` 为文件名。Hugo默认的站点目录结构如下：
- `archetypes`：原型目录，用于定义各种类型的内容模板。原型匹配顺序是优先本站点内，其次再到主题内查找。
- `assets`：资产目录，用于放置 CSS，JavaScript 等全局资源库。
- `config`：配置文件目录，主配置文件 hugo.yaml，支持多文件配置、多环境配置。
- `content`：内容目录，用于放置文章、分类、标签等内容页面。
- `data`：数据目录，用于存取自定义配置数据。
- `i18n`：国际化目录，用于存放多语言配置文件和博客页面。
- `layouts`：布局目录，用于存放模板文件。
- `public`：部署目录，用于存放 Hugo 构建的静态站点文件。
- `resources`：资源目录，包含 Hugo 资产构建流水线产生的可缓存文件，如 CSS、图片等。
- `static`：静态资源目录，用于放置静态资源文件，如图片、CSS、JavaScript 等。该目录下的文件会被直接拷贝到站点部署目录。
- `themes`：主题目录，用于存放主题文件。

具体每个目录的作用可以参考官方文档：[目录结构 | Hugo官方文档](https://hugo.opendocs.io/zh-cn/getting-started/directory-structure/)

## Hugo安装与常用命令

### Hugo安装
```bash
macOS: brew install hugo
Linux: snap install hugo
Windows: choco install hugo
```
博主用的是Windows，但是没有选择`choco`进行安装，而是直接下载二进制zip包进行安装，因为可以装最新版，也更方便控制安装路径。`hugo`命令安装包下载地址：[Hugo Releases | GitHub](https://github.com/gohugoio/hugo/releases)。下载后解压，将解压后的文件夹添加到环境变量中，就可以使用`hugo`命令了。至于怎么添加环境变量，这里就不赘述了。

### hugo命令行

```bash
hugo help

hugo is the main command, used to build your Hugo site.

Hugo is a Fast and Flexible Static Site Generator
built with love by spf13 and friends in Go.

Complete documentation is available at https://gohugo.io/.

Usage:
  hugo [flags]
  hugo [command]

Available Commands:
  build       Build your site
  completion  Generate the autocompletion script for the specified shell
  config      Display site configuration
  convert     Convert front matter to another format
  env         Display version and environment info
  gen         Generate documentation and syntax highlighting styles
  help        Help about any command
  import      Import a site from another system
  list        List content
  mod         Manage modules
  new         Create new content
  server      Start the embedded web server
  version     Display version

Flags:
  -b, --baseURL string             hostname (and path) to the root, e.g. https://spf13.com/
  -D, --buildDrafts                include content marked as draft
  -E, --buildExpired               include expired content
  -F, --buildFuture                include content with publishdate in the future
      --cacheDir string            filesystem path to cache directory
      --cleanDestinationDir        remove files from destination not found in static directories
      --clock string               set the clock used by Hugo, e.g. --clock 2021-11-06T22:30:00.00&#43;09:00
      --config string              config file (default is hugo.yaml|json|toml)
      --configDir string           config dir (default &#34;config&#34;)
  -c, --contentDir string          filesystem path to content directory
  -d, --destination string         filesystem path to write files to
      --disableKinds strings       disable different kind of pages (home, RSS etc.)
      --enableGitInfo              add Git revision, date, author, and CODEOWNERS info to the pages
  -e, --environment string         build environment
      --forceSyncStatic            copy all files when static is changed.
      --gc                         enable to run some cleanup tasks (remove unused cache files) after the build
  -h, --help                       help for hugo
  -h, --help                       help for hugo
      --ignoreCache                ignores the cache directory
      --ignoreVendorPaths string   ignores any _vendor for module paths matching the given Glob pattern
  -l, --layoutDir string           filesystem path to layout directory
      --logLevel string            log level (debug|info|warn|error)
      --minify                     minify any supported output format (HTML, XML etc.)
      --minify                     minify any supported output format (HTML, XML etc.)
      --noBuildLock                don&#39;t create .hugo_build.lock file
      --noChmod                    don&#39;t sync permission mode of files
      --noTimes                    don&#39;t sync modification time of files
      --noBuildLock                don&#39;t create .hugo_build.lock file
      --noChmod                    don&#39;t sync permission mode of files
      --noTimes                    don&#39;t sync modification time of files
      --panicOnWarning             panic on first WARNING log
      --panicOnWarning             panic on first WARNING log
      --poll string                set this to a poll interval, e.g --poll 700ms, to use a poll based approach to watch for file system changes
      --poll string                set this to a poll interval, e.g --poll 700ms, to use a poll based approach to watch for file system changes
      --printI18nWarnings          print missing translations
      --printI18nWarnings          print missing translations
      --printMemoryUsage           print memory usage to screen at intervals
      --printMemoryUsage           print memory usage to screen at intervals
      --printPathWarnings          print warnings on duplicate target paths etc.
      --printPathWarnings          print warnings on duplicate target paths etc.
      --printUnusedTemplates       print warnings on unused templates.
      --printUnusedTemplates       print warnings on unused templates.
      --quiet                      build in quiet mode
      --quiet                      build in quiet mode
      --renderSegments strings     named segments to render (configured in the segments config)
      --renderSegments strings     named segments to render (configured in the segments config)
  -M, --renderToMemory             render to memory (mostly useful when running the server)
  -M, --renderToMemory             render to memory (mostly useful when running the server)
  -s, --source string              filesystem path to read files relative from
      --templateMetrics            display metrics about template executions
  -s, --source string              filesystem path to read files relative from
      --templateMetrics            display metrics about template executions
      --templateMetricsHints       calculate some improvement hints when combined with --templateMetrics
      --templateMetrics            display metrics about template executions
      --templateMetricsHints       calculate some improvement hints when combined with --templateMetrics
      --templateMetricsHints       calculate some improvement hints when combined with --templateMetrics
  -t, --theme strings              themes to use (located in /themes/THEMENAME/)
      --themesDir string           filesystem path to themes directory
      --trace file                 write trace to file (not useful in general)
  -w, --watch                      watch filesystem for changes and recreate as needed

Use &#34;hugo [command] --help&#34; for more information about a command.
```

最常用的几个命令如下：
```bash
hugo new site my-site # 创建一个新站点
hugo new content posts/my-first-post.md # 创建一个新文章
hugo server -D # 启动本地服务器，-D表示草稿文件也会进行编译
hugo server --environment production # 启动本地服务器时指定生产环境，看到的页面是生产环境的
hugo --gc # 清理无用文件
hugo --minify --gc --cleanDestinationDir # 压缩文件并清理无用文件
```

---

> 作者: [beneliu](https://github.com/beneliu)  
> URL: https://beneliu.github.io/posts/hugo-fixit/738f0f6/  

