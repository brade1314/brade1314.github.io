---
layout:       post
title:        "博客搭建"
subtitle:     "Jekyll + Github Pages"
date:         2019-03-07 14:30:00
author:       "Brade"
header-style: text
header-mask:  0.3
catalog:      true
tags:
    - Jekyll
    - 博客
---

# 一 、[Jekyll](https://www.jekyll.com.cn/) 是什么？
> 定义来自官网：Jekyll 是一个简单的博客形态的静态站点生产机器。它有一个模版目录，其中包含原始文本格式的文档，
通过[Markdown](https://daringfireball.net/projects/markdown/)（或者[Textile](http://textile.sitemonks.com/)） 
以及[Liquid](http://docs.shopify.com/themes/liquid-basics) 转化成一个完整的可发布的静态网站，你可以发布在任何你喜爱的服务器上。
Jekyll也可以运行在[GitHub Page](http://pages.github.com/) 上，也就是说，你可以使用 GitHub 的服务来搭建你的项目页面、博客或者网站，而且是完全免费的。
>+ 简单：无需数据库、评论功能，不需要不断的更新版本--只用关心你的博客内容。
>+ 静态：只用 Markdown (或 Textile)、Liquid、HTML & CSS 就可以构建可部署的静态网站。
>+ 博客形态：自定义地址、分类、页面、博客内容 以及 自定义的布局设计 都是系统中的一等公民。

# 二、[安装Jekyll](https://www.jekyll.com.cn/docs/installation/)
 具体安装流程官网都有，就简单写下：
    
1. 系统(`linux/Linux/Unix/Mac OS X`) ，你可以使用 `Jekyll running on Windows`，但是官方文档并不建议你在 `Windows` 平台上安装 `Jekyll`。

2. 安装好 Ruby，RubyGems  
  $ `gem install jekyll`
  如果需要安装指定版本，在上述命令后加上一个`-v`参数  
  $ `gem install jekyll -v '指定版本号'`
     
3. 进入到博客目录，可以从github进行clone，也可以本地新建，然后(不指定ip默认只能本地访问)启动
  $ `jekyll serve -w --host=0.0.0.0`  
  其它命令使用
  $ `jekyll serve -h`  

       Usage:
         jekyll serve [options]
         Options:
                   --config CONFIG_FILE[,CONFIG_FILE2,...]  Custom configuration file
               -d, --destination DESTINATION  The current folder will be generated into DESTINATION
               -s, --source SOURCE  Custom source directory
                   --future       Publishes posts with a future date
                   --limit_posts MAX_POSTS  Limits the number of posts to parse and publish
               -w, --[no-]watch   Watch for changes and rebuild
               -b, --baseurl URL  Serve the website from the given base URL
                   --force_polling  Force watch to use polling
                   --lsi          Use LSI for improved related posts
               -D, --drafts       Render posts in the _drafts folder
                   --unpublished  Render posts that were marked as unpublished
               -q, --quiet        Silence output.
               -V, --verbose      Print verbose output.
               -I, --incremental  Enable incremental rebuild.
                   --strict_front_matter  Fail if errors are present in front matter
                   --ssl-cert [CERT]  X.509 (SSL) certificate.
               -H, --host [HOST]  Host to bind to
               -o, --open-url     Launch your site in a browser
               -B, --detach       Run the server in the background
                   --ssl-key [KEY]  X.509 (SSL) Private Key.
               -P, --port [PORT]  Port to listen on
                   --show-dir-listing  Show a directory listing instead of loading your index file.
                   --skip-initial-build  Skips the initial site build which occurs before the server is started.
               -l, --livereload   Use LiveReload to automatically refresh browsers
                   --livereload-ignore ignore GLOB1[,GLOB2[,...]]  Files for LiveReload to ignore. Remember to quote the values so your shell won't expand them
                   --livereload-min-delay [SECONDS]  Minimum reload delay
                   --livereload-max-delay [SECONDS]  Maximum reload delay
                   --livereload-port [PORT]  Port for LiveReload to listen on
               -h, --help         Show this message
               -v, --version      Print the name and version
               -t, --trace        Show the full backtrace when an error occurs
               -s, --source [DIR]  Source directory (defaults to ./)
               -d, --destination [DIR]  Destination directory (defaults to ./_site)
                   --safe         Safe mode (defaults to false)
               -p, --plugins PLUGINS_DIR1[,PLUGINS_DIR2[,...]]  Plugins directory (defaults to ./_plugins)
                   --layouts DIR  Layouts directory (defaults to ./_layouts)
                   --profile      Generate a Liquid rendering profile
               -h, --help         Show this message
               -v, --version      Print the name and version
               -t, --trace        Show the full backtrace when an error occurs 
4. 目录结构，可以查看[官网](https://www.jekyll.com.cn/docs/structure/)。
一个基本的 Jekyll 网站的目录结构一般是像这样的：
		
	|文件/目录|描述|
	| ---- | ---- |
	|`_config.yml`| 保存配置数据。很多配置选项都会直接从命令行中进行设置，但是如果你把那些配置写在这儿，你就不用非要去记住那些命令了。|
	|`_drafts`| `drafts`是未发布的文章。这些文件的格式中都没有 `title.MARKUP` 数据。
	|`_includes`| 你可以加载这些包含部分到你的布局或者文章中以方便重用。可以用[Liquid](http://docs.shopify.com/themes/liquid-basics) 来把文件 `_includes/file.ext` 包含进来。  |
	|`_layouts`| layouts 是包裹在文章外部的模板。布局可以在 YAML 头信息中根据不同文章进行选择。 这将在下一个部分进行介绍。标签  `{{ content }}` 可以将content插入页面中。|
	|`_posts`| 这里放的就是你的文章了。文件格式很重要，必须要符合: `YEAR-MONTH-DAY-title.MARKUP`。 `The permalinks` 可以在文章中自己定制，但是数据和标记语言都是根据文件名来确定的。|
	|`_data`| 格式良好的网站数据应该放在这里。 jekyll引擎将自动加载此目录中的所有yaml文件（以.yml或.yaml结尾）。 如果目录下有`member.yml`文件，则可以通过`site.data.members`访问该文件的内容。|
	|`_site`| 一旦 `Jekyll` 完成转换，就会将生成的页面放在这里（默认）。最好将这个目录放进你的 `.gitignore` 文件中。|
	|`index.html` `and other HTML`, `Markdown`, `Textile files`| 如果这些文件中包含 YAML 头信息 部分，Jekyll 就会自动将它们进行转换。当然，其他的如 `.html`， `.markdown`，  `.md`，或者 `.textile` 等在你的站点根目录下或者不是以上提到的目录中的文件也会被转换。|
	|`Other Files/Folders`| 其他一些未被提及的目录和文件如 `css` 还有 `images` 文件夹， `favicon.ico` 等文件都将被完全拷贝到生成的 `_site` 中。


# 三、写博客
1. 先将github上clone到本地，使用`jekyll serve -w -- host=0.0.0.0`启动，实时查看文章效果。
2. 在_posts目录下按文件名定义格式要求新建文件，比如`2019-03-06-myblog.md`。
3. 所有博客文章顶部必须有一段[YAML头信息](https://www.jekyll.com.cn/docs/frontmatter/)(`YAML front- matter`)。 
在它下面，就可以选择你喜欢的格式来写文章。Jekyll支持2种流行的标记语言格式：[Markdown](http://www.markdown.cn/)和[Textile](http://textile.sitemonks.com/)。 
这些格式都有自己的方式来标记文章中不同类型的内容，所以你首先需要熟悉这些格式并选择一种最符合你需求的。
具体请查看[官网说明](https://www.jekyll.com.cn/docs/posts/)。

# 四. 使用Git page 部署
> GitHub Pages 是一个静态网站托管服务，直接从github仓库托管你个人、公司或者项目页面 ，并且不需要你写任何后端语言来支持。
具体查看[git page帮助文档](https://help.github.com/en#github-pages-basics)。

1. 建立仓库名字改为： `<username>.github.io`，`<username>`就是你的github用户名。
2. 将你写好的博客文件，提交到github，可以使用命令`git add --all`，`git commit -m '提交博客***'`，`git push`。
3. 在浏览器打开`https://xxx.github.io`，你就可以看到博客页面。
