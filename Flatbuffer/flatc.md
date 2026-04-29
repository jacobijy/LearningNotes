# flatc flatbuffer可执行文件
可打包json到二进制文件，也可根据fbs文件生成各种语言的模板
## 打包json
```sh
    flatc -b foo.fbs foo.json
```
## 生成json
```
    flatc --json --raw-binary theater.fbs -- theater.bin
```
## 生成模板代码文件
ts为例
```
    flatc -T foo.fbs
```