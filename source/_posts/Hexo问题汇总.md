---
title: Hexo问题汇总
date: 2018-04-01 10:37:13
tags:
    - Hexo
    - Markdown
    - MathJax
categories: 工具使用
---

## MathJax换行符

网上很多在`Markdown`中插入`LaTeX`公式的教程中，换行用的`\\`双反斜杠，在使用默认的渲染引擎时，换行不能生效，使用`\newline`换行是非法的。笔者猜测是引擎的渲染语义问题，一次偶然的尝试中发现用三反斜杠`\\\`之后可以解决问题。

但是与下一条问题一样，更换默认的渲染引擎之后可以从根本上解决这个`符号语义`问题，从而可以使用`\\`正常换行。

## MathJax下划线冲突

在`Markdown`中渲染`MathJax`公式时，若下划线`_`随后跟着大括号`{}`，则这段`LaTex`公式会渲染失败。

解决方案，替换默认的`hexo-render-marked`引擎：
```
npm uninstall hexo-renderer-marked --save
npm install hexo-renderer-kramed --save
```

更多详细的解决方案细节，观看[源博客](https://www.cnblogs.com/Ai-heng/p/7282110.html)。