 一开始我是想在Markdown中展示文件夹的树状格式，发现可以用tree命令输出文件树。Windows支持tree命令，然而自带的tree命令不支持排除指定目录等高级的功能，这时候就需要安装一个加强版的`Tree for Windows`，它提供了丰富的参数。
 
## 先来看看Windows自带的tree命令用法

```shell
# 查看使用说明
PS C:\Users\Administrator> tree /?

TREE [drive:][path] [/F] [/A]
   /F   显示每个文件夹中文件的名称。
   /A   使用 ASCII 字符，而不使用扩展字符。
```   

## 安装强化的`Tree`命令 - Tree for Windows

只需要下载一个可执行文件，就可以使用强化的tree命令了。

1. 进入[Tree for Windows](http://gnuwin32.sourceforge.net/packages/tree.htm)

2. 选择 Binaries ZIP 下载

3. 解压，找到bin目录下的可执行文件 `tree.exe`，复制到任意目录（我常常将它们放在`C:\bin`）

## 强化版Tree命令的用法

在`tree.exe`所在的目录执行命令
```shell
# 查看使用说明
PS C:\bin> ./tree.exe --help
usage: tree [-adfghilnpqrstuvxACDFNS] [-H baseHREF] [-T title ] [-L level [-R]]
        [-P pattern] [-I pattern] [-o filename] [--version] [--help] [--inodes]
        [--device] [--noreport] [--nolinks] [--dirsfirst] [--charset charset]
        [--filelimit #] [<directory list>]
  -a            All files are listed.
  -d            List directories only.
  -l            Follow symbolic links like directories.
  -f            Print the full path prefix for each file.
  -i            Don't print indentation lines.
  -q            Print non-printable characters as '?'.
  -N            Print non-printable characters as is.
  -p            Print the protections for each file.
  -u            Displays file owner or UID number.
  -g            Displays file group owner or GID number.
  -s            Print the size in bytes of each file.
  -h            Print the size in a more human readable way.
  -D            Print the date of last modification.
  -F            Appends '/', '=', '*', or '|' as per ls -F.
  -v            Sort files alphanumerically by version.
  -r            Sort files in reverse alphanumeric order.
  -t            Sort files by last modification time.
  -x            Stay on current filesystem only.
  -L level      Descend only level directories deep.
  -A            Print ANSI lines graphic indentation lines.
  -S            Print with ASCII graphics indentation lines.
  -n            Turn colorization off always (-C overrides).
  -C            Turn colorization on always.
  -P pattern    List only those files that match the pattern given.
  -I pattern    Do not list files that match the given pattern.
  -H baseHREF   Prints out HTML format with baseHREF as top directory.
  -T string     Replace the default HTML title and H1 header with string.
  -R            Rerun tree when max dir level reached.
  -o file       Output to file instead of stdout.
  --inodes      Print inode number of each file.
  --device      Print device ID number to which each file belongs.
  --noreport    Turn off file/directory count at end of tree listing.
  --nolinks     Turn off hyperlinks in HTML output.
  --dirsfirst   List directories before files.
  --charset X   Use charset X for HTML and indentation line output.
  --filelimit # Do not descend dirs with more than # files in them.
```

## 使用例子

```bash
# 查看E:\testdir目录下的文件树，排除目录cachedir，输出排序以目录优先
> ./tree.exe E:\testdir\ -I cachedir  --dirsfirst
E:\testdir\
|-- app
|   |-- code
|   |   `-- test.txt
|   |-- empty
|   `-- index.txt
|-- bar
|   `-- 233.txt
|-- 1.txt
|-- 2.txt
`-- a.txt
```