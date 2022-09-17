 如果我们想在Markdown中展示文件夹的树状格式，可以使用Windows自带的tree命令输出文件树，然而自带的tree命令不支持排除指定目录等高级功能。这时候就需要安装一个加强版的tree命令 - `Tree for Windows`，它提供了排除目录、排序规则等丰富的参数。
 
## 先来看看Windows自带的tree命令用法

```bash
# 查看使用说明
PS C:\Users\Administrator> tree /?

TREE [drive:][path] [/F] [/A]
   /F   显示每个文件夹中文件的名称。
   /A   使用 ASCII 字符，而不使用扩展字符。
```
自带的tree命令十分简单，显然不能满足我们排除目录的需求，此时我们就需要安装强化版的tree命令。

## 使用强化版的Tree命令 - Tree for Windows

### 下载安装

1. 进入[Tree for Windows](http://gnuwin32.sourceforge.net/packages/tree.htm)

2. 选择 Binaries ZIP 下载

3. 解压，找到bin目录下的可执行文件 `tree.exe`，复制到任意目录（我常常将它们放在`C:\bin`）

### 强化版Tree命令的使用说明

```bash
# 在tree.exe所在目录执行查看使用说明的命令
> ./tree.exe --help
usage: tree [-adfghilnpqrstuvxACDFNS] [-H baseHREF] [-T title ] [-L level [-R]]
        [-P pattern] [-I pattern] [-o filename] [--version] [--help] [--inodes]
        [--device] [--noreport] [--nolinks] [--dirsfirst] [--charset charset]
        [--filelimit #] [<directory list>]
  -a            All files are listed.
  -d            List directories only.
  -l            Follow symbolic links like directories.
  -f            Print the full path prefix for each file.
  -i            Don\'t print indentation lines.
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

### 使用例子

查看目录`E:\testdir`下的文件树，排除目录`cachedir`，输出排序规则以目录优先。
```bash
> ./tree.exe E:\testdir\ -I cachedir  --dirsfirst
E:\testdir\
|-- app
|   |-- code
|   |   `-- test.txt
|   |-- emptydir
|   `-- index.txt
|-- bar
|   `-- 233.txt
|-- 1.txt
|-- 2.txt
`-- a.txt
```