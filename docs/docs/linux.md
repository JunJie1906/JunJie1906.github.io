# Linux常用命令

### 查看指标命令

1. CPU 相关指标：
   - `top`：实时查看 CPU 使用率、进程信息等。
   - `mpstat`：显示 CPU 的详细统计信息。
   - `lscpu`：显示 CPU 的详细信息，如核心数、线程数等。
2. 内存相关指标：
   - `free`：查看系统的内存使用情况。
   - `top`：显示内存占用情况及进程信息。
   - `vmstat`：提供虚拟内存统计信息，包括内存使用、交换分区等。
3. 磁盘相关指标：
   - `df`：显示文件系统的磁盘空间使用情况。
   - `du`：计算目录或文件的磁盘使用情况。
   - `iostat`：显示磁盘 I/O 统计信息，包括读写速度、响应时间等。
4. 网络相关指标：
   - `ifconfig`：显示网络接口的配置信息。
   - `netstat`：显示网络连接、路由表等信息。
   - `ping`：测试网络连接和延迟。
   - `nload`：实时监测网络流量。
5. 进程相关指标：
   - `ps`：显示当前运行的进程。
   - `top`：查看系统的进程列表、CPU 使用率等。
   - `htop`：交互式地查看进程列表和系统状态。

### 查看文件命令

在 Linux 中，有几个常用的命令可以用于查看文件和查询文件内容：

1. ```
   ls
   ```

   ：用于列出当前目录中的文件和文件夹。

2. ```
   cat
   ```

   ：用于显示文件的内容。

3. ```
   more
   ```

   ```
   less
   ```

   ：用于逐页查看文件内容。这些命令允许你在文件内容超过一个屏幕时进行分页查看。

   ````
   more file_name
   less file_name
   ```
   ````

4. ```
   head
   ```

   ：用于显示文件的前几行，默认显示前 10 行。

   ````
   head file_name
   head -n 5 file_name  # 显示前 5 行
   ```
   ````

5. ```
   tail
   ```

   ：用于显示文件的后几行，默认显示最后 10 行。

   ```
   tail
   ```

   命令通常用于查看日志文件的最新内容。

   ````
   tail file_name
   tail -n 5 file_name  # 显示最后 5 行
   ```
   ````

6. ```
   grep
   ```

   ：用于在文件中搜索指定的字符串或模式，并显示匹配的行。

   ````
   grep "pattern" file_name
   ```
   ````

7. ```
   find
   ```

   ：用于在指定目录下查找文件。

   ````
   find /path/to/directory -name "filename"
   ```
   ````

8. ```
   file
   ```

   ：用于确定文件的类型。

   ```
   file file_name
   ```

