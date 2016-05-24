---
layout: post
title: "找出引起异常的文字"
date: 2016-05-25 00:23:23 +0800
comments: true
categories: Debug
---

大家可能都遇到过，一些比较特殊的字符，在排版的时候，或者在渲染的时候，会抛出异常。
当我们调试的时候，加了异常断点，就会触发断点。
这次主要是记录一下，当出现这种异常的时候，怎样把引起这个异常的文字找出来。

我们先看一下出现这种异常时的堆栈信息
<!--MORE-->

```
(lldb) bt
* thread #1: tid = 0x70ed65, 0x0000000118914c6b libc++abi.dylib`__cxa_throw, queue = 'com.apple.main-thread', stop reason = breakpoint 3.1
    frame #0: 0x0000000118914c6b libc++abi.dylib`__cxa_throw
    frame #1: 0x00000001311c782f libType1Scaler.dylib`TType1LWFNMultipleMasterHFMXTable::TType1LWFNMultipleMasterHFMXTable(TFontObjectSurrogate const&, int, short, short, int*) + 121
    frame #2: 0x00000001311c7756 libType1Scaler.dylib`TType1LWFNFont::CreateHFMXTable() const + 86
    frame #3: 0x00000001311c76f5 libType1Scaler.dylib`TType1HFMXTableSurrogate::TType1HFMXTableSurrogate(TType1PSFont const&) + 31
    frame #4: 0x00000001311c6ec9 libType1Scaler.dylib`TType1PSFont::GetFontInstanceSpec(TType1Transform const&, FixedPoint&, FixedPoint&, FixedPoint&, FixedPoint&, FixedPoint&, FixedRectangle&, FontMetrics&) const + 647
    frame #5: 0x00000001311c6be4 libType1Scaler.dylib`TType1Strike::GetSpecs(StrikeSpecs&) const + 100
    frame #6: 0x00000001311bff2d libType1Scaler.dylib`Type1GetStrikeSpecs(unsigned int, TStrikeDescription const*, StrikeSpecs*) + 48
    frame #7: 0x000000011aba87a0 libFontParser.dylib`TConcreteFontScaler::GetFontInfo(FPFontInfo*) const + 48
    frame #8: 0x000000011ab80525 libFontParser.dylib`TFPFont::FillFontInfo(TFPFontInfo&) const + 203
    frame #9: 0x000000011ab80412 libFontParser.dylib`TFPFont::GetFontInfo() const + 46
    frame #10: 0x000000011ab803de libFontParser.dylib`FPFontIsMonospaced + 9
    frame #11: 0x0000000113bf63cd CoreGraphics`get_font_info + 244
    frame #12: 0x0000000113b84b47 CoreGraphics`get_font_info + 63
    frame #13: 0x0000000113b84af9 CoreGraphics`CGFontGetNumberOfGlyphs + 9
    frame #14: 0x00000001168131ed CoreText`TBaseFont::CopyGraphicsFont() const + 43
    frame #15: 0x0000000116863446 CoreText`TBaseFont::CopyTable(unsigned int) const + 188
    frame #16: 0x00000001168476fc CoreText`TBaseFont::GetCmapTable() const + 60
    frame #17: 0x000000011685d7a2 CoreText`TBaseFont::GetUnicodeEncoding() const + 58
    frame #18: 0x000000011685c367 CoreText`TBaseFont::GetUnicodeCmapSubHeader(void const**) const + 29
    frame #19: 0x000000011685c506 CoreText`TBaseFont::CopyCharacterSet() const + 206
    frame #20: 0x000000011680a2c2 CoreText`TBaseFont::GetCharacterSetInternal() const + 42
    frame #21: 0x0000000116808211 CoreText`CompareCharSet(__CFCharacterSet const*, TBaseFont const*) + 17
    frame #22: 0x000000011680ff1d CoreText`TDescriptorSource::CopyDescriptorsForRequestFromArray(__CFArray const*, __CFDictionary const*, CFComparisonResult (*)(void const*, void const*, void*), void*, unsigned long, bool) const + 1325
    frame #23: 0x000000011680e47e CoreText`TDescriptorSource::CopyDescriptorsForRequest(__CFDictionary const*, __CFSet const*, CFComparisonResult (*)(void const*, void const*, void*), void*, unsigned long, TCFRef<__CFArray const*>*) const + 2374
    frame #24: 0x000000011680d747 CoreText`TDescriptorSource::CopySystemWideFallbackDescriptorForCharacters(TBaseFont const&, unsigned short const*, long, UIFontFlag, TCFRef<__CFArray const*>*) const + 681
    frame #25: 0x000000011680ee03 CoreText`TDescriptorSource::CopySystemWideFallbackDescriptorForString(TBaseFont const&, __CFString const*, CFRange, UIFontFlag, TCFRef<__CFArray const*>*) const + 199
    frame #26: 0x00000001167f9ab3 CoreText`TFontCascade::CreateSystemWideFallback(__CTFont const*, __CFString const*, CFRange) const + 115
    frame #27: 0x00000001167f99ad CoreText`TFontCascade::CreateFallback(__CTFont const*, __CFString const*, CTEmojiPolicy) const + 1525
    frame #28: 0x00000001167c9204 CoreText`TGlyphEncoder::AppendUnmappedCharRun(unsigned int, TCFRef<CTRun*>&, __CTFont const*, CFRange, CFRange, TGlyphList<TDeletedGlyphIndex>&, TGlyphList<TDeletedGlyphIndex>&, TFontCascade const&, TGlyphEncoder::ClusterMatching) + 510
    frame #29: 0x00000001167c8e93 CoreText`TGlyphEncoder::RunUnicodeEncoderRecursively(unsigned int, TCFRef<CTRun*>&&, __CTFont const*, CFRange, TGlyphList<TDeletedGlyphIndex>&, TGlyphList<TDeletedGlyphIndex>&, TFontCascade const*, TGlyphEncoder::ClusterMatching, bool) + 1819
    frame #30: 0x00000001167c8703 CoreText`TGlyphEncoder::RunUnicodeEncoder(TCFRef<CTRun*>&&, __CTFont const*, CFRange, TGlyphList<TDeletedGlyphIndex>&, TFontCascade const*) + 137
    frame #31: 0x00000001167c82e0 CoreText`TGlyphEncoder::EncodeChars(CFRange, TAttributes const&, TGlyphList<TDeletedGlyphIndex>&, TGlyphEncoder::Fallbacks) + 1684
    frame #32: 0x00000001167df8a8 CoreText`TTypesetterAttrString::Initialize(__CFAttributedString const*) + 526
    frame #33: 0x00000001167cbe34 CoreText`CTLineCreateWithAttributedString + 63
    frame #34: 0x000000011cdffe3e UIFoundation`__NSStringDrawingEngine + 9479
    frame #35: 0x000000011cdfd824 UIFoundation`-[NSString(NSExtendedStringDrawing) drawWithRect:options:attributes:context:] + 167
    frame #36: 0x0000000114d69b4b UIKit`-[UILabel _drawTextInRect:baselineCalculationOnly:] + 6596
    frame #37: 0x0000000114d6788a UIKit`-[UILabel drawTextInRect:] + 669
    frame #38: 0x0000000114d69caf UIKit`-[UILabel drawRect:] + 100
    frame #39: 0x0000000114bab3e5 UIKit`-[UIView(CALayerDelegate) drawLayer:inContext:] + 495
    frame #40: 0x00000001148237ea QuartzCore`-[CALayer drawInContext:] + 257
    frame #41: 0x000000011508ab47 UIKit`-[_UILabelContentLayer drawInContext:] + 197
    frame #42: 0x0000000114712faa QuartzCore`CABackingStoreUpdate_ + 2725
    frame #43: 0x00000001148234d4 QuartzCore`___ZN2CA5Layer8display_Ev_block_invoke + 68
    frame #44: 0x000000011482334a QuartzCore`CA::Layer::display_() + 1614
    frame #45: 0x0000000114817e87 QuartzCore`CA::Layer::display_if_needed(CA::Transaction*) + 293
    frame #46: 0x0000000114817f17 QuartzCore`CA::Layer::layout_and_display_if_needed(CA::Transaction*) + 35
    frame #47: 0x000000011480c3c9 QuartzCore`CA::Context::commit_transaction(CA::Transaction*) + 277
    frame #48: 0x000000011483a086 QuartzCore`CA::Transaction::commit() + 486
    frame #49: 0x000000011483a7f8 QuartzCore`CA::Transaction::observer_callback(__CFRunLoopObserver*, unsigned long, void*) + 92
    frame #50: 0x0000000118485c37 CoreFoundation`__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__ + 23
    frame #51: 0x0000000118485ba7 CoreFoundation`__CFRunLoopDoObservers + 391
    frame #52: 0x000000011847b7fb CoreFoundation`__CFRunLoopRun + 1147
    frame #53: 0x000000011847b0f8 CoreFoundation`CFRunLoopRunSpecific + 488
    frame #54: 0x0000000119682ad2 GraphicsServices`GSEventRunModal + 161
    frame #55: 0x0000000114af0f09 UIKit`UIApplicationMain + 171

```

大致可以看到， 是UILabel在 **drawTextInRect**时，获取某些字元信息时出异常了。
我们的目的就是想看一下到底是什么文字引起这个问题的。

我们都知道 text 是存放在 UILabel.text 的属性上，第一印象都认为  ``` po self.text ``` 就可以看到罪魁祸首了。

OK， 我们看一下实际输出结果。


```
(lldb) po self.text
error: use of undeclared identifier 'self'
error: 1 errors parsing expression
```

可以看到提示，没有找到 self ，因为当前栈帧已经去到

```
    frame #0: 0x0000000118914c6b libc++abi.dylib`__cxa_throw
    frame #1: 0x00000001311c782f libType1Scaler.dylib`TType1LWFNMultipleMasterHFMXTable::TType1LWFNMultipleMasterHFMXTable(TFontObjectSurrogate const&, int, short, short, int*) + 121
 
 ``` 
 所以提示没找到self也是很正常， 那么我们切换一下当前堆栈，把它切换到 ``` [UILabel drawRect:] ``` 这个函数里面:
 
 ```
 UIKit`-[UILabel drawRect:]:
    0x114d69c4b <+0>:   pushq  %rbp
    0x114d69c4c <+1>:   movq   %rsp, %rbp
    0x114d69c4f <+4>:   pushq  %rbx
    0x114d69c50 <+5>:   subq   $0x48, %rsp
    0x114d69c54 <+9>:   movq   %rdi, %rbx
    0x114d69c57 <+12>:  testq  %rbx, %rbx
    0x114d69c5a <+15>:  je     0x114d69c71               ; <+38>
    0x114d69c5c <+17>:  movq   0xa6c4ad(%rip), %rdx      ; "bounds"
    0x114d69c63 <+24>:  leaq   -0x30(%rbp), %rdi
    0x114d69c67 <+28>:  movq   %rbx, %rsi
    0x114d69c6a <+31>:  callq  0x1155c2af6               ; symbol stub for: objc_msgSend_stret
    0x114d69c6f <+36>:  jmp    0x114d69c7c               ; <+49>
    0x114d69c71 <+38>:  xorps  %xmm0, %xmm0
    0x114d69c74 <+41>:  movaps %xmm0, -0x20(%rbp)
    0x114d69c78 <+45>:  movaps %xmm0, -0x30(%rbp)
    0x114d69c7c <+49>:  movq   0xa7cdd5(%rip), %rsi      ; "drawTextInRect:"
    0x114d69c83 <+56>:  movq   -0x18(%rbp), %rax
    0x114d69c87 <+60>:  movq   %rax, 0x18(%rsp)
    0x114d69c8c <+65>:  movq   -0x20(%rbp), %rax
    0x114d69c90 <+69>:  movq   %rax, 0x10(%rsp)
    0x114d69c95 <+74>:  movq   -0x30(%rbp), %rax
    0x114d69c99 <+78>:  movq   -0x28(%rbp), %rcx
    0x114d69c9d <+82>:  movq   %rcx, 0x8(%rsp)
    0x114d69ca2 <+87>:  movq   %rax, (%rsp)
    0x114d69ca6 <+91>:  movq   %rbx, %rdi
    0x114d69ca9 <+94>:  callq  *0xaf0531(%rip)           ; (void *)0x0000000117c64800: objc_msgSend
    0x114d69caf <+100>: addq   $0x48, %rsp
    0x114d69cb3 <+104>: popq   %rbx
    0x114d69cb4 <+105>: popq   %rbp
    0x114d69cb5 <+106>: retq   
```
 
再看看结果：

```
(lldb) po self.text
error: use of undeclared identifier 'self'
error: 1 errors parsing expression

```

同样，也是没有找到self标识符。但是我们从上面的汇编代码可以看到，在调用 drawTextInRext 的时候，实际上最后也是调用 ```objc_msgSend``` 这个函数。

然后我们再看一下 ```objc_msgSend ``` 的函数原型：

```
OBJC_EXPORT id objc_msgSend(id self, SEL op, ...)

```

可以清楚的看到，传给objc_msgSend的第一个参数就是 self 对象。

而且从

```
    0x114d69c7c <+49>:  movq   0xa7cdd5(%rip), %rsi      ; "drawTextInRect:"
    0x114d69c83 <+56>:  movq   -0x18(%rbp), %rax
    0x114d69c87 <+60>:  movq   %rax, 0x18(%rsp)
    0x114d69c8c <+65>:  movq   -0x20(%rbp), %rax
    0x114d69c90 <+69>:  movq   %rax, 0x10(%rsp)
    0x114d69c95 <+74>:  movq   -0x30(%rbp), %rax
    0x114d69c99 <+78>:  movq   -0x28(%rbp), %rcx
    0x114d69c9d <+82>:  movq   %rcx, 0x8(%rsp)
    0x114d69ca2 <+87>:  movq   %rax, (%rsp)
    0x114d69ca6 <+91>:  movq   %rbx, %rdi
    0x114d69ca9 <+94>:  callq  *0xaf0531(%rip)           ; (void *)0x0000000117c64800: objc_msgSend
    
```
这里可以看到， 在 drawRect 里面，实际上是调用了 ```drawTextInRect```,根据一些调用约定，可以判断 ```objc_msgSend``` 的第一个参数是最后压栈的，这时候我们可以猜测就是 **rbx**的寄存器的值。

我们可以通过 **register read** 指令查看当前寄存器的值。

```
(lldb) register read
General Purpose Registers:
       rbx = 0x00007f9f833d71e0
       rbp = 0x00007fff5096bf70
       rsp = 0x00007fff5096bf20
       r12 = 0x00007f9f83102850
       r13 = 0x00007f9f833d71e0
       r14 = 0x0000000000000020
       r15 = 0x00007f9f80f119f0
       rip = 0x0000000114d69caf  UIKit`-[UILabel drawRect:] + 100
13 registers were unavailable.

```

然后我们尝试一下输出 rbx 的对象类型（如果它是一个对象的话）

```
(lldb) po [(id)$rbx class]
UILabel

```
可以看到rbx就是我们要找的那个UILabel对象，既然找到UILabel，那么就很容易找到相应的 text 了。

```
(lldb) po [(id)$rbx text]
ᘡ ‍Amy🌀晴天冠 ୨୧˙˳⋆﻿

```

可以看到 ** ᘡ ‍Amy🌀晴天冠 ୨୧˙˳⋆﻿ ** 是这个文本引起异常。

如果不好判断哪些寄存器是存放 self 对象的话，可以一个一个打印一下类型试试。


###总结
* 很多时候异常了，而且不在我们的函数里面，这个时候直接取 self 是取不到的。
* 所有方法最终都调用了```objc_msgSend```， 调用的对象会作为第一个参数（self）传给它。
* 可以通过查看寄存器命令 ```register read ```， 获取到self地址。
* 也可以通过```frame``` 指令查看当前栈的参数信息。 具体可以看看 ``` help frame ``` 的输出信息。




 