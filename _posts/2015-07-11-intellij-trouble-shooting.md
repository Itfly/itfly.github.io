---
title: Intellij trouble shooting
layout: post
---

Intellij 打开某个非常大的java文件（如使用thrift自动生成的java文件）时，会超出其的默认配置，导致 Intellij 报错：

    File size exceeds configured limit(2560000). Code insight features are not available.

解决办法是修改Intellij 的 `idea.max.intellisense.filesize`
属性配置，位于：

    /Applications/{Intellij Home}/bin/idea.properties

由于 Intellij 默认内存堆栈的设置是128MB, 打开多个项目时能够明显感到Intellij很卡。此时，可以增加其内存堆的大小，位于：

    /Applications/{Intellij Home}/bin/idea.vmoptions

