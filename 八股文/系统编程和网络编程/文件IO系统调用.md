这些函数是 Linux/Unix 系统编程中**最基础的文件 I/O 系统调用**。下面我为你详细讲解每个函数的用法，包含头文件、参数、返回值和实用代码示例。

---

## 📁 1. `open()` - 打开或创建文件

### 📌 头文件

```
#include <fcntl.h>
#include <sys/types.h>
#include <sys/stat.h>
```

### 📌 函数原型


```
int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);
```

### 📌 参数说明

| 参数         | 说明                              |
| ---------- | ------------------------------- |
| `pathname` | 文件路径名（绝对路径或相对路径）                |
| `flags`    | 打开方式标志，常用组合：                    |
|            | `O_RDONLY` - 只读                 |
|            | `O_WRONLY` - 只写                 |
|            | `O_RDWR` - 读写                   |
|            | `O_CREAT` - 文件不存在则创建            |
|            | `O_TRUNC` - 文件存在则清空             |
|            | `O_APPEND` - 追加写入               |
|            | `O_EXCL` - 配合 O_CREAT，文件已存在则失败  |
| `mode`     | 文件权限（仅当使用 O_CREAT 时需要），如 `0644` |

### 📌 返回值

- **成功**：返回文件描述符（非负整数，通常从 3 开始）
- **失败**：返回 `-1`，并设置 `errno`

### 💡 代码示例



```
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <sys/stat.h>

int main() {
    int fd;
    
    // 1. 只读打开
    fd = open("test.txt", O_RDONLY);
    if (fd == -1) {
        perror("open read failed");
    } else {
        printf("Read fd: %d\n", fd);
        close(fd);
    }
    
    // 2. 写入打开，不存在则创建，权限 644
    fd = open("test.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd == -1) {
        perror("open write failed");
    } else {
        printf("Write fd: %d\n", fd);
        close(fd);
    }
    
    // 3. 追加写入
    fd = open("log.txt", O_WRONLY | O_CREAT | O_APPEND, 0666);
    if (fd != -1) {
        write(fd, "Append data\n", 12);
        close(fd);
    }
    
    return 0;
}
```

---

## 📖 2. `read()` - 从文件读取数据

### 📌 头文件


```
#include <unistd.h>
```

### 📌 函数原型



```
ssize_t read(int fd, void *buf, size_t count);
```

### 📌 参数说明


|参数|说明|
|---|---|
|`fd`|文件描述符|
|`buf`|读取数据存储的缓冲区地址|
|`count`|最多读取的字节数|

### 📌 返回值

- **成功**：返回实际读取的字节数（可能小于 `count`）
- **文件末尾**：返回 `0`
- **失败**：返回 `-1`，并设置 `errno`
- **被信号中断**：返回 `-1`，`errno = EINTR`

### 💡 代码示例


```
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>

int main() {
    int fd = open("test.txt", O_RDONLY);
    if (fd == -1) {
        perror("open failed");
        return 1;
    }
    
    char buf[1024];
    ssize_t bytes_read;
    
    // 循环读取直到文件末尾
    while ((bytes_read = read(fd, buf, sizeof(buf) - 1)) > 0) {
        buf[bytes_read] = '\0'; // 添加字符串结束符
        printf("Read %zd bytes: %s", bytes_read, buf);
    }
    
    if (bytes_read == -1) {
        perror("read failed");
    }
    
    close(fd);
    return 0;
}
```

---

## ✍️ 3. `write()` - 向文件写入数据

### 📌 头文件



```
#include <unistd.h>
```

### 📌 函数原型



```
ssize_t write(int fd, const void *buf, size_t count);
```

### 📌 参数说明

|参数|说明|
|---|---|
|`fd`|文件描述符|
|`buf`|要写入的数据缓冲区地址|
|`count`|要写入的字节数|

### 📌 返回值

- **成功**：返回实际写入的字节数（可能小于 `count`）
- **失败**：返回 `-1`，并设置 `errno`
- **被信号中断**：返回 `-1`，`errno = EINTR`

### 💡 代码示例


```
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>

int main() {
    int fd = open("output.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd == -1) {
        perror("open failed");
        return 1;
    }
    
    const char *msg = "Hello, Linux System Programming!\n";
    size_t len = strlen(msg);
    ssize_t bytes_written = write(fd, msg, len);
    
    if (bytes_written == -1) {
        perror("write failed");
    } else {
        printf("Wrote %zd bytes to file\n", bytes_written);
    }
    
    // 写入二进制数据
    int numbers[] = {1, 2, 3, 4, 5};
    write(fd, numbers, sizeof(numbers));
    
    close(fd);
    return 0;
}
```

---

## 📍 4. `lseek()` - 移动文件读写位置

### 📌 头文件

```
#include <unistd.h>
```

### 📌 函数原型

```
off_t lseek(int fd, off_t offset, int whence);
```

### 📌 参数说明

|参数|说明|
|---|---|
|`fd`|文件描述符|
|`offset`|偏移量（可以是正数或负数）|
|`whence`|偏移基准点：|
||`SEEK_SET` - 从文件开头开始|
||`SEEK_CUR` - 从当前位置开始|
||`SEEK_END` - 从文件末尾开始|

### 📌 返回值

- **成功**：返回新的文件偏移量（从文件开头算起的字节数）
- **失败**：返回 `-1`，并设置 `errno`

### 💡 代码示例

```
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>

int main() {
    int fd = open("test.txt", O_RDWR);
    if (fd == -1) {
        perror("open failed");
        return 0;
    }
    
    // 获取当前文件位置
    off_t pos = lseek(fd, 0, SEEK_CUR);
    printf("Current position: %ld\n", pos);
    
    // 移动到文件开头
    lseek(fd, 0, SEEK_SET);
    printf("Position after SEEK_SET: %ld\n", lseek(fd, 0, SEEK_CUR));
    
    // 向后移动 10 字节
    lseek(fd, 10, SEEK_CUR);
    printf("Position after +10: %ld\n", lseek(fd, 0, SEEK_CUR));
    
    // 移动到文件末尾
    off_t file_size = lseek(fd, 0, SEEK_END);
    printf("File size: %ld bytes\n", file_size);
    
    // 在文件末尾追加数据（稀疏文件）
    lseek(fd, 100, SEEK_END);
    write(fd, "EOF", 3);
    
    close(fd);
    return 0;
}
```
---

## 🔄 5. `dup()` - 复制文件描述符

### 📌 头文件

```
#include <unistd.h>
```

### 📌 函数原型

```
int dup(int oldfd);
```

### 📌 参数说明

|参数|说明|
|---|---|
|`oldfd`|要复制的文件描述符|

### 📌 返回值

- **成功**：返回新的文件描述符（最小的可用描述符）
- **失败**：返回 `-1`，并设置 `errno`

### 📌 特点

- 新旧文件描述符**共享文件偏移量和文件状态标志**
- 关闭任一描述符，文件不会关闭，直到所有描述符都关闭

### 💡 代码示例

```
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>

int main() {
    int fd1 = open("test.txt", O_RDONLY);
    if (fd1 == -1) {
        perror("open failed");
        return 1;
    }
    
    // 复制 fd1
    int fd2 = dup(fd1);
    if (fd2 == -1) {
        perror("dup failed");
        close(fd1);
        return 1;
    }
    
    printf("fd1 = %d, fd2 = %d\n", fd1, fd2);
    
    // 两个描述符共享偏移量
    char buf1[10], buf2[10];
    read(fd1, buf1, 5); // 读取前 5 字节
    read(fd2, buf2, 5); // 从第 6 字节开始读（因为共享偏移量）
    
    printf("fd1 read: %.*s\n", 5, buf1);
    printf("fd2 read: %.*s\n", 5, buf2);
    
    close(fd1);
    close(fd2);
    return 0;
}
```

---

## 🎯 6. `dup2()` - 复制文件描述符到指定位置

### 📌 头文件

```
#include <unistd.h>
```

### 📌 函数原型


```
int dup2(int oldfd, int newfd);
```

### 📌 参数说明

|参数|说明|
|---|---|
|`oldfd`|要复制的文件描述符|
|`newfd`|目标文件描述符（如果已打开则先关闭）|

### 📌 返回值

- **成功**：返回 `newfd`
- **失败**：返回 `-1`，并设置 `errno`

### 📌 特点

- 如果 `newfd` 已打开，会**自动关闭**后再复制
- 常用于**重定向标准输入 / 输出 / 错误**
- dup2 (oldfd, newfd) = 将对 newfd 的操作重定向到 oldfd

### 💡 代码示例 1：重定向标准输出到文件

```
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>

int main() {
    int fd = open("output.log", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd == -1) {
        perror("open failed");
        return 1;
    }
    
    // 1.保存标准输出
    int saved_stdout = dup(STDOUT_FILENO); // 该STDOUT_FILENO与
    
    
    // 重定向前：
    // 1 → 屏幕 
    // fd(3) → output.log
    
    // 2.将标准输出重定向到文件
    dup2(fd, STDOUT_FILENO); // STDOUT_FILENO = 1
    close(fd);               // 关闭原描述符，因为后续不需要使用该fd了
    
    // 重定向后：
    // 1 → output.log
    // fd(3) → output.log
    
    // 现在 printf 会输出到文件
    printf("This goes to output.log\n");
    fprintf(stdout, "Also to output.log\n");
    
    
    // 3.恢复标准输出
    dup2(saved_stdout, STDOUT_FILENO); // 恢复
    close(saved_stdout);     // 关闭用于保存的saved_stdout，因为后续不需要使用该fd了
    
    
    printf("This goes to terminal again\n");
    
    return 0;
}
```

### 💡 代码示例 2：实现管道重定向（类似 shell 的 `|`）


```
#include <unistd.h>
#include <stdio.h>
#include <fcntl.h>

int main() {
    int pipefd[2];
    pipe(pipefd); // 创建管道
    
    pid_t pid = fork();
    
    if (pid == 0) {
        // 子进程：读取管道
        close(pipefd[1]); // 关闭写端
        dup2(pipefd[0], STDIN_FILENO); // 将管道读端重定向到 stdin
        close(pipefd[0]);
        
        // 运行 grep 命令，从标准输入中筛选出包含 hello 的行(已重定向为管道读端)
        // grep 要找的内容 从哪里找
        execlp("grep", "grep", "hello", NULL);
    } else {
        // 父进程：写入管道
        close(pipefd[0]); // 关闭读端
        dup2(pipefd[1], STDOUT_FILENO); // 将 stdout 重定向到管道写端
        close(pipefd[1]);
        
        // 打印的内容会通过管道传给子进程
        printf("hello world\n");
        printf("goodbye world\n");
        printf("hello Linux\n");
        
        close(STDOUT_FILENO); // ▲关闭所有写端，读端收到EOF，用于通知子进程结束
        wait(NULL);
    }
    
    return 0;
}
```

---

## 📊 函数对比总结

|函数|作用|关键特点|
|---|---|---|
|`open()`|打开 / 创建文件|返回最小可用描述符|
|`read()`|读取数据|可能返回少于请求的字节数|
|`write()`|写入数据|可能返回少于请求的字节数|
|`lseek()`|移动文件指针|支持随机访问|
|`dup()`|复制描述符|返回最小可用描述符|
|`dup2()`|复制到指定描述符|自动关闭目标描述符|

---

## 🎓 实战：实现简单的文件复制程序

```
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>

#define BUFFER_SIZE 4096

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "Usage: %s <source> <destination>\n", argv[0]);
        return 1;
    }
    
    int src_fd = open(argv[1], O_RDONLY);
    if (src_fd == -1) {
        perror("open source");
        return 1;
    }
    
    int dst_fd = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (dst_fd == -1) {
        perror("open destination");
        close(src_fd);
        return 1;
    }
    
    char buffer[BUFFER_SIZE];
    ssize_t bytes_read, bytes_written;
    
    while ((bytes_read = read(src_fd, buffer, BUFFER_SIZE)) > 0) {
        char *ptr = buffer;
        while (bytes_read > 0) {
            bytes_written = write(dst_fd, ptr, bytes_read);
            if (bytes_written == -1) {
                perror("write");
                close(src_fd);
                close(dst_fd);
                return 1;
            }
            ptr += bytes_written;
            bytes_read -= bytes_written;
        }
    }
    
    if (bytes_read == -1) {
        perror("read");
        close(src_fd);
        close(dst_fd);
        return 1;
    }
    
    printf("File copied successfully!\n");
    close(src_fd);
    close(dst_fd);
    return 0;
}
```

---

## ⚠️ 重要注意事项

1. **错误处理**：所有系统调用都可能失败，必须检查返回值
2. **资源泄漏**：打开的文件描述符必须在使用后关闭
3. **缓冲区大小**：`read`/`write` 可能返回少于请求的字节数，需循环处理
4. **原子操作**：管道写入小于 `PIPE_BUF`（通常 4096 字节）时是原子的
5. **文件描述符限制**：每个进程有最大文件描述符数限制（`ulimit -n`）