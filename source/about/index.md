---
title: 关于
layout: page
comments: false
---
### 对不起，无法找到该页。

页面将在<span id="jumpTo"></span>秒内自动跳转回[首页](https://xmaihh.github.io/)

<script type="text/javascript">
function countDown(secs, surl) {
    var jumpTo = document.getElementById('jumpTo');
    jumpTo.innerHTML = secs;
    if (--secs > 0) {
        setTimeout("countDown(" + secs + ",'" + surl + "')", 1000);
    } else {
        location.href = surl;
    }
}
</script>
<script type="text/javascript">countDown(7,'/');</script>