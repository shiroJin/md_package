---
title: PNG图片编解码探索
---
在应用开发过程中会碰到加载大图占用内存过多导致内存溢出的情况。一张图片占用的内存大小是有迹可循的。比如一个RGBA的像素，占用的内存大小为4byte。一张100x100像素的图片解压缩后占用的内存大小大约为100x100x4byte的内存（这里暂时不考虑图片信息之类的内存占用，仅举例说明）。

对于一个手机应用来说，图片显示的区域不会太大。假如图片显示的区域为1000x1000pixel，显示的图片原始数据为1wx1w的图片，那这张图片的像素数量远远超过了屏幕能显示的像素数量。那大部分的像素其实是没有价值的。反而解压缩原始尺寸后的图片占用的内容是非常庞大的。

于是乎就可以想到这么一个方案——在显示图片的时候根据原始图片生成一个满足显示需求的小图，这样就可以减少内存的占用。这也是现在业内使用的主流方案。

在这里有一个问题，生成小图的方式是可以有效的解决之后显示图片所需要的内存。那么在生成小图这个阶段，是不是需要将原始的图片数据先读取到内存中，对原始数据解压缩，然后再生成相应的压缩图片。那在这一步中，图片读取和压缩的过程一定会带来一个内存峰值。也来到本文的一个主题，图片的解压缩存不存一种方式———边读取图片数据边生成采样率低的图片，从而保证处理过的数据的内存可以及时的释放。

<!-- more -->

这里只探索一般格式的PNG图片。

# PNG格式图片

便携式网络图形是一种[无损压缩](https://baike.baidu.com/item/无损压缩/2817566)的位图片形格式，其设计目的是试图替代[GIF](https://baike.baidu.com/item/GIF)和[TIFF](https://baike.baidu.com/item/TIFF/2106)文件格式，同时增加一些[GIF文件格式](https://baike.baidu.com/item/GIF文件格式)所不具备的特性。PNG使用从LZ77派生的无损数据压缩算法，一般应用于JAVA程序、[网页](https://baike.baidu.com/item/网页/99347)或S60程序中，原因是它[压缩比](https://baike.baidu.com/item/压缩比/109904)高，生成文件体积小。

## 数据结构

对于一个PNG文件来说，其文件头总是由位固定的字节来描述的，HEX： 89 50 4E 47 0D 0A 1A 0A。其中 50 4E 47对应的ASCII码是P N G。第一个字节0x89超出了ASCII字符的范围，这是为了避免某些软件将PNG文件当做文本文件来处理。文件中剩余的部分由多个PNG的数据块组成。一个标准的PNG文件结构应该如下：

> PNG文件标志 |  PNG数据块   …   PNG数据块 

### 数据块结构

PNG文中，每个数据块都由4个部分组成，如下

| **名称**                       | **字节数** | **说明**                                                     |
| ------------------------------ | ---------- | ------------------------------------------------------------ |
| Length (长度)                  | 4字节      | 指定数据块中数据域的长度，其长度不超过(231－1)字节。length的值是指chunkData的长度。 |
| Chunk Type Code (数据块类型码) | 4字节      | 数据块类型码由ASCII字母(A-Z和a-z)组成                        |
| Chunk Data (数据块数据)        | 可变长度   | 存储按照Chunk Type Code指定的数据                            |
| CRC (循环冗余检测)             | 4字节      | 存储用来检测是否有错误的循环冗余码                           |

### Chunk Type

png文件中有很多种数据块，不同符号的数据块涵盖的信息不同。

| **PNG文件格式中的数据块** |                        |              |            |                  |
| ------------------------- | ---------------------- | ------------ | ---------- | ---------------- |
| **数据块符号**            | **数据块名称**         | **多数据块** | **可选否** | **位置限制**     |
| IHDR                      | 文件头数据块           | 否           | 否         | 第一块           |
| cHRM                      | 基色和白色点数据块     | 否           | 是         | 在PLTE和IDAT之前 |
| gAMA                      | 图像γ数据块            | 否           | 是         | 在PLTE和IDAT之前 |
| sBIT                      | 样本有效位数据块       | 否           | 是         | 在PLTE和IDAT之前 |
| PLTE                      | 调色板数据块           | 否           | 是         | 在IDAT之前       |
| ...                       | ….                     | *            | *          | ...              |
| IDAT                      | 图像数据块             | 是           | 否         | 与其他IDAT连续   |
| tIME                      | 图像最后修改时间数据块 | 否           | 是         | 无限制           |
| tEXt                      | 文本信息数据块         | 是           | 是         | 无限制           |
| zTXt                      | 压缩文本数据块         | 是           | 是         | 无限制           |
| ...                       | ...                    | *            | *          | *                |
| IEND                      | 图像结束数据           | 否           | 否         | 最后一个数据块   |

<b> IHDR</b>

IHDR块中包含图片最基本的信息，包括宽、高、色域、压缩算法等信息。

| **域的名称**       | **字节数** | **说明**                                                     |
| ------------------ | ---------- | ------------------------------------------------------------ |
| Width              | 4 bytes    | 图像宽度，以像素为单位                                       |
| Height             | 4 bytes    | 图像高度，以像素为单位                                       |
| Bit depth          | 1 byte     | 图像深度：  索引彩色图像：1，2，4或8  灰度图像：1，2，4，8或16  真彩色图像：8或16 |
| ColorType          | 1 byte     | 颜色类型： 0：灰度图像, 1，2，4，8或16  2：真彩色图像，8或16  3：索引彩色图像，1，2，4或8  4：带α通道数据的灰度图像，8或16  6：带α通道数据的真彩色图像，8或16 |
| Compression method | 1 byte     | 压缩方法(LZ77派生算法)                                       |
| Filter method      | 1 byte     | 滤波器方法                                                   |
| Interlace method   | 1 byte     | 隔行扫描方法： 0：非隔行扫描  1： Adam7(由Adam M. Costello开发的7遍隔行扫描方法) |

<b> IDAT</b>

数据包存放的位置，存储实际的数据，在数据流中可以包含多个连续顺序的图像数据块。

<b> IEND</b>

表示数据段的结束。

## PNG Datasteam

IHDR块中Compression现在国际标准定义的只有0。方法0使用的LZ77算法，使用滑动窗口的deflate/inflate算法。该算法是由计算机天才Phil Katz创造的，也是让数据压缩技术免费的人物。

IDATA中的数据就是Deflate算法压缩后的zlib格式的数据。zlib的格式如下：

| zlib compression method/flags code | 1 byte  |
| ---------------------------------- | ------- |
| Additional flags/check bits        | 1 byte  |
| Compressed data blocks             | n bytes |
| Check value                        | 4 bytes |

##### compression method/flags code

其中method/flags code值始终为8，因为国际标准的压缩算法目前只有一种。Additional flags/check bits应该设置为0。

##### Additional flags/check bits 

一般为0。

##### Compressed data blocks

compressed data blocks就是压缩后的数据。

##### Check value

最后check value存在datasteam的最后面，Adler32算法计算的未压缩前的数据，来确认数据是否被损坏。block可以有多个，而且每个block的大小不可以不一样。

### Block结构

再把焦点聚焦到block上，每个block的开头都有3bits的头：

```json
first bit       BFINAL
next 2 bits     BTYPE
```

BFINAL表示是否是最后一个block。BTYPE的2个bits代表了数据如何压缩，有如下几种值

```none
00 - no compression
01 - compressed with fixed Huffman codes
10 - compressed with dynamic Huffman codes
11 - reserved (error)
```

##### Non-compressed block (BTYPE=00)

> BTYPE所在字节的余下bit会被忽略，LEN表示数据的字节长度，NLEN是LEN的一的[补数](https://baike.baidu.com/item/%E4%B8%80%E7%9A%84%E8%A1%A5%E6%95%B0/19701457?fr=aladdin)。之后的数据格式如下：

```
  0   1   2   3   4...
+---+---+---+---+================================+
|  LEN  | NLEN  |... LEN bytes of literal data...|
+---+---+---+---+================================+
```

##### Compressed with fixed Huffman codes (BTYPE=01) & Compressed with dynamic Huffman codes (BTYPE=10)

> 上述两中压缩方式的区别在于是静态哈夫曼编码还是动态哈夫曼编码。如果是静态哈夫曼编码，哈夫曼表不需要存储在数据中，使用时只需要查询标准的哈夫曼表即可。而动态哈夫曼表存在数据之中，由图片原始数据生成。

以上3bits表示了block的相关属性，余下的数据就是真正的数据。解码数据的步骤如下：

```
do
   read block header from input stream.
   if stored with no compression
      skip any remaining bits in current partially
         processed byte
      read LEN and NLEN (see next section)
      copy LEN bytes of data to output
   otherwise
      if compressed with dynamic Huffman codes
         read representation of code trees (see
            subsection below)
      loop (until end of block code recognized)
         decode literal/length value from input stream
         if value < 256
            copy value (literal byte) to output stream
         otherwise
            if value = end of block (256)
               break from loop
            otherwise (value = 257..285)
               decode distance from input stream

               move backwards distance bytes in the output
               stream, and copy length bytes from this
               position to the output stream.
      end loop
while not last block
```

可以观察到解压数据是逐块进行的，虽然存在拷贝前部分数据的操作，但滑动窗口的大小是有限制的，也就是说超出滑动窗口的内容不会被后续解码所使用。

那么对于PNG的解码就存在一种可能性，我们可以将超过滑动窗口的解码数据先行处理，然后释放掉原本的数据，保留加工后的数据，也就可以达到控制内存的峰值的作用。

# PNG图片解码

通过iOS代码生成了一行纯红色1x1的图片，得到的png文件数据如下

> 89504e47 0d0a1a0a 
> 0000000d 49484452 00000001 00000001 08060000 001f15c4 89
> 00000001 73524742 00aece1ce9
> 0000000d <font color="red"> 49444154</font> <font color="blue">081d63f8 cfc0f01f 00050001 ff</font>c80f6216
> 00000000 49454e44 ae426082

从hex文件中可以看到IDATA块内容的大小为13字节（蓝色标记内容）。

> 081d<font color="blue">63f8 cfc0f01f 00</font>050001 ff

compression method/flags code：08

Additional flags/check bits：1d

Check value: 050001ff

Data blocks: 63f8 cfc0f01f 00

对data blocks中的数据进行处理：

> 110(final, fixed huffman code)00110 000(0x00)11111 1111(0xFF)0011 0000(0x00)0011 0000(0x00)1111 11111(0xFF)000 0000(256)0000 

通过inflate算法解析可以得到原始数据如下：

>  0x00 0xFF 0x00 0x00 0xFF 

原图的过滤器为nonefilter，色域为RGBA。可以看出这是一个红色的像素点。

# 参考

[W3C](https://www.w3.org/TR/PNG/#figure49)

[https://www.ietf.org/rfc/rfc1951.txt](https://www.ietf.org/rfc/rfc1950.txt)

[https://www.ietf.org/rfc/rfc1951.txt](https://www.ietf.org/rfc/rfc1951.txt)
