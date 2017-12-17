+++
title = "Hugo Katex Support"
date = 2017-12-11T21:57:08+08:00
tags = ["hugo", "tex"]
categories = ['web']
keywords = ['hugo','mmark','katex']
markup = 'mmark'
+++
最近需要写一篇关于tcp拥塞控制的博客，里面需要用到好些公式。结果遇到了hugo默认的markdown
解析器`blackberry`关于解析latex格式相关的问题。最终通过搜索采用__katex+改用mmark解析器__
的方式解决了.
<!--more-->
## mmark解析器
由于普通markdown语法和latex语法中包含类似转义符`\`以及`_`强调-下标符号，还有`&`符号的转换，
导致在普通的markdown语法中写latex公式并转换到html中间有很多麻烦。

mmark是一个markdown语法的超集，其中就包含特定的对于latex公式方面的兼容，并且兼顾了markdown
的特点。

1. 所有包含在`$$`和`$$`之间的内容不做任何转义处理，包括`&`和`\`、`_`.
2. `$$ $$`根据环境，如果在段落当中，则转换为`\(\)`根据katex或者mathjax的默认规定，表示行内
公式。如果独立成段落（前后都有空行），则转换为`\[\]`。
3. 将上述内容包裹在`<span class="math"></span>`当中，方便公式渲染器渲染。

## Katex支持
通过在模版的head.html或者其他需要显示公式的页面均能够载入的模版文件当中，添加katex的三个文件
`katex.min.js`、`katex.min.css`、`auto-render.min.js`。具体的链接到[github/katex](2)说明页面
中找。

其中的`auto-render.min.js`提供了对于页面内容的渲染支持函数。只需要载入后使用

    renderMathInElement(document.body)
就可以自动将页面中`class="math"`的内容渲染成公式。使用如下方式让页面加载完成后自动执行该函数

    document.addEventListener("DOMContentLoaded", function() {
      renderMathInElement(document.body,
      [
          {left: "$$", right: "$$", display: true},
          {left: "\\[", right: "\\]", display: true},
          {left: "\\(", right: "\\)", display: false},
      ]);
    });


[2]:https://github.com/Khan/KaTeX
