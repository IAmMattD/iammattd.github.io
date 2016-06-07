---
title: 深度论坛最近逗比很多 
author: MattD
layout: post
permalink: /2016/05/27/there-are-many-noobs-in-deepin-forum.html
categories: 乱七八糟
---
随着 Deepin 这个品牌知名度的提高，Deepin 论坛也不可避免地沦为了是非之地。

各种日经傻逼建议已经习以为常，最近居然连哲学家都冒出来了。

感觉有点对不住 `diyiliaoya` 兄，他算是间接躺枪，帮我背了个锅。

既然不能在深度论坛明目张胆开喷，那我就在个人博客这一方小空间里面发发牢骚吧。

我昨天在论坛的发帖多多少少顾及了论坛规则，所以没有指名道姓，在这里当然就无所谓了。

<!-- more -->

我看着不爽的逗比有三个：`zzz19760225`、`suoniao` 和 `youyou2011`。

`zzz19760225`，哲学家、宗教狂热分子。看这 ID 是个 40 多岁的家伙了，然而其言行、其人品、其智商堪比小学生。之前是在他人的回帖里面大量点评，言而无物，文不对题。后来引起多人的反感，被警告一次。最近又开始死灰复燃，而且摇身一变成了哲学家。歪楼、针对其他人的同一回帖进行多次回复，这都是家常便饭了。果然年纪大了，有些劣根性已经死性不改。估计生活里是个 loser，就上论坛来找存在感来了。对其他新人毫无帮助，反而把论坛搞得乌烟瘴气，口水不断。果断用脚本屏蔽，眼不见为净。建议 `ArthurDeepin` 也别和这家伙纠缠了，一是有代沟，二是永远不要试图说服一个傻逼。

`suoniao`，先知、营销家。昨天突然崛起的一枚逗比新星。网易云音乐 Linux 版的发布引起了这家伙的高潮，在水区连发多帖，恨不得立马给 Deepin 规划好未来五年十年的道路。呵呵，这种逗比其实一直存在，时不时就会有一些新注册的号来为 Deepin 指明未来发展方向，个个都以为自己比 Deepin 公司的销售和策划聪明。然而在我看来，不过是一群跳梁小丑。果断也用脚本屏蔽。

`youyou2011`，策划师。在论坛多次提出各种不切实际的建议，总是一副“Deepin 应该这样这样”，“Deepin 有没有考虑这样那样”的嘴脸。然而提出来的“建议”完全不过大脑，估计深度自己都想过不知道多少回了。说得好听叫自作聪明，说得难听呢就是不自量力，没有自知之明。当然，这位逗比比起以上两位来还是稍微那么守序一点，所以我并没有屏蔽。

好了，喷完，最后附上我用来屏蔽一些逗比的油猴脚本：

{% highlight bash %}
// ==UserScript==
// @name           Discuz屏蔽ID
// @namespace      
// @include        */viewthread.php*
// @include        */thread*
// @include        */redirect.php*
// @include        */forum*
// @include        *bbs.deepin.org*
// ==/UserScript==
var bl = new Array("zzz19760225","id2","id3");
for (x in bl) {
        b = document.evaluate('//table/tbody[tr[1]/td[1]//a[text()="' + bl[x] + '"]]', document, null, XPathResult.UNORDERED_NODE_SNAPSHOT_TYPE, null);
        if (b.snapshotLength) {
                for (var i = 0,c=""; i < b.snapshotLength; i++) {
                c = b.snapshotItem(i).firstChild.childNodes[3].textContent.replace(/\s*/g,"").slice(0,2);
                c = (Number(c) > 9)?c+"楼":c
                        b.snapshotItem(i).innerHTML = "<center>被屏蔽帖子 " +c+" <font color=red>" + bl[x] + "</font></center>";
                }
        }
}

for (x in bl) {
        b = document.evaluate('//table/tbody[tr[1]/td[1]/div[1]//font[text()="' + bl[x] + '"]]', document, null, XPathResult.UNORDERED_NODE_SNAPSHOT_TYPE, null);
        if (b.snapshotLength) {
                for (var i = 0,c=""; i < b.snapshotLength; i++) {
                c =String(b.snapshotItem(i).firstChild.childNodes[3].textContent.match(/\d+#/)).replace(/#/,"楼");
                b.snapshotItem(i).innerHTML = "<center>被屏蔽帖子 " +c+" <font color=red>" + bl[x] + "</font></center>";
                }
        }
}

for (x in bl) {
        b = document.evaluate('//table/tbody[tr[1]/td[2]//cite/a[text()="' + bl[x] + '"]]', document, null, XPathResult.UNORDERED_NODE_SNAPSHOT_TYPE, null);
        if (b.snapshotLength) {
                for (var i = 0,c=""; i < b.snapshotLength; i++) {
                b.snapshotItem(i).innerHTML = "";
                }
        }
}
{% endhighlight %}
