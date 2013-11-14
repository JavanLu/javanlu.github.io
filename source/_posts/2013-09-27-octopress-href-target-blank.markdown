---
layout: post
title: "Octopress中设置markdown超链接target"
date: 2013-09-27 16:19
comments: true
categories: Tools
---

在使用Markdown语法生成超链接的时候，默认是在本窗口打开的。但是这会影响阅读体验，如果能将超链接在新窗口中打开就OK了。一种方案是使用HTML语法，但是这样破坏了写作时的体验，无法体现markdown的简洁性。好在Octopress可以很自由地定制新的功能。
<!-- more -->
Octopress的[Issues: Open links in a new window](https://github.com/imathis/octopress/issues/410)<sup>[1]</sup>中给出了几种可行的方案，我选择了[azone的方案](https://gist.github.com/azone/4523641)<sup>[2]</sup>。在``octopress\source\_includes\custom\head.html``中添加如下代码：

```javascript
<script type="text/javascript">
function addBlankTargetForLinks () {
  $('a[href^="http"]').each(function(){
      $(this).attr('target', '_blank');
  });
}

$(document).bind('DOMNodeInserted', function(event) {
  addBlankTargetForLinks();
});
</script>
```


>### 参考资料
>[1] Issues: Open links in a new window: [https://github.com/imathis/octopress/issues/410](https://github.com/imathis/octopress/issues/410)
>
>[2] azone的方案：[https://gist.github.com/azone/4523641](https://gist.github.com/azone/4523641) 
