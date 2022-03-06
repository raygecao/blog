---
weight: 20
title: "Linux文本处理工具"
subtitle: ""
date: 2022-03-05T16:39:25+08:00
lastmod: 2022-03-05T16:39:25+08:00
draft: false
author: "raygecao"
authorLink: ""
description: ""
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["linux"]
categories: ["整理与总结"]


toc:
  auto: false
lightgallery: true
license: ""
---

grep, sed, awk被称为linux文本处理三剑客，分别侧重于**文本搜索**，**流式编辑**和**格式化文本**。

<!--more-->

## grep

- **文本搜索工具**

- Schema：`grep [option...] [patterns] [file...]`

### Options

| 短选项       | 长选项                                         | 作用                                                         |
| ------------ | ---------------------------------------------- | ------------------------------------------------------------ |
| `-e pattern` | `--regexp=pattern`                             | 指定搜索pattern，可以不显式指定                              |
| `-f file`    | `--file=file`                                  | 从file中获取pattern进行搜索                                  |
| `-i/-y`      | `ignore-case`                                  | 匹配忽略大小写                                               |
| `-v`         | `--invert-match`                               | 匹配反选                                                     |
| `-w`         | `--word-regexp`                                | 指定单词完整匹配，其边界为行首或者除字母、数字、下划线以外的字符 |
| `-x`         | `--line-regexp`                                | 行完整匹配                                                   |
| `-c`         | `--count`                                      | 不打印匹配行，只打印匹配行数                                 |
| `-L` `-l`    | `--files-without-match` `--files-with-matches` | 不打印匹配行，只打印完全不匹配/存在匹配的文件名              |
| `-m num`     | `--max-count=num`                              | 指定最大匹配次数                                             |
| `-o`         | `--only-matching`                              | 只打印完整的匹配内容，不打印匹配行                           |
| `-q`         | `--quiet/silent`                               | 不打印任何内容，只以返回码来描述是否存在匹配                 |
| `-h` `-H`    | `--no-filename` `--with-filename`              | 不打印/打印匹配的文件名前缀，默认为打印(`-H`)                |
| `-n`         | `--line-number`                                | 打印匹配的行号前缀                                           |
| `-A num`     | `--after-context=num`                          | 打印匹配行，及其以后的num行                                  |
| `-B num`     | `--before-context=num`                         | 打印匹配行，及其以前的num行                                  |
| `-C num`     | `-context=num/-num`                            | 打印匹配行，及其前后的各num行                                |
| `-R`         | `--dereference-recursive`                      | 递归的匹配目录下所有文件，遇到符号链接正常递归               |

### Tips

- grep不支持换行符的匹配

## sed

- **流编辑器**，用于逐行地对输入流进行基本的文本转换

- schema：`sed SCRIPT INPUTFILE`
  - `INPUTFILE`未指定或者指定为`-`时，表示从stdin读取内容
  - 默认输出为stdout，可以通过`w` cmd/重定向符保存到文件，可以指定`-i`在文件内原地修改
- 多个cmd可以通过`;`或换行来指定，或者`-e`指定。（`a,c,i`命令中不应使用`;`，会将其当做plain text）
- data buffer
  - **模式空间**（Pattern space）：数据处理空间，sed读取每行到模式空间，进行地址匹配并执行命令
  - **保持空间**（Hold space）：辅助暂存空间，在复杂处理过程中，作为数据的暂存区域

### Options

| 短选项  | 长选项                | 作用                                                         |
| ------- | --------------------- | ------------------------------------------------------------ |
| `-n`    | `--quiet/--silent`    | 禁掉打印模式空间的内容，一般结合`p`命令打印处理过的行        |
| `-f`    | `--file`              | 指定sed脚本文件路径，命令形式指定则对应选项`-e`，一般无需显式指定`-e` |
| `-i`    | `--in-place=[suffix]` | 原地修改文件，如指定suffix会拼接到原文件名作为原文件内容的backup |
| `-E/-r` | `--regexp-entended`   | 使用扩展正则表达式                                           |
| `-s`    | `--seperate`          | 指定操作多个独立的文件                                       |
|         | `--posix`             | 严格限定posix，禁用GNU扩展，便于简化可移植脚本的编写         |

### Commands

基本形式：`[addr]X[options]`，其中`addr`为地址匹配模式，`X`为单字符命令，`options`指一些命令需要的额外选项。

- **s命令**：替换字符串
  - `s/regexp/replacement/flags`
  - 根据regexp去模式空间找匹配，匹配成功后用replacement去替换掉regexp
  - 特殊符号需插入`\`转义
  - 常用flag：
    - `g`：所有匹配均替换
    - `number`：只替换第`number`个匹配
    - `p`：如果发生匹配，打印替换后的模式空间
    - `i/I`：匹配忽略字母大小写
    - `m/M`：支持多行匹配
- 其他常见命令

| cmd                         | 作用                                                         |
| --------------------------- | ------------------------------------------------------------ |
| `d`                         | 删除模式空间内容，并开始下一轮匹配                               |
| `p`                         | 匹配成功时，打印模式空间，一般配合`-n`选项使用               |
| `n`                         | 读取下一行替换当前模式空间的行，执行下一条处理命令而非第一条命令。一般用于周期性替换，如偶数行替换 `seq 6 \| sed 'n;s/./x/'`。`N`命令在此基础上支持换行处理 |
| `{}`                        | 封装一组命令用以执行                                         |
| `a text`                    | 在一行后添加text                                             |
| `i text`                    | 在一行前插入text                                             |
| `c text`                    | 使用text替换行                                               |
| `y/source-chars/dest-chars` | 执行字符的替换，如使用`0-9`替换`a-j`：`echo hello \| sed 'y/abcdefghij/0123456789/'` |
| `r filename`                | 读取文件                                                     |
| `w filename`                | 将模式空间内容写入到文件                                         |
| `b label`                   | 类似于`if-else`的分支命令，goto label，如跳过第一行的替换：`printf '%s\n' a1 a2 a3 \| sed -E '/1/bx ; s/a/z/ ; :x'` |

- 与保持空间相关的命令

| cmd  | 作用                                           |
| ---- | ---------------------------------------------- |
| `g`  | 使用保持空间的内容替换模式空间的内容                 |
| `G`  | 添加一个新行，并将保持空间的内容追加到模式空间 |
| `h`  | 使用模式空间的内容替换保持空间的内容                 |
| `H`  | 添加一个新行，并将模式空间的内容追加到保持空间 |
| `x`  | 交换保持空间和模式空间的内容                   |

### 地址匹配

- 行号匹配
  - **number**：第几行
  - **$**: 最后一行
  - **first~step**：从first开始，每隔step匹配
- 正则匹配
- 范围匹配
  - **addr1,+N**：匹配`[addr1, addr1+N]`行
  - **addr1,~N**：匹配`[addr1, addr1+addr1%N]`行

```shell
# examples
sed '4,17s/hello/world/' input.txt # 第4-17行将所有hello替换成world
sed '/apple/s/hello/world/' input.txt # 所有包含apple的行将hello替换为world
sed '2!s/hello/world/' input.txt > output.txt # 除第二行以外将所有hello替换为world
```

### Tips

- `-i`可以指定可选参数，所以其后不应有其他的短选项
  - `sed -Ei ...` => `sed -E -i ...`
  - `sed -iE ...` => `sed --in-place=E ...`
- `label`用作循环处理文本，一般会配合`n/N`使用，如多行合并的示例：`seq 6 | sed ':x; N;s/\n//; bx;'`

- `D,G,H,N,P`支持多行处理，每个命令的作用与其小写命令相同

## awk

**文本格式化工具**，多用于格式化文本，生成报表。

### 基本用法

**schema**

```shell
awk [options] 'Pattern{Action}' file
```

**普通格式化**

```shell
# $0表示完整行，$1表示第一列，$NF表示最后一列，$(NF-1)表示倒数第二列
df | awk '{print $1, $2}'
```

**设置分隔符**

```shell
# 输入字段分隔符，可以使用-F: 或者-v FS=:的形式设置
cat /etc/passwd | awk -F: '{print $1}

# 输出字段分隔符，可以使用-v OFS=@的形式设置
cat /etc/passwd | awk -v FS=: -v OFS=@ '{print $1, $3}'
```

**内置变量**

| 变量名       | 作用                     | 默认值 |
| ------------ | ------------------------ | ------ |
| **FS**       | 输入字段分隔符           | 空格   |
| **OFS**      | 输出字段分隔符           | 空格   |
| **RS**       | 输入换行符（记录分隔符） | \n     |
| **ORS**      | 输出换行符               | \n     |
| **NF**       | 字段数目                 |        |
| **NR**       | 行号                     |        |
| **FNR**      | 文件序号                 |        |
| **FILENAME** | 当前文件名               |        |
| **ARGC**     | 命令行参数的个数         |        |
| **ARGV**     | 命令行参数组成的数组     |        |

```shell
# 打印行号，变量在action里不应加$，加$表示获取对应列的内容
df | awk '{print NR, $2}'
```

**自定义变量**

```shell
# 使用-v设置
awk -v a="b" 'BEGIN {print a}'
# 在action中设置，并用分号分隔
awk 'BEGIN {a="b"; print a}'
```

### 特殊模式

- **BEGIN**表示在处理文本前执行action，可用于打印表头
- **END**表示在处理文本后执行action，可用于打印表尾

```shell
df | awk 'BEGIN{print "开始时执行"} {print $1} END{print "结束时执行"}'
```

**条件匹配**

```shell
# 打印第2行，第一列的内容
df | awk 'NR == 2 {print $1}'

# 如果第3行的值大于0，打印完整行
df | awk '$3>0 {print}'
```

**正则相关**

```shell
# 打印/dev开头的行
# 正则模式采用awk '/regexp1/{action}'的形式
df | awk '/^\/dev/{print}' 

# 打印最后一列以/dev开头的行
# 正则匹配采用awk 'var~/regexp1/{action}'的形式
df | awk '$NF~/^\/dev/{print}'

# 打印最后一列从/d开头到/p开头之间的行
# 范围模式采用 awk '/regexp1/, /regexp2/ {action}'形式
df | awk '$NF~/^\/d/,$NF~/^\/p/{print}'
```

**分支动作**

```shell
# 判断分支
df | awk '{ if($2>0){print "大于0"}else{print "小于0"} }'

# 循环分支
df | awk '{ for(i=1; i<3; i++){print $i}}'
df | awk '{ i=0; while(i<4){print $0; i++}}' # 将每行打印4遍
```

### Tips

- `{}`意味着代码块，可以通过`;`将语句划归到同一代码块中
- print动作如果使用`,`分隔两列，则输出会以输出字段分隔符将两列连接起来，如果以空格分隔，则输出会将两列直接拼接打印
- awk支持三元运算符

## 参考文档

- https://www.gnu.org/software/grep/manual/grep.html
- https://www.gnu.org/software/sed/manual/sed.html
- https://www.gnu.org/software/gawk/manual/gawk.html
