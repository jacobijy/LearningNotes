# 文件名
another_name.dat another_name.tbl
# 解析方法
首先解析another_name.tbl，相当于文本的目录名
## 解析another_name.tbl
用二进制打开another_name.tbl，前4个字节转化为字符串为AHTB，为通用的magic header

然后读取int32 5-8字节，为该压缩文本的条目数目0x70000000，小端读取为112

确定该文本的条目数量，0x7000, 为112

循环读取后续字节，8字节为条目的hash值

第一个条目为例，即9-16字节值，为A5 E7 0D 34 37 B4 80 6A

后续两位为条目名称长度，0x1400 长度20 msg_another_name_00，字节串以0x00结束

验证hash值
```python
def HashFNV1_64(word):
    """Fowler-Noll-Vo hash function; 64-bit"""
    fnvPrime_64 = 0x100000001b3
    offsetBasis_64 = 0xCBF29CE484222645
    hash = offsetBasis_64
    for c in word:
        hash = hash ^ ord(c)
        # Cast hash to at 64-bit value
        hash = (hash * fnvPrime_64) % 2**64
    return hash
HashFNV1_64('msg_another_name_00')
```
结果为 7674331914228852645 与hash值一致

每个条目的长度可计算 8 + 2 + length = 30

## 解析another_name.dat
同样用二进制打开another_name.dat进行观察

前两个字节，TextSections, 文本段落， 值为1

读取条目数量，后续2字节，0x6f00， 数量为111

文本长度0x4，后续4字节，0xc4150000, 值为5572

后续4字节0x8，应该为初始标志，为0x00000000

后续4字节0xc, 为0x10000000, 值为16，为文本段落分割间距

后续开始正文0x10，第一个为文本段落长度，因为文本段落数是1，所以长度与总长相同0xc4150000, 为5572

### 解析正文
从文本间隔开始，段落间距16(变量定义为sdo)

分析每行文本，循环111次，每行数据位置为i * 8 + sdo + 4， 每行的间距计算为该位置处4为int32值 + sdo  i = 0, 0x7c0300 = 892

文本长度位置i * 8 + sdo + 8处short值，2位short值 i= 0, 0x1800 = 24

文本内容占用length * 2, 48

文本内容地址，[offset, offset+ 48], offset = 908 ## [908, 956]

文本内容需要解密 方法

```python
key = 0x7C89
key_Advance = 0x2983
def __CryptLineData(data, key):
        """Decrypts the given line into a list of bytes"""
        copied = copy.copy(data)
        result = [None] * len(copied)

        for i in range(0, len(copied), 2):
            result[i] = copied[i] ^ (key % 256)
            result[i + 1] = copied[i + 1] ^ ((key >> 8) % 256)
            # Bit-shift and OR key, then cast to 16-bits (otherwise things break)
            key = (key << 3 | key >> 13) % 2**16

        return result
for i in len(0, len, 2):
    key = (key + self.__KEY_ADVANCE) % 2**16
    __CryptLineData(data, key)
```

具体规则待查
