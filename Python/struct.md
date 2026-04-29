# struct 用法
## struct.pack(fmt,v1,v2,.....)
在转化过程中，主要用到了一个格式化字符串(format strings)，用来规定转化的方法和格式。
## struct.unpack(fmt,string)
解包
## fmt 定义
| Format | c Type | Python |Note|
| :-----| ----: | :----: |:----: |
| x | pad type | no value | |
| c | char | string of length 1 | |
| b | signedchar | integer | |
| B | unsignedchar | integer | |
| ? | _Bool | bool |(1)|
| h | short | integer | |
| H | unsignedshort | integer | |
| i | int | integer | |
| I | UnsignedInt | integer or long | |
| l | long | integer |(2)|
| L | unsignedLong | integer |(2)|
| q | longlong | long | |
| Q | unsignedlonglong | long | |
| f | float | float | |
| d | double | float | |
| s | char[] | string | |
| p | char[] | string | |
| P | void* | long | |