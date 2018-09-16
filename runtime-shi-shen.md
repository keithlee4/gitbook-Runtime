# Runtime 是什麼？

## 介紹

Objective-C 是一個基於 C 語言，加入物件導向特性和 [Smalltalk](https://zh.wikipedia.org/wiki/Smalltalk) 式的消息傳遞機制的擴展。該擴展的核心是一個用 C 和[編譯語言](https://medium.com/@totoroLiu/%E7%B7%A8%E8%AD%AF%E8%AA%9E%E8%A8%80-vs-%E7%9B%B4%E8%AD%AF%E8%AA%9E%E8%A8%80-5f34e6bae051)寫的 Runtime 程式庫。並且作為 Objective-C 物件導向和動態機制的基石。Runtime 基本由 C 與 組合語言 編寫。由於 OC 並不能直接轉譯成組合語言，因此由 runtime 來負責 OC 至 C 的過渡，C 可以直接編譯成組語，最後編寫成機器語言讓計算機識別。

理解 Runtime 能夠幫助開發者更好地了解自己所撰寫的語言特性，並且能夠適當地運用進行語言擴展，解決專案中的設計或技術問題。

而所有 Runtime 的特性都來自於一個重要核心 - **消息傳遞與轉發**。

