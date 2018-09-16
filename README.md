# 前言

Objective-C 屬於 C 語言的一個超集，而 C 是一個靜態語言。（[什麼是動態語言與靜態語言](https://www.jianshu.com/p/898b7de664da)）

但是 OC 藉由 runtime 賦予這個語言一些動態性，讓我們可以在運行時動態的決定執行內容。當撰寫大型專案、框架或者為第三方庫添加特性時，runtime 能幫助我們很簡單的完成這些事。

{% hint style="info" %}
對於支持了「部分」動態特性的 OC 來說到底應該定位成動態語言還是靜態語言。我認為沒有標準答案，這其實是一個「支援程度」上的問題，可以參考[這篇](https://www.zhihu.com/question/19970471)。
{% endhint %}

本文將介紹的主題如下：

* [Runtime 是什麼？](runtime-shi-shen.md)
* [Runtime 的常見結構與類別有哪些？](runtime-xiao-xi.md)
* [Runtime 如何進行消息傳遞與轉發？](runtime-de-xiao-xi.md)
* [Runtime 實際的應用場景有哪些？](runtime-de-yong-jing.md)



