# Hugo 自定义页面


{{&lt; admonition type=info title=&#34;内容简介&#34; open=true &gt;}}
基于主题在 **Hugo** 博客中添加一个新的页面。这里说的页面不是指 **Hugo** 利用 **markdown** 文件生成的页面，而是我们自己手动创建的 **html** 页面。
{{&lt; /admonition &gt;}}

我想添加一个网站合集页面，这个页面分类展示各类有用的网站。但是 FixIt 主题中并没有提供这个页面实现，所以只能自己动手写一个。

## 前置知识

添加自定义页面其实并不复杂，但是需要了解一些 HTML、CSS、JavaScript 的基础，还有[ Hugo 中函数的用法](https://hugo.opendocs.io/functions/)。其次不太推荐使用类似 Vue 这样的 JS 框架，因为 Hugo 的页面使用了大量的 golang 模板语法，就是你在主题文件中看到的`{{}}`语法，而 Vue 中也有到这个语法，这可能会在无意中给你的开发造成障碍，甚至在解析模板时出错。

## 添加自定义页面

作为一个前端小白，自己从头实现一个新的页面，有点难度。这里我参考了 FixIt 主题友链页面的实现方式，友链页面添加可以参考 [**Fixit-主题美化**](https://benelin.site/posts/hugo-fixit/a9f08a3/) 一文。

### 定义站点页面的内容模板

在`archetypes`目录下创建`websites.md`，内容参考`archetypes/friends.md`。
```md
---
title: {{ replace .TranslationBaseName &#34;-&#34; &#34; &#34; | title }}
subtitle:
layout: websites
date: {{ .Date }}
description: &#34;{{ .Site.Params.author.name }}&#39;s websites&#34;
keywords:
  - &#39;Hugo FixIt&#39;
  - &#39;websites template&#39;
  - 站点导航
comment: false
---

&lt;!-- The `websites.yml` file placed in the `yourProject/data/` directory will be loaded automatically here. --&gt;

---

&lt;!-- You can define additional content below for this page. --&gt;

## Base info

- nickname: Lruihao
- avatar: https://lruihao.cn/images/avatar.jpg
- url: https://lruihao.cn
- description: Lruihao&#39;s Note
```

### 定义站点页面的模板

在`layouts/page/`目录下创建`websites.html`，内容参考`layouts/page/friends.html`。这里在`friends.html`页面模板上做了一些针对性的修改。
```html
{{- define &#34;title&#34; -}}
{{- cond (.Param &#34;capitalizeTitles&#34;) (title .Title) .Title -}}
{{- if .Site.Params.withSiteTitle }} {{ .Site.Params.titleDelimiter }} {{ .Site.Title }}{{- end -}}
{{- end -}}

{{- define &#34;content&#34; -}}
{{- $params := partial &#34;function/params.html&#34; -}}
&lt;article class=&#34;page single special websites&#34;&gt;
  &lt;div class=&#34;header&#34;&gt;
    {{- /* Title */ -}}
    &lt;h1 class=&#34;single-title animate__animated animate__pulse animate__faster&#34;&gt;{{- cond (.Param &#34;capitalizeTitles&#34;)
      (title .Title) .Title -}}&lt;/h1&gt;
    {{- /* Subtitle */ -}}
    {{- with $params.subtitle -}}&lt;p class=&#34;single-subtitle animate__animated animate__fadeIn&#34;&gt;{{ . | $.RenderString }}
    &lt;/p&gt;{{- end -}}
  &lt;/div&gt;

  {{- /* website links grouped by category */ -}}
  &lt;script src=&#34;//at.alicdn.com/t/font_578712_g26jo2kbzd5qm2t9.js&#34; async defer&gt;&lt;/script&gt;
  {{- $websites := .Site.Data.websites -}}
  {{- $categories := dict -}}
  {{- range $website := $websites -}}
  {{- $category := $website.category | default &#34;未分类&#34; -}}
  {{- $items := index $categories $category | default slice -}}
  {{- $items = $items | append $website -}}
  {{- $categories = merge $categories (dict $category $items) -}}
  {{- end -}}
  {{- /* 获取所有分类并按权重排序 */ -}}
  {{- $sortedCategories := slice -}}
  {{- range $category, $items := $categories -}}
    {{- $weight := 9999 -}}  {{- /* 默认权重 */ -}}
    {{- range $items -}}
      {{- if isset . &#34;category_weight&#34; -}}
        {{- $weight = .category_weight -}}
      {{- end -}}
    {{- end -}}
    {{- $sortedCategories = $sortedCategories | append (dict &#34;name&#34; $category &#34;items&#34; $items &#34;weight&#34; $weight) -}}
  {{- end -}}
  {{- $sortedCategories = sort $sortedCategories &#34;weight&#34; &#34;asc&#34; -}}
  {{- /* 渲染排序后的分类 */ -}}
  {{- range $category := $sortedCategories -}}
  &lt;h2 class=&#34;website-category&#34;&gt;{{ $category.name }}&lt;/h2&gt;
  &lt;div class=&#34;website-links&#34;&gt;
    {{ range $index, $website := $category.items }}
    &lt;a class=&#34;website&#34; title=&#34;{{ $website.description }}&#34; href=&#34;{{ $website.url | safeURL }}&#34; rel=&#34;external noopener&#34;
      target=&#34;_blank&#34;&gt;
      {{ if $website.avatar }}
      {{- dict &#34;Src&#34; $website.avatar &#34;Alt&#34; $website.nickname &#34;Title&#34; $website.description &#34;Class&#34; &#34;website-avatar&#34; |
      partial &#34;plugin/image.html&#34; -}}
      {{ else }}
      &lt;svg class=&#34;website-avatar&#34; aria-hidden=&#34;true&#34;&gt;
        &lt;use xlink:href=&#34;#icon-{{ add 1 $index }}&#34;&gt;&lt;/use&gt;
      &lt;/svg&gt;
      {{ end }}
      &lt;div class=&#34;website-info&#34;&gt;
        &lt;span class=&#34;website-nickname&#34; title=&#34;{{ $website.nickname }}&#34;&gt;{{ $website.nickname }}&lt;/span&gt;
        &lt;span class=&#34;website-description&#34; title=&#34;{{ $website.description }}&#34;&gt;{{ $website.description }}&lt;/span&gt;
      &lt;/div&gt;
    &lt;/a&gt;
    {{ end }}
  &lt;/div&gt;
  {{- end -}}

  {{- /* Content */ -}}
  &lt;div class=&#34;content&#34; id=&#34;content&#34;&gt;
    {{- dict &#34;Content&#34; .Content &#34;Ruby&#34; $params.ruby &#34;Fraction&#34; $params.fraction &#34;Fontawesome&#34; $params.fontawesome |
    partial &#34;function/content.html&#34; | safeHTML -}}
  &lt;/div&gt;

  {{- /* Comment */ -}}
  {{- partial &#34;single/comment.html&#34; . -}}
&lt;/article&gt;
{{- end -}}
```

在`data/`目录下面创建`websites.yml`，这是和模板对应的站点配置文件。

```yaml
# 站点信息
- nickname: Hugo
  avatar: https://gohugo.io/favicon.ico
  url: https://gohugo.io/
  description: |
    Hugo is one of the most popular open-source static site generators.
    With its amazing speed and flexibility,
    Hugo makes building websites fun again.
  category: 本站技术支持
  category_weight: 1

- nickname: FixIt
  avatar: https://fixit.lruihao.cn/favicon.ico
  url: https://fixit.lruihao.cn/zh-cn/
  description: 一个简洁、优雅且高效的 Hugo 主题
  category: 本站技术支持
  category_weight: 1

- nickname: Font Awesome
  avatar: https://fontawesome.com/favicon.ico
  url: https://fontawesome.com/icons
  description: 一个包含很多icon的开源图标库
  category: 工具类
  category_weight: 2
```

### 调整站点页面样式

在`assets/css/_custom.scss`文件中添加以下代码：
```scss
.website {
  display: flex;
  align-items: center;
}

.website-info {
  display: flex;
  flex-direction: column;
  margin-left: 10px;
}

.website-description {
  font-size: 0.8em;
  color: #666;
}

.website-links {
  display: flex;
  flex-wrap: wrap; // 允许换行
  gap: 20px; // 卡片间距
  margin-bottom: 30px; // 分类间间距
}

.website {
  display: flex;
  align-items: center;
  height: 80px;
  min-height: 80px;
  width: 250px; // 固定卡片宽度
  overflow: hidden;
  flex: 0 0 auto;
  max-width: none;
  box-sizing: border-box;
  padding: 12px;
  border-radius: 8px;
  background: rgba(0, 0, 0, 0.02);
  transition: all 0.2s ease;

  &amp;:hover {
    background: rgba(0, 0, 0, 0.05);
  }
}

.website-avatar {
  width: 60px;
  height: 60px;
  object-fit: cover;
  flex-shrink: 0;
}

.website-info {
  display: flex;
  flex-direction: column;
  justify-content: center; // 保持垂直居中
  height: 100%;
  margin-left: 12px;
  overflow: hidden;
  width: calc(100% - 72px);
}

.website-nickname {
  font-weight: bold;
  margin-bottom: 6px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.website-description {
  display: -webkit-box;
  -webkit-line-clamp: 1; // 限制显示1行
  -webkit-box-orient: vertical;
  overflow: hidden;
  text-overflow: ellipsis;
  font-size: 0.8em;
  color: #666;
  line-height: 1.4;
  max-height: 3.6em; // 3行高度
}
```

### 创建站点链接页面

创建一个站点链接页面，在 Front matter 中设置`layout: websites`，然后运行hugo命令生成页面：
```bash
hugo new content websites/index.md
hugo server -D
```





---

> 作者: [beneliu](https://github.com/beneliu)  
> URL: https://beneliu.github.io/posts/hugo-fixit/3c94d6e/  

