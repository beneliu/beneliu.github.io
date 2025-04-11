# Fixit-主题美化


{{&lt; admonition type=info title=&#34;内容简介&#34; open=true &gt;}}
记录FixIt主题美化的操作步骤。
{{&lt; /admonition &gt;}}

## 自定义css

### 自定义字体

在站点项目根目录下面，在`/assets/css/_custom.scss`中添加以下代码即可自定义字体。推荐在[中文网字计划](https://chinese-font.netlify.app/zh-cn/cdn/)中挑选自己喜欢的字体。
```scss
// 引入字体 https://chinese-font.netlify.app/zh-cn/cdn/
@import url(&#39;https://chinese-fonts-cdn.deno.dev/packages/lxgwwenkai/dist/LXGWWenKai-Light/result.css&#39;);
// 自定义字体
html,body {
   font-family: &#34;LXGW WenKai Light&#34;;
   font-weight: normal;
   font-size: 1rem;
}
```

### 自定义文章网格背景

在`/assets/css/_custom.scss`文件中添加如下代码：
```scss
/** 添加网格背景 */
.single {
  .single-subtitle {
    color: #57606a;
  }

  .content {
    background-image: linear-gradient(90deg, rgba(60, 10, 30, .04) 3%, transparent 0), linear-gradient(1turn, rgba(60, 10, 30, .04) 3%, transparent 0);
    background-size: 20px 20px;
    background-position: center;
  
    [data-theme=&#39;dark&#39;] &amp; {
      background-image: linear-gradient(90deg, rgba(195, 245, 215, .04) 3%, transparent 0), linear-gradient(1turn, rgba(195, 245, 215, .04) 3%, transparent 0);
    }
  }
}
```

## 自定义模板

### 自定义文章页面侧边目录模板

在站点项目根目录下面，`/layouts/partials/`文件夹下创建`custom`文件夹，可以用来存储自定义模板。Hugo有自带的toc模板, 可以通过`{{.Content}}`引入这个模板, 具体可以看[官方使用教程](https://hugo.opendocs.io/zh-cn/methods/page/tableofcontents/)。在使用过程发现有多级标题时，侧边目录标号顺序不能实现`1.1`、`1.1.1`的形式。索性想自己定义一个toc模板使用，需要`1.1`、`1.1.1`的形式文章页侧边目录编号时，自己在md文档中添加即可。
```html
{{- /* 自定义toc目录 */ -}}
{{ $headers := findRE &#34;&lt;h[1-4].*?&gt;(.|\n])&#43;?&lt;/h[1-4]&gt;&#34; .Content }}
&lt;!-- at least one header to link to --&gt;
{{ if ge (len $headers) 1 }}
{{ $h1_n := len (findRE &#34;(.|\n])&#43;?&#34; .Content) }}
{{ $re := (cond (eq $h1_n 0) &#34;&lt;h[2-4]&#34; &#34;&lt;h[1-4]&#34;) }}
{{ $renum := (cond (eq $h1_n 0) &#34;[2-4]&#34; &#34;[1-4]&#34;) }}

&lt;!--Scrollspy--&gt;
&lt;div class=&#34;toc&#34;&gt;

    &lt;div class=&#34;page-header&#34;&gt;&lt;strong&gt;- CATALOG -&lt;/strong&gt;&lt;/div&gt;

    &lt;div id=&#34;page-scrollspy&#34; class=&#34;toc-nav&#34;&gt;

        {{ range $headers }}
        {{ $header := . }}
            {{ range first 1 (findRE $re $header 1) }}
                {{ range findRE $renum . 1 }}
                {{ $next_heading := (cond (eq $h1_n 0) (sub (int .) 1 ) (int . ) ) }}
                    {{ range seq $next_heading }}
                    &lt;ul class=&#34;nav&#34;&gt;
                    {{end}}
                    {{ $anchorId := (replaceRE &#34;.* id=\&#34;(.*?)\&#34;.*&#34; &#34;$1&#34; $header ) }}
                        &lt;li class=&#34;nav-item&#34;&gt;
                            &lt;a class=&#34;nav-link text-left&#34; href=&#34;#{{ $anchorId }}&#34;&gt;
                            {{ $header | plainify | htmlUnescape }}
                            &lt;/a&gt;
                        &lt;/li&gt;
                    &lt;!-- close list --&gt;
                    {{ range seq $next_heading }}
                    &lt;/ul&gt;
                    {{ end }}
                {{ end }}
            {{ end }}
        {{ end }}

    &lt;/div&gt;

&lt;/div&gt;
&lt;!--Scrollspy--&gt;

{{ end }}
```

## 自定义js

在站点项目根目录下面，在`/assets/js/custom.js`中添加自定义的js代码。

### 设置网站title动态

设置网站title动态，当网页失去焦点时改变网页title，引起用户注意。
```js
// 动态设置网站title，当网页失去焦点时改变网页title，引起用户注意。
document.addEventListener(&#39;DOMContentLoaded&#39;, function () {
    // 调试日志
    // console.log(&#39;[动态标题] 脚本已加载&#39;);

    const originTitle = document.title;
    let titleTimer;

    function updateTitle(newTitle, duration = 2000) {
        document.title = newTitle;
        if (duration &gt; 0) {
            clearTimeout(titleTimer);
            titleTimer = setTimeout(() =&gt; {
                document.title = originTitle;
            }, duration);
        }
    }

    // 页面可见性变化
    document.addEventListener(&#39;visibilitychange&#39;, function () {
        // console.log(&#39;[动态标题] 可见性变化:&#39;, document.hidden);
        if (document.hidden) {
            updateTitle(&#39;👀 别走呀～ &#39;, 0);
        } else {
            updateTitle(&#39;🎉 欢迎回来！ &#39;);
        }
    });

    // 窗口焦点变化
    window.addEventListener(&#39;blur&#39;, function () {
        // console.log(&#39;[动态标题] 窗口失去焦点&#39;);
        updateTitle(&#39;💤 我在等你哦～ &#39;, 0);
    });

    window.addEventListener(&#39;focus&#39;, function () {
        // console.log(&#39;[动态标题] 窗口获得焦点&#39;);
        updateTitle(&#39;😍 你回来啦！ &#39;);
    });

    // console.log(&#39;[动态标题] 初始化完成&#39;);
});
```

## 添加友链

先创建友情链接页面：
```bash
hugo new content friends/index.md
```
在 Front matter 中设置`layout: friends`，并在`yourSite/data/`目录下创建`friends.yml`，其内容格式如下：
```yaml
# 朋友/站点信息例子
- nickname: 朋友名字
  avatar: 朋友头像
  url: 站点链接
  description: 对朋友或其站点的说明
```
然后就可以将友链页面添加到主页菜单栏了。

**参考资料：**
1. [内容管理概述 | FixiIt](https://fixit.lruihao.cn/zh-cn/documentation/content-management/introduction/)
2. [Fixit-主题美化记录](https://geekswg.js.cool/posts/2023/fixit-beautification/)
3. [Hugo添加文章目录toc](https://www.ariesme.com/posts/2019/add_toc_for_hugo/)


---

> 作者: [beneliu](https://github.com/beneliu)  
> URL: https://beneliu.github.io/posts/hugo-fixit/a9f08a3/  

