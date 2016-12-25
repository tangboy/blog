---
title: mathjax 在hexo应用
date: 2016-12-25 16:20:26
tags:
    - Mathjax
---

# Hexo支持公式

## 常规方法
在 Hexo 中使用 Mathjax 最常规的方法就是主题的 after_footer.ejs 里加入 (该 Mathjax 代码根据 Math StackExchange 所用修改)

{% codeblock lang:javascript %}
<script type="text/x-mathjax-config">
    MathJax.Hub.Config({"HTML-CSS": { preferredFont: "TeX", availableFonts: ["STIX","TeX"], linebreaks: { automatic:true }, EqnChunk: (MathJax.Hub.Browser.isMobile ? 10 : 50) },
        tex2jax: { inlineMath: [ ["$", "$"], ["\\(","\\)"] ], processEscapes: true, ignoreClass: "tex2jax_ignore|dno",skipTags: ['script', 'noscript', 'style', 'textarea', 'pre', 'code']},
        TeX: {  noUndefined: { attributes: { mathcolor: "red", mathbackground: "#FFEEEE", mathsize: "90%" } }, Macros: { href: "{}" } },
        messageStyle: "none"
    }); 
</script>
<script type="text/x-mathjax-config">
    MathJax.Hub.Queue(function() {
        var all = MathJax.Hub.getAllJax(), i;
        for(i=0; i < all.length; i += 1) {
            all[i].SourceElement().parentNode.className += ' has-jax';
        }
    });
</script>
<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>
{% endcodeblock %}

接下来的使用方法就和平常使用 Mathjax 一样了.

不过这个方法有个令人不爽的缺点, 众所周知加载 Mathjax 的数学公式时是很消耗资源和时间的. 即使在网页中并没有生成公式时, 也会加载最基本 MathJax.js, 而这个文件也有 58KB. 为解决这个问题可参考如下进阶版.

## 进阶版
我们可以考虑只有在用到公式的页面才加载 Mathjax, 因此需要加载一些控制.

首先将上面的代码写成一个 mathjax.ejs 文件放在 partial 目录下, 并且在 Hexo 的根目录的 \_config.yml 里面加入 mathjax: true, 接下来在 after_footer.ejs 里加入

{% codeblock lang:javascript %}
<% if (page.mathjax){ %>
<%- partial('mathjax') %>
<% } %>
{% endcodeblock %}


在文章需要调用 Mathjax 时, 只需在 front-matter 前加上 mathjax: true 即可, 即

{% codeblock %}
title: 测试Mathjax
date: 2014-2-14 23:25:23
tags: Mathmatics
categories: Mathjax
mathjax: true
---
{% endcodeblock %}

### 缺点
*Markdown* 里使用 *Mathjax* 有一个很大的缺点, 比如下面这个问题.
```
博主你好，你这样用mathjax，在用markdown写博客时能正常书写带下标的公式吗？
比如下面一段话：
This is an example for $x_mu$ and $y_mu$.
两个`—`会被看成markdown中的斜体。
```

这个的解决办法可以使用转义符, 即如下输出即可

```
This is an example for $x\_mu$ and $y\_mu$.
```

## 解决方法
除了上述方法以外, 还有下面几种.
### 插件
安装插件 *Hexo-math*, 安装方法如下, 依次为
{% codeblock lang:shell %}
npm install hexo-math --save
{% endcodeblock %}

在*Hexo*文件夹中执行
{% codeblock lang:shell %}
hexo math install
{% endcodeblock%}

在*_config.yml*文件中添加：
{% codeblock %}
plugins:
   hexo-math
{% endcodeblock %}
对于不含特殊符号的公式，可以直接使用 MathJax 的 inline math 表达式. 如果含有特殊符号，则需要人肉 escape，如 \ 之类的特殊符号在 LaTex 表达式中出现频率很高，这样就很麻烦，使用 tag 能够省不少事。

## 示例
### 行内公式
1. 对于不含特殊符号的公式，可以直接使用MathJax的inline math表达式，比如：
```
一条直线$y=ax+b$使这几个点
```

效果为
$y=ax+b$

2. 对于含有特殊符号的公式，比如\和_符号在Markdown转义过程会出现一些问题，通常可以在符号前面加\,变为 \\ 和\_进行数学公式编辑。比如：
```
一系列点$(x\_i,y\_i)(i=1,2,...n,n\ge 3)$
```

效果为
一系列点$(x_i,y_i)(i = 1,2,\cdots, n, n\ge 3)$

### 行间公式
对于行间公式有帖子说可以采用Tag Block方式解决Markdown转义问题，不知为何总感觉使用Tag Block有点不方便。我亲自测试了一下行内公式方法依旧可用。比如：
$$
\begin{aligned}
w&=ax^2 + bxy + cy^2\\\
&=a(x + \frac{by}{2a})^2 + (c - \dfrac{(by)^2}{4a})\\\
&=\frac{1}{4a}[4a^2(x+\frac{by}{2a})^2 + (4ac-b^2)y^2]
\end{aligned}
$$
## 参考
1. [hexo添加hexo-math插件以支持LaTeX公式](http://tidus.xyz/2016/03/06/hexo%E6%B7%BB%E5%8A%A0hexo-math%E6%8F%92%E4%BB%B6%E4%BB%A5%E6%94%AF%E6%8C%81LaTeX%E5%85%AC%E5%BC%8F/)
2. [hexo MathJax插件](http://catx.me/2014/03/09/hexo-mathjax-plugin/)
3. [MathJax插件在Hexo中的应用](http://www.catxue.com/2015/03/20/MathJax/)
4. [在Hexo中完美使用Mathjax输出数学公式](http://lukang.me/2014/mathjax-for-hexo.html)
