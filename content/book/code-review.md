Title: Code 书评
Date: 2014-03-21 16:33
Category: Book
Tags: Code, review
Slug: code-review
Author: Qian Gu
Summary: 最近花了一个星期的时间，终于把 Code 看完了，感慨颇深。

最近花了一个星期的时间，终于把 [*Code*][Code] 看完了，看完全书，观后感总结出来就一个字——爽～

书名： [*Code:  The Hidden Language of Computer Hardware and Software*][code]

作者： [Charles Petzold][CP]

这本书的作者 [Charles Petzold][CPwiki] 是 Windows 编程界一位大师，世界顶级技术作家。1994 年 5 月，Petzold 作为仅有的七个人之一（并且是唯一的作家）被《Window Magazine》和 Microsoft 公司授予 “Windows Pioneer” 奖，以表彰他对 Microsoft Windows 的成功做出的贡献 。

关于这本书的 “江湖地位” ，只需要来看看 stackoverflow 上的这个帖子就明白了：

[What is the single most influential book every programmer should read?][wsm]

本书排名在 *第15*，当然绝对排名没有很大的意义，但是也能说明本书的影响力 。另外本书号称 “完全不懂计算机的人也能看懂”... 如果这都不读，我不知道还有什么样的书值得我们去花费时间 : D

[Code]: http://book.douban.com/subject/1494026/
[CP]: http://charlespetzold.com/
[CPwiki]: http://en.wikipedia.org/wiki/Charles-Petzold
[wsm]: http://stackoverflow.com/questions/1711/what-is-the-single-most-influential-book-every-programmer-should-read

## 打通任督二脉

纵观程序员的 “武功秘籍”，不外乎两种：一种是讲 `道`，另外一种是讲 `术` 。

`道` 即 “道理”（真理），是脱离了具体事物，抽象出来的事物的本质 。比如平时学习的 *数学*、*物理*、*化学* 知识，再比程序中的 *算法*、分析解决问题的 *思维方式* 等，都不会因时间和环境变化而不适用。

`术` 即 “技术”（技巧），是人们在具体工程实践中总结出来的一些技巧。比如 “The art of XXX” ，“The principle of XXX” 等 。

是的，这本书既不是单独讲 `道`，也不是单独讲 `术`，而是将两者结合起来，让你不仅明白 *道*  的深奥，也明白 *术* 的精妙；让你体会到 *道* 是如何在 *术* 中实现，也体会到 *术* 是如何在 *道*   的约束下追求极限的 。更加关键的是全书都是用很平白的语言讲述，平白到 “即使完全不懂计算机的人也能读懂”的地步，这就是其厉害之处了。

就好像你学习了五花八门的心法秘籍和各式武功，每一门都厉害无比，可偏偏无法把这些武功融会贯通，修至大成 。这时候，你需要的不是一本《葵花宝典》，而是《论天下武功之间的关系及其融会贯通方法》，打通自己的任督二脉，也就是这本 [Code][Code] 。

读这本书时，让我想起了在大学 4 年间读的各种课程：

读到 第 6 章 “电报机与继电器” 让我想起 *大学物理* ，虽然翘了很多课，但是电磁感应什么的有中学基础，还好没有忘记 。

读到 第 7～9 章，书中对计数的方法和进制的讨论，让我想起 *大学计算机基础* ，然后花了一天时间去把以前一知半解的 “原码、反码、补码” 仔细推敲总结一遍，收获颇大 。

读到 第 10 章，布尔代数 让我想起 *离散数学* ，尤其发人深醒的是布尔发明了 布尔逻辑 100 多年后， 才有人（香农）将其和电路联系起来 。

读到 第 11～13 章，跟随作者的思路，我才明白当年令人痛不欲生的 *模拟电路* 、*数字电路*  到底是来干什么的，数字电路是如何从最基本的门搭建出来更强的功能的元器件的，联系前面学习的 编码的知识，明白这样做的道理 。

读到 第 16～22 章，让我想起 *微机原理* ，对微处理器的工作方式不再模糊不清，跟随着作者的思路搭建一台 “计算机”（当然和现在的计算机完全不同，但是原理是一模一样的），为自己搭建的处理器设计一套可以实现自动功能的编程语言，然后写一段程序，甚至一窥操作系统 。

还有等等...

胡扯了这么多，书固然是好书，但是更让人深思的是国内外教育的差异。吐槽我们的教育体制毫无意义，作为普通人，我们能做到的就是自己努力，找真正的好书、向真正的大师学习，比如上面提到过的

[What is the single most influential book every programmer should read?][wsm]

除了技术、还有很多程序员需要看的书。找到一本好书，然后 Let's start it!
