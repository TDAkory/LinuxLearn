# [dd](https://phoenixnap.com/kb/linux-dd-command)

`dd` 是Linux/Unix下常用的磁盘**顺序读写性能测试工具**（注：`dd` 更适合测顺序读写，随机读写建议用 `fio`），以下是常见的测试命令及说明：

### 一、磁盘写速度测试（测试向磁盘写数据的速度）

#### 1. 绕过内存缓存的写测试（真实磁盘写速度）

```bash
# 生成1GB的0数据，写入到testfile文件，绕过页缓存（直接写磁盘）
dd if=/dev/zero of=./testfile bs=1G count=1 oflag=direct
```

- 参数说明：
  - `if=/dev/zero`：输入源为“无限0数据”（用于生成测试数据）；
  - `of=./testfile`：输出到当前目录的`testfile`文件；
  - `bs=1G`：每次读写的**块大小**为1GB（可根据需求调整，如`bs=4K`测小文件写）；
  - `count=1`：读写的**块数量**为1（总数据量=bs×count=1GB）；
  - `oflag=direct`：启用**直接IO**，绕过系统页缓存（避免内存缓存“伪装”成磁盘速度）。


#### 2. 强制同步到磁盘的写测试（确保数据落盘）

若要确保数据真正写入磁盘（而非暂存到缓存），可加 `conv=fdatasync`：

```bash
dd if=/dev/zero of=./testfile bs=1G count=1 conv=fdatasync
```

### 二、磁盘读速度测试（测试从磁盘读数据的速度）

需先完成**写测试生成`testfile`**，再执行读测试：

```bash
# 读取testfile文件，输出到“黑洞”/dev/null，绕过页缓存（真实磁盘读速度）
dd if=./testfile of=/dev/null bs=1G count=1 iflag=direct
```

- 参数说明：
  - `if=./testfile`：输入源为刚才生成的测试文件；
  - `of=/dev/null`：输出到“数据黑洞”（只读不保存，仅测读速度）；
  - `iflag=direct`：启用直接IO，绕过系统页缓存（避免读缓存影响结果）。

### 三、小文件/块的读写测试（模拟细碎IO场景）

若要测试小文件/小块的读写性能（比如4KB块），可调整`bs`参数：

```bash
# 写测试（4KB块，总大小1GB）
dd if=/dev/zero of=./testfile bs=4K count=262144 oflag=direct

# 读测试（对应4KB块）
dd if=./testfile of=/dev/null bs=4K count=262144 iflag=direct
```

### 四、测试结果说明

执行命令后，`dd` 会输出类似结果：

```shell
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 8.23456 s, 130 MB/s
```

其中 `130 MB/s` 就是本次测试的**读写速度**。

### 注意事项

1. **测试前确保磁盘有足够空间**：比如测10GB数据，目标磁盘需至少有10GB空闲；
2. **多次测试取平均值**：单次测试可能受系统负载影响，建议测3-5次取平均；
3. **测试后清理测试文件**：`rm -f ./testfile` （避免占用磁盘空间）；
4. **`dd` 不适合测随机读写**：随机读写性能建议用专业工具 `fio`。