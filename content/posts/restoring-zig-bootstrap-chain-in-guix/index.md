+++
title = "用 GNU Guix 重建 Zig 自舉鏈"
author = ["Hilton Chain"]
description = "Zig 打包記（上）。"
date = 2025-01-28T21:07:00+08:00
lastmod = 2025-01-29T01:22:00+08:00
tags = ["Guix", "Zig"]
categories = ["notes"]
draft = false
image = "cover.png"
+++

2024 年末，連同 48 個中間版本一起，Zig 0.13 終於進入了 GNU Guix，目前已在 `x86_64-linux` 和 `aarch64-linux` 可用。Guix 的 `zig-build-system` 也一併改進：支持 Zig 內置包管理器、完善交叉編譯支持、解決 Zig 包再現性問題。其實 2023 年我就有嘗試添加新版本 Zig，可惜因爲自己缺乏瞭解作罷，如今能夠完成實屬機緣。

Zig 工具鏈一開始由 C++ 實現，到 0.10 版本時完成並啓用 Zig 實現，有了分三階段的自舉流程：

```text
  \
stage1 (C++)
    \
  stage2 (Zig)
     |
  stage3 (Zig)
```

但在 0.11 版本，[C++ 實現移除](https://ziglang.org/news/goodbye-cpp/)，Zig 通過編譯核心工具鏈到 WebAssembly 解決自舉問題，流程隨之更新：

```text
zig1.wasm
    | WebAssembly -> C
 stage1 (C)
     \ Zig -> C
   stage2 (C)
       \
     stage3 (Zig)
```

（其中 `zig1.wasm` 可經 stage2 或 stage3 編譯）

```text
   stage2
   stage3
     / Zig -> WebAssembly
zig1.wasm
```

新流程利用 Zig 工具鏈自身特性，實現得很巧妙，缺點則是形成了循環依賴：Zig 自舉從 `zig1.wasm` 開始，編譯 `zig1.wasm` 又需要 Zig。打包 Zig 的首要問題就是如何解開這一循環。

最合適的方法顯然是用其他語言實現 Zig，但目前還沒有人完成。另一種方法打包者大概比較熟悉：從 Zig 開發歷史入手，找出最後一個同時支持兩種自舉流程的 Zig 版本，以此重建曾用於開發過程的整條自舉鏈。

可惜翻遍 Git 倉庫提交歷史，也找不到這種條件——Zig 急於擺脫 C++ 實現，以致沒有過渡階段明確記錄在案。我在 2023 年嘗試時打算先擱置這個問題，用最初的 `zig1.wasm` 完成自舉鏈，再回過頭來解決。但類似問題也出現在後續一些 `zig1.wasm` 更新上，我不瞭解 Zig，也就失去動力了。

```text
   \
 stage1 (C++)
     \
   stage2 (Zig)
      |
   stage3 (Zig)
     ?
zig1.wasm
```

轉機出現在 2024 年 11 月，Motiejus Jakštys 開始[記錄自己重建自舉鏈的嘗試](https://ziggit.dev/t/building-self-hosted-from-the-original-c-implementation/6607)，計劃一直到 Zig 0.13，Ekaitz Zarraga 在 [#74217](https://issues.guix.gnu.org/74217) 分享了這一消息。在我第一次檢查進度時，Motiejus 就已經解決了先前阻礙我的問題，這讓我看到了希望。

我從 11 月 8 日開始着手在 Guix 中復刻 Motiejus 的工作，翌日致信說明來意，隨後幾天追趕進度，到 11 日時終於趕上，便再次致信報告進度。此時自舉鏈離 Zig 0.12 已經很近，我就接着繼續到了 0.13。至此一半工作就完成了，幾天後 Motiejus 發佈博文 [_Zig Reproduced Without Binaries_](https://jakstys.lt/2024/zig-reproduced-without-binaries/)。

```text
$ guix graph --path zig@0.13.0 -e "(@@ (gnu packages zig) zig-0.10.0-610)"
zig@0.13.0
zig@0.12.0-109.b7799ef
zig@0.12.1
zig@0.11.0-3604.7611d90
zig@0.11.0-3506.fb192df
zig@0.11.0-3503.17673dc
zig@0.11.0-3501.9b2345e
zig@0.11.0-3344.e646e01
zig@0.11.0-3245.4f782d1
zig@0.11.0-1967.6beae6c
zig@0.11.0-761.9a09651
zig@0.11.0-702.63bd2bf
zig@0.11.0-638.9763573
zig@0.11.0-631.2178089
zig@0.11.0-587.6bd54a1
zig@0.11.0-494.a8d2ed8
zig@0.11.0-384.88f5315
zig@0.11.0-149.7a85ad1
zig@0.11.0
zig@0.10.0-3985.47d5bf2
zig@0.10.0-3980.4bce7b1
zig@0.10.0-3813.21ac0be
zig@0.10.0-3807.be0c699
zig@0.10.0-3728.a4d1eda
zig@0.10.0-3726.a6c8ee5
zig@0.10.0-3660.22c6b6c
zig@0.10.0-2838.a8de15f
zig@0.10.0-2797.35d82d3
zig@0.10.0-2571.31738de
zig@0.10.0-2566.e2fe190
zig@0.10.0-2558.d3a237a
zig@0.10.0-1891.ac1b0e8
zig@0.10.0-1888.c839c18
zig@0.10.0-1713.09a84c8
zig@0.10.0-1712.705d2a3
zig@0.10.0-1681.0bb178b
zig@0.10.0-1657.321ccbd
zig@0.10.0-1638.7199d7c
zig@0.10.0-1506.f16c10a
zig@0.10.0-1497.a9b6830
zig@0.10.0-1073.4c1007f
zig@0.10.0-1027.a43fdc1
zig@0.10.0-962.622311f
zig@0.10.0-961.54160e7
zig@0.10.0-853.2a5e142
zig@0.10.0-851.aac2d6b
zig@0.10.0-748.08b2d49
zig@0.10.0-747.7b2a936
zig@0.10.0-722.d10fd78
zig@0.10.0-675.9d93b2c
zig@0.10.0-610.e7d2834
```

重建 Zig 自舉鏈最困難的階段是在 0.11 以前，新自舉流程剛開始應用的時候，要想順利更新 `zig1.wasm` 往往需要未記錄的改動。很難想象如果沒有 Motiejus 事先完成這一部分我該怎麼繼續。Zig 開發者大概也意識到了這個問題，後來的更新通常會在提交歷史留下過渡階段，提交信息也往往解釋了更新步驟，這種做法很好。

我之後整理也發現，整條自舉鏈的構建模式的確在 0.11 前就已經穩定成型了，因此我重構了 Guix 方面構建參數，現在配合 Zig 開發者留下的提示，基本不需要試錯就能繼續自舉鏈了。

完成基礎打包只是一半，確保軟件可用纔是重點，這部分留待下篇博文再講。我也會將兩篇博文寫作英文發往 Guix Blog，敬請期待。
