# BNTX图片解析
案例 pm0003_00_00_00.bntx pm0003_00_00_00_big.bntx
## 头文件分析BNTXHeader
首先还是用二进制打开文件，观察数据构成

pos用来标识文件读取地址

头文件包含前32个字节, pos = 0， 格式8sIH2BI2H2I (格式参考python struct的格式定义)

前4个字节转化为字符串为BNTX，为通用的magic header，之后4个字节都为0x00，是为magic header的分割标志 pos = 8

读取后4个字节int32, `00 01 04 00`, 为 262400 代表文件version, 版本pos来到12

读取后2个字节short, `FF FE`, 为文件大小端标识， 0xfffe 为小端，0xfeff则为大端pos来到14

读取后续2个byte, `0C`和`40`, 前一个为字节对齐偏移alignmentShift=12， 后一个为targetAddrSize=64，pos来到16

读取后4个字节int32, `BA 01 00 00`结果为442， 为文件名地址，pos来到20

读取后2个字节short, `00 00`, 结果为0, 定义为flag, pos来到22

继续读取2个字节short， `A0 01`, 结果为416，定义为firstBlkAddr，是第一段文件的地址，pos来到24

读取后4个字节int32， `00 10 01 00`，结果为69632，定义为relocAddr，为下一段文件的地址，pos来到28

读取剩余4个字节int32, `68 10 01 00`, 结果为69736，为文件的大小，pos来到32，头文件解析结束

### 总结
头文件格式为8sIH2BI2H2I，

**8s, string8**——长度8的字符串***magic header***`(8s, pos: 0-7)`，

**I, uint32**——文件版本***version***`(I, pos: 8-11)`,

**H, ushort**——大小端标识***endianess***`(H, pos: 12-13)`，

**2B, byte**——字节对齐偏移***alignmentShift***`(2B_1, pos: 14)`,

**2B, btye**——目标地址大小***targetAddrSize***`(2B-2, pos: 15)`,

**I, uint32**——文件名地址***filenameAdd***, `(I, pos: 16- 19)`

**2H, ushort**——文件flag***flag***, `(H, pos: (20-21))`

**2H, ushort**——第一段文件的地址***firstBlkAddr***, `(H, pos: (22-23))`

**2I, uint32**——下一段文件的地址***relocAddr***`(I, pos: 24-27)`,

**2I, uint32**——文件大小***filesize***`(I, pos: 28-31)`,

总计32字节

## 文件内容地址索引texContainer

pos = 32， 内存格式为4sI5qI4x

首先读取长度4字节的字符串`4E 58 20 20`, 转化为字符串是NX  pos来到36

读取4字节int32， `01 00 00 00`, 文件数量，记为count, 数值为1， pos来到40

5q, 后续读取5个long，各为8字节

1 `98 01 00 00 00 00 00 00`, 记为infoPtrsAddr，数值为408, pos来到48

2 `F0 0F 00 00 00 00 00 00`, 记为dataBlkAddr, 数值为4080, pos来到56

3 `D0 01 00 00 00 00 00 00`, 记为dictAddr， 纹理字典地址寻址, 数值为464, pos来到64

4 `58 00 00 00 00 00 00 00`, 记为memPoolAddr， 数值为88, pos来到72

5 `00 00 00 00 00 00 00 00`, 记为currMemPoolAddr， 数值为0, pos来到80

读取4字节uint32, `00 00 00 00`, 记为baseMemPoolAddr, 数值为0, pos来到84

最后4x代表 4个单位的填充字节 pos来到88

## 数据块头分析BlockHeader

pos 为头文件分析出的第一个块地址，**firstBlkAddr**, pos = 416, 格式4s2I4x

首先读取长度4字节的字符串`5F 53 54 52`，结果为_STR, 为magic header, pos来到420

2I，连续读取2个4字节的int32, `58 00 00 00`和`58 00 00 00`, 分别为下一个块地址和块的大小，都是88， pos= 428

之后跳过4个空白字节 pos = 432

## StringTable分析

在数据块头之后继续，pos为432

pos = 432， 读取4字节int32, `01 00 00 00`, 为table长度，count = 1, pos = 436

跳过4字节 pos = 440后续table具体内容，根据长度依次解析

### 条目名称解析

根据长度遍历，这里为1，pos = 440

读取2字节short, `13 00`, 为该条目的大小定义， size = 19, pos = 442

条目名称为之后长度为19的string, `70 6D 30 30 30 33 5F 30 30 5F 30 30 5F 30 30 5F 62 69 67`, 解析结果为**pm0003_00_00_00_big** pos = 442 + size = 461

如果有下一个条目，下一条目地址计算
```python
    pos += size + 3 # 地址变为为当前条目跳过字符长度和大小再跳过1个字节
    # pos - 1， 是当前条目结尾的数值， (pos - 1) | 1 + 1 将pos变更为小于等于pos的偶数
    pos = ((pos - 1) | 1) + 1 
```

### TexNameDict(纹理名称字典)

pos 是文件内容地址 texContainer.dictAddr = 464, 格式4sI

4s解析4字节的字符串`5F 44 49 43`, 为_DIC, magic header pos = 468

I解析4字节uint, `01 00 00 00`字典长度, count = 1 pos = 472

#### 字典条目解析

pos = 472 根据字典长度count 遍历
```python
    for i in range(0, count + 1):
        pos = pos + i * 16
        '''
        初始地址pos, 解析条目entry
        '''
```
i = 0，pos = 472, 格式I2Hq

I读取4字节uint32 `FF FF FF FF` 为2 ** 32 - 1 = 4294967295

2H连续读取2个2字节`01 00`和`00 00` 分别为**leftIdx = 1**和**rightIdx = 0**

q读取8字节int64 `B4 01 00 00 00 00 00 00` , 为**strTblEntryAddr**， 结果为464

i = 1, pos = 472 + 16 = 488, 同上解析uint32 `00 00 00 00` = 0, 2H `00 00`和`01 00`分别为**leftIdx = 0**和**rightIdx = 1** int 64 `B8 01 00 00 00 00 00 00` 为**strTblEntryAddr**， 440

## 纹理信息解析

初始地址pos = texContainer.infoPtrsAddr = 408

texContainer.count = 1

```python
for i in range(count):
    pos = readInt64(data, infoPtrsAddr + 8 * i, self.header.endianness) #
```
读取地址信息int64 `F8 01 00 00 00 00 00 00` pos = 504

从504开始读取4字节字符串`42 52 54 49` 'BRIT', 同样为一个magic header pos = 508

后续各读取4字节`F8 0D 00 00`和`F8 0D 00 00`， 应该是文件大小类似的内容，再分析结果为3576 pos = 516

跳过4个字节`00 00 00 00` pos = 520

具体纹理格式2B4H2x2I3i3I20x3IB3x8q

| Format |                     Bytes |     var name     |  value   |
| :----- | ------------------------: | :--------------: | :------: |
| B      |                      `09` |      flags       |    9     |
| B      |                      `02` |       dim        |    2     |
| H      |                   `00 00` |     tileMode     |    0     |
| H      |                   `00 00` |     swizzle      |    0     |
| H      |                   `01 00` |     numMips      |    1     |
| H      |                   `01 00` |    numSamples    |    1     |
| 2x     |                   `00 00` |      blank       |    -     |
| I      |             `01 1B 00 00` |     format_      |   6913   |
| I      |             `20 00 00 00` |   accessFlags    |    32    |
| i      |             `00 01 00 00` |      width       |   256    |
| i      |             `00 01 00 00` |      height      |   256    |
| i      |             `01 00 00 00` |      depth       |    1     |
| I      |             `01 00 00 00` |   arrayLength    |    1     |
| I      |             `03 00 00 00` |  textureLayout   |    3     |
| I      |             `07 00 01 00` |  textureLayout2  |  65543   |
| 20x    |               `00 00 ...` |      blank       |    -     |
| I      |             `00 00 01 00` |    image size    |  65536   |
| I      |             `00 02 00 00` |    alignment     |   512    |
| I      |             `02 03 04 05` |     _compSel     | 84148994 |
| B      |                      `01` |      imgDim      |    1     |
| 3x     |                `00 00 00` |      blank       |    -     |
| q      | `B8 01 00 00 00 00 00 00` |     nameAddr     |   440    |
| q      | `20 00 00 00 00 00 00 00` |    parentAddr    |    32    |
| q      | `98 04 00 00 00 00 00 00` |     ptrsAddr     |   1176   |
| q      | `00 00 00 00 00 00 00 00` |   userDataAddr   |    0     |
| q      | `98 02 00 00 00 00 00 00` |      texPtr      |   664    |
| q      | `98 03 00 00 00 00 00 00` |    texViewPtr    |   920    |
| q      | `00 00 00 00 00 00 00 00` | descSlotDataAddr |    0     |
| q      | `00 00 00 00 00 00 00 00` |   userDictAddr   |    0     |
```python
    compSel = [(_compSel >> (8 * i)) & 0xff for i in range(4)]
    readTexLayout = flags & 1 # flags 9 9 & 1 = 1
    sparseBinding = flags >> 1 # 9 >> 1 = 0b1001 >> 1 = 0b100 = 4
    sparseResidency = flags >> 2 # 9 >> 2 = 0b1001>> 2 = 0b10 = 1
    blockHeightLog2 = textureLayout & 7 # 3 & 7 = 3
    firstMipOffset = readInt64(data, ptrsAddr, '<') # 读取ptrsAddr地址处的int64
    # 00 10 00 00 00 00 00 00 结果为4096
```