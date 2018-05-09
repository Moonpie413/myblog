---
title: netty解码器
date: 2018-05-03 21:31:51
tags: netty
---

记录netty中的 `LengthFieldBasedFrameDecoder` 与 `LengthFieldPrepender` 用法

<!-- more -->

LengthFieldBasedFrameDecoder
-----------------------------

数据在网络中是以流的形式传输的, 但是传输/接收方希望数据是完整的而不是接收到片段,
因此netty通过 `decoder` 的形式根据不同的协议将无须的二进制流拆成需要的数据格式

LengthFieldBasedFrameDecoder 可以将用于解码各种基于长度的协议

拿其中一个构造函数举例

```java
/**
     * Creates a new instance.
     *
     * @param maxFrameLength
     *        the maximum length of the frame.  If the length of the frame is
     *        greater than this value, {@link TooLongFrameException} will be
     *        thrown.
     * @param lengthFieldOffset
     *        the offset of the length field
     * @param lengthFieldLength
     *        the length of the length field
     * @param lengthAdjustment
     *        the compensation value to add to the value of the length field
     * @param initialBytesToStrip
     *        the number of first bytes to strip out from the decoded frame
     */
    public LengthFieldBasedFrameDecoder(
            int maxFrameLength,
            int lengthFieldOffset, int lengthFieldLength,
            int lengthAdjustment, int initialBytesToStrip) {
        this(
                maxFrameLength,
                lengthFieldOffset, lengthFieldLength, lengthAdjustment,
                initialBytesToStrip, true);
    }
```

- 第一个参数表示帧长度,
- 第二个参数表示长度位的起始索引
- 第三个参数表示长度为长度
- 第四个参数表示开始截取位置


