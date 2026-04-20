下面我按照头文件、函数原型、参数说明、返回值、特性、代码示例的格式，详细整理和补充 IO 多路复用的 API。

## 📚 一、fd_set 操作函数

### 📌 fd_set 位图操作函数

#### 头文件

```
#include <sys/select.h>
```

#### 函数原型

```
void FD_ZERO(fd_set *set);           // 清除所有位
void FD_SET(int fd, fd_set *set);    // 添加某位
void FD_CLR(int fd, fd_set *set);    // 清除某位
int FD_ISSET(int fd, fd_set *set);   // 检查某位是否存在
```

#### 参数说明

|函数|参数|说明|
|---|---|---|
|FD_ZERO|set|fd_set 结构体指针|
|FD_SET|fd、set|文件描述符|
|FD_CLR|fd、set|文件描述符|
|FD_ISSET|fd、set|文件描述符|

#### 返回值

|函数|返回值|说明|
|---|---|---|
|FD_ZERO|无|-|
|FD_SET|无|-|
|FD_CLR|无|-|
|FD_ISSET|1|位存在|
||0|位不存在|

#### 特性

用于操作文件描述符集合

fd_set 本质是一个位图（bitmap）

最大支持 1024 个文件描述符（FD_SETSIZE）

#### 代码示例

```
#include <sys/select.h>
#include <stdio.h>

int main() {
    fd_set readfds;
    int fd1 = 3, fd2 = 5;
    
    // 初始化：清除所有位
    FD_ZERO(&readfds);
    
    // 添加文件描述符
    FD_SET(fd1, &readfds);
    FD_SET(fd2, &readfds);
    
    // 检查文件描述符是否存在
    if (FD_ISSET(fd1, &readfds)) {
        printf("fd %d is in the set\n", fd1);
    }
    
    if (FD_ISSET(fd2, &readfds)) {
        printf("fd %d is in the set\n", fd2);
    }
    
    // 移除文件描述符
    FD_CLR(fd1, &readfds);
    
    if (!FD_ISSET(fd1, &readfds)) {
        printf("fd %d is removed from the set\n", fd1);
    }
    
    return 0;
}
```

## 📚 二、select 函数

### 📌 select - 多路复用（跨平台）

#### 头文件

```
#include <sys/select.h>
#include <sys/time.h>
#include <unistd.h>
```

#### 函数原型

```
int select(int nfds, fd_set *readfds, fd_set *writefds, 
           fd_set *exceptfds, struct timeval *timeout);
```

#### 参数说明

| 参数        | 说明                                                                                  |
| --------- | ----------------------------------------------------------------------------------- |
| nfds      | 需要监视的**最大文件描述符值 + 1**                                                               |
| readfds   | 传入传出参数，可读事件集合                                                                       |
| writefds  | 传入传出参数，可写事件集合                                                                       |
| exceptfds | 传入传出参数，异常事件集合                                                                       |
| timeout   | 超时时间：<br><br>NULL - 永久阻塞<br><br>tv_sec=0, tv_usec=0 - 立即返回（非阻塞）<br><br>其他值 - 等待指定时间 |

timeval 结构
```
struct timeval {
    long tv_sec;   // 秒
    long tv_usec;  // 微秒
};
```

#### 返回值

|返回值|说明|
|---|---|
|> 0|成功：就绪的文件描述符数量|
|0|超时|
|-1|失败，设置 errno|

#### 特性

跨平台：支持所有操作系统

文件描述符限制：最大 1024（FD_SETSIZE）

效率问题：每次调用都需要将 fd_set 从用户态拷贝到内核态

线性扫描：内核需要遍历所有文件描述符

水平触发：只要缓冲区有数据，就会持续通知

#### 代码示例 1：基本使用

```
#include <sys/select.h>
#include <sys/time.h>
#include <stdio.h>
#include <unistd.h>

int main() {
    fd_set readfds;
    struct timeval timeout;
    int maxfd = 0;
    int ret;
    
    // 初始化
    FD_ZERO(&readfds);
    FD_SET(STDIN_FILENO, &readfds);  // 监听标准输入
    maxfd = STDIN_FILENO;
    
    // 设置超时时间：5秒
    timeout.tv_sec = 5;
    timeout.tv_usec = 0;
    
    printf("Waiting for input (timeout: 5s)...\n");
    
    ret = select(maxfd + 1, &readfds, NULL, NULL, &timeout);
    
    if (ret == -1) {
        perror("select error");
    } else if (ret == 0) {
        printf("Timeout!\n");
    } else {
        if (FD_ISSET(STDIN_FILENO, &readfds)) {
            char buf[1024];
            ssize_t n = read(STDIN_FILENO, buf, sizeof(buf));
            if (n > 0) {
                buf[n] = '\0';
                printf("Read: %s", buf);
            }
        }
    }
    
    return 0;
}
```

#### 代码示例 2：TCP 服务器（多路转接）

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/select.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <ctype.h>            //toupper()

#define PORT 8888
#define MAX_CLIENTS 1024
#define BUFFER_SIZE 1024

int main() {
    int listenfd, connfd, maxfd, sockfd;
    int client[MAX_CLIENTS];
    fd_set readfds, allfds;
    struct sockaddr_in serv_addr, cli_addr;
    socklen_t cli_len;
    char buffer[BUFFER_SIZE];
    ssize_t n;
    char cli_ip[INET_ADDRSTRLEN];
    
    // 初始化客户端数组
    for (int i = 0; i < MAX_CLIENTS; i++) {
        client[i] = -1;
    }
    
    // 创建监听套接字
    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd == -1) {
        perror("socket error");
        exit(1);
    }
    
    // 允许端口重用
    int opt = 1;
    setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    
    // 绑定地址
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    
    if (bind(listenfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr)) == -1) {
        perror("bind error");
        close(listenfd);
        exit(1);
    }
    
    // 监听
    if (listen(listenfd, 128) == -1) {
        perror("listen error");
        close(listenfd);
        exit(1);
    }
    
    printf("Select server started on port %d\n", PORT);
    
    // 初始化 fd_set
    FD_ZERO(&allfds);
    FD_SET(listenfd, &allfds);
    maxfd = listenfd;
    
    // 主循环
    while (1) {
        readfds = allfds;  // 每次都要重新设置
        
        // 等待事件
        int ret = select(maxfd + 1, &readfds, NULL, NULL, NULL);
        
        if (ret == -1) {
            if (errno == EINTR) {
                continue;  // 被信号中断，重试
            }
            perror("select error");
            break;
        }
        
        // 检查监听套接字（新连接）
        if (FD_ISSET(listenfd, &readfds)) {
            cli_len = sizeof(cli_addr);
            connfd = accept(listenfd, (struct sockaddr*)&cli_addr, &cli_len);
            
            if (connfd == -1) {
                perror("accept error");
                continue;
            }
            
            // 打印客户端信息
            inet_ntop(AF_INET, &cli_addr.sin_addr, cli_ip, sizeof(cli_ip));
            printf("New connection from %s:%d\n", 
                   cli_ip, ntohs(cli_addr.sin_port));
            
            // 找到空闲位置
            int i;
            for (i = 0; i < MAX_CLIENTS; i++) {
                if (client[i] == -1) {
                    client[i] = connfd;
                    break;
                }
            }
            
            if (i == MAX_CLIENTS) {
                printf("Too many clients, closing connection\n");
                close(connfd);
            } else {
                // 添加到监听集合
                FD_SET(connfd, &allfds);
                
                // 更新 maxfd
                if (connfd > maxfd) {
                    maxfd = connfd;
                }
            }
            
            if (--ret <= 0) {
                continue;  // 没有其他事件
            }
        }
        
        // 检查客户端套接字（数据到达）
        for (int i = 0; i < MAX_CLIENTS; i++) {
            if ((sockfd = client[i]) == -1) {
                continue;
            }
            
            if (FD_ISSET(sockfd, &readfds)) {
                n = read(sockfd, buffer, BUFFER_SIZE - 1);
                
                if (n == -1) {
                    perror("read error");
                    close(sockfd);
                    FD_CLR(sockfd, &allfds);
                    client[i] = -1;
                } else if (n == 0) {
                    // 客户端关闭连接
                    printf("Client %d closed connection\n", sockfd);
                    close(sockfd);
                    FD_CLR(sockfd, &allfds);
                    client[i] = -1;
                } else {
                    buffer[n] = '\0';
                    printf("Received from client %d: %s", sockfd, buffer);
                    
                    // 转大写
                    for (int j = 0; j < n; j++) {
                        buffer[j] = toupper(buffer[j]);
                    }
                    
                    // 发送回客户端
                    write(sockfd, buffer, n);
                }
                
                if (--ret <= 0) {
                    break;  // 没有其他事件
                }
            }
        }
    }
    
    close(listenfd);
    return 0;
}
```

## 📚 三、poll 函数

### 📌 poll - 多路复用（改进版）

#### 头文件

```
#include <poll.h>
```

#### 函数原型

```
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

pollfd 结构
```
struct pollfd {
    int fd;         // 待监听的文件描述符
    short events;   // 监听的事件
    short revents;  // 返回的事件
};
```

#### 事件类型

|事件|说明|
|---|---|
|POLLIN|可读|
|POLLOUT|可写|
|POLLERR|错误|
|POLLHUP|挂起（对端关闭）|
|POLLNVAL|无效的 fd|

#### 参数说明

|参数|说明|
|---|---|
|fds|pollfd 结构体数组指针|
|nfds|数组中有效元素的数量|
|timeout|超时时间（毫秒）：<br><br>-1 - 永久阻塞<br><br>0 - 立即返回（非阻塞）<br><br>>0 - 等待指定毫秒数|

#### 返回值

|返回值|说明|
|---|---|
|> 0|成功：就绪的文件描述符数量|
|0|超时|
|-1|失败，设置 errno|

#### 特性

无文件描述符数量限制：使用数组结构

分离监听和返回事件：events 和 revents 分开

效率问题：仍然需要线性扫描所有文件描述符

不能跨平台：仅支持 Linux/Unix

#### 代码示例 1：基本使用

```
#include <poll.h>
#include <stdio.h>
#include <unistd.h>

int main() {
    struct pollfd fds[1];
    int ret;
    
    // 设置监听
    fds[0].fd = STDIN_FILENO;
    fds[0].events = POLLIN;  // 监听可读事件
    
    printf("Waiting for input (timeout: 5s)...\n");
    
    // 等待 5 秒
    ret = poll(fds, 1, 5000);
    
    if (ret == -1) {
        perror("poll error");
    } else if (ret == 0) {
        printf("Timeout!\n");
    } else {
        if (fds[0].revents & POLLIN) {
            char buf[1024];
            ssize_t n = read(STDIN_FILENO, buf, sizeof(buf));
            if (n > 0) {
                buf[n] = '\0';
                printf("Read: %s", buf);
            }
        }
    }
    
    return 0;
}
```

#### 代码示例 2：TCP 服务器（多路转接）

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <poll.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <ctype.h>

#define PORT 8888
#define MAX_CLIENTS 1024
#define BUFFER_SIZE 1024

int main() {
    int listenfd, connfd;
    struct pollfd fds[MAX_CLIENTS + 1];  // +1 用于监听套接字
    struct sockaddr_in serv_addr, cli_addr;
    socklen_t cli_len;
    char buffer[BUFFER_SIZE];
    ssize_t n;
    char cli_ip[INET_ADDRSTRLEN];
    int nfds = 1;  // 当前有效监听数量
    
    // 创建监听套接字
    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd == -1) {
        perror("socket error");
        exit(1);
    }
    
    // 允许端口重用
    int opt = 1;
    setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    
    // 绑定地址
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    
    if (bind(listenfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr)) == -1) {
        perror("bind error");
        close(listenfd);
        exit(1);
    }
    
    // 监听
    if (listen(listenfd, 128) == -1) {
        perror("listen error");
        close(listenfd);
        exit(1);
    }
    
    printf("Poll server started on port %d\n", PORT);
    
    // 初始化 pollfd
    fds[0].fd = listenfd;
    fds[0].events = POLLIN;
    
    // 主循环
    while (1) {
        // 等待事件
        int ret = poll(fds, nfds, -1);
        
        if (ret == -1) {
            if (errno == EINTR) {
                continue;  // 被信号中断，重试
            }
            perror("poll error");
            break;
        }
        
        // 检查监听套接字（新连接）
        if (fds[0].revents & POLLIN) {
            cli_len = sizeof(cli_addr);
            connfd = accept(listenfd, (struct sockaddr*)&cli_addr, &cli_len);
            
            if (connfd == -1) {
                perror("accept error");
                continue;
            }
            
            // 打印客户端信息
            inet_ntop(AF_INET, &cli_addr.sin_addr, cli_ip, sizeof(cli_ip));
            printf("New connection from %s:%d\n", 
                   cli_ip, ntohs(cli_addr.sin_port));
            
            // 添加到监听数组
            if (nfds >= MAX_CLIENTS) {
                printf("Too many clients, closing connection\n");
                close(connfd);
            } else {
                fds[nfds].fd = connfd;
                fds[nfds].events = POLLIN;
                nfds++;
            }
            
            if (--ret <= 0) {
                continue;  // 没有其他事件
            }
        }
        
        // 检查客户端套接字（数据到达）
        for (int i = 1; i < nfds; i++) {
            if (fds[i].revents & (POLLIN | POLLERR | POLLHUP | POLLNVAL)) {
                if (fds[i].revents & (POLLERR | POLLHUP | POLLNVAL)) {
                    // 错误或对端关闭
                    printf("Client %d closed or error\n", fds[i].fd);
                    close(fds[i].fd);
                    
                    // 从数组中移除
                    fds[i] = fds[nfds - 1];
                    nfds--;
                    i--;  // 重新检查当前位置
                    continue;
                }
                
                n = read(fds[i].fd, buffer, BUFFER_SIZE - 1);
                
                if (n == -1) {
                    perror("read error");
                    close(fds[i].fd);
                    fds[i] = fds[nfds - 1];
                    nfds--;
                    i--;
                } else if (n == 0) {
                    // 客户端关闭连接
                    printf("Client %d closed connection\n", fds[i].fd);
                    close(fds[i].fd);
                    fds[i] = fds[nfds - 1];
                    nfds--;
                    i--;
                } else {
                    buffer[n] = '\0';
                    printf("Received from client %d: %s", fds[i].fd, buffer);
                    
                    // 转大写
                    for (int j = 0; j < n; j++) {
                        buffer[j] = toupper(buffer[j]);
                    }
                    
                    // 发送回客户端
                    write(fds[i].fd, buffer, n);
                }
                
                if (--ret <= 0) {
                    break;  // 没有其他事件
                }
            }
        }
    }
    
    close(listenfd);
    return 0;
}
```

## 📚 四、epoll 函数

### 📌 epoll_create - 创建 epoll 实例

#### 头文件

```
#include <sys/epoll.h>
```

#### 函数原型

```
int epoll_create(int size);
int epoll_create1(int flags);  // 新版本
```

#### 参数说明

|参数|说明|
|---|---|
|size|建议的监听节点数量（仅供内核参考）|
|flags|标志位：<br><br>0 - 默认<br><br>EPOLL_CLOEXEC - 执行 exec 时关闭|

#### 返回值

|返回值|说明|
|---|---|
|> 0|成功：epoll 文件描述符|
|-1|失败，设置 errno|

#### 特性

创建一个 epoll 实例

返回的 fd 用于后续的 epoll_ctl () 和 epoll_wait ()

内核使用红黑树管理文件描述符

### 📌 epoll_ctl - 管理 epoll 监听

#### 函数原型

```
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

epoll_event 结构
```
struct epoll_event {
    uint32_t events;  // 监听的事件
    union {
        void *ptr;
        int fd;
        uint32_t u32;
        uint64_t u64;
    } data;  // 用户数据
};
```

#### 操作类型（op）

|操作|说明|
|---|---|
|EPOLL_CTL_ADD|添加 fd 到监听|
|EPOLL_CTL_MOD|修改 fd 的监听事件|
|EPOLL_CTL_DEL|从监听中删除 fd|

#### 事件类型（events）

|事件|说明|
|---|---|
|EPOLLIN|可读|
|EPOLLOUT|可写|
|EPOLLERR|错误|
|EPOLLHUP|挂起|
|EPOLLET|边缘触发模式（ET）|
|EPOLLONESHOT|一次性事件|

#### 参数说明

|参数|说明|
|---|---|
|epfd|epoll_create 返回的 fd|
|op|操作类型|
|fd|要操作的文件描述符|
|event|epoll_event 结构体指针|

#### 返回值

|返回值|说明|
|---|---|
|0|成功|
|-1|失败，设置 errno|

#### 特性

动态管理监听的文件描述符

支持添加、修改、删除操作

无需每次都传递所有 fd

#### 代码示例

```
#include <sys/epoll.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main() {
    int epfd, fd = STDIN_FILENO;
    struct epoll_event event;
    
    // 创建 epoll
    epfd = epoll_create(1024);
    if (epfd == -1) {
        perror("epoll_create error");
        exit(1);
    }
    
    // 设置事件
    event.events = EPOLLIN;  // 监听可读
    event.data.fd = fd;      // 保存 fd
    
    // 添加到 epoll
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &event) == -1) {
        perror("epoll_ctl error");
        close(epfd);
        exit(1);
    }
    
    printf("Added fd %d to epoll\n", fd);
    
    // 修改事件
    event.events = EPOLLIN | EPOLLOUT;
    if (epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &event) == -1) {
        perror("epoll_ctl error");
        close(epfd);
        exit(1);
    }
    
    printf("Modified events for fd %d\n", fd);
    
    // 从 epoll 中删除
    if (epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL) == -1) {
        perror("epoll_ctl error");
        close(epfd);
        exit(1);
    }
    
    printf("Removed fd %d from epoll\n", fd);
    
    close(epfd);
    return 0;
}
```

### 📌 epoll_wait - 等待事件

#### 函数原型

```
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

#### 参数说明

|参数|说明|
|---|---|
|epfd|epoll_create 返回的 fd|
|events|传出参数，就绪事件数组|
|maxevents|events 数组的最大容量|
|timeout|超时时间（毫秒）：<br><br>-1 - 永久阻塞<br><br>0 - 立即返回（非阻塞）<br><br>>0 - 等待指定毫秒数|

#### 返回值

|返回值|说明|
|---|---|
|> 0|成功：就绪的事件数量|
|0|超时|
|-1|失败，设置 errno|

#### 特性

阻塞等待事件就绪

返回的 events 数组只包含就绪的事件

无需遍历所有文件描述符

#### 代码示例

```
#include <sys/epoll.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define MAX_EVENTS 10

int main() {
    int epfd, nfds;
    struct epoll_event event, events[MAX_EVENTS];
    
    // 创建 epoll
    epfd = epoll_create(1024);
    if (epfd == -1) {
        perror("epoll_create error");
        exit(1);
    }
    
    // 添加标准输入
    event.events = EPOLLIN;
    event.data.fd = STDIN_FILENO;
    
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, STDIN_FILENO, &event) == -1) {
        perror("epoll_ctl error");
        close(epfd);
        exit(1);
    }
    
    printf("Waiting for input...\n");
    
    // 等待事件
    nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
    
    if (nfds == -1) {
        perror("epoll_wait error");
        close(epfd);
        exit(1);
    }
    
    // 处理就绪事件
    for (int i = 0; i < nfds; i++) {
        if (events[i].data.fd == STDIN_FILENO) {
            char buf[1024];
            ssize_t n = read(STDIN_FILENO, buf, sizeof(buf));
            if (n > 0) {
                buf[n] = '\0';
                printf("Read: %s", buf);
            }
        }
    }
    
    close(epfd);
    return 0;
}
```

## 📚 五、完整 epoll 服务器示例

### 💡 epoll LT 模式服务器

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/epoll.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <ctype.h>
#include <errno.h>

#define PORT 8888
#define MAX_EVENTS 1024
#define BUFFER_SIZE 1024

int main() {
    int listenfd, connfd, epfd, nfds;
    struct sockaddr_in serv_addr, cli_addr;
    socklen_t cli_len;
    struct epoll_event event, events[MAX_EVENTS];
    char buffer[BUFFER_SIZE];
    ssize_t n;
    char cli_ip[INET_ADDRSTRLEN];
    
    // 创建监听套接字
    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd == -1) {
        perror("socket error");
        exit(1);
    }
    
    // 允许端口重用
    int opt = 1;
    setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    
    // 绑定地址
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    
    if (bind(listenfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr)) == -1) {
        perror("bind error");
        close(listenfd);
        exit(1);
    }
    
    // 监听
    if (listen(listenfd, 128) == -1) {
        perror("listen error");
        close(listenfd);
        exit(1);
    }
    
    // 创建 epoll
    epfd = epoll_create(MAX_EVENTS);
    if (epfd == -1) {
        perror("epoll_create error");
        close(listenfd);
        exit(1);
    }
    
    // 添加监听套接字
    event.events = EPOLLIN;
    event.data.fd = listenfd;
    
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, listenfd, &event) == -1) {
        perror("epoll_ctl error");
        close(listenfd);
        close(epfd);
        exit(1);
    }
    
    printf("Epoll LT server started on port %d\n", PORT);
    
    // 主循环
    while (1) {
        // 等待事件
        nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        
        if (nfds == -1) {
            if (errno == EINTR) {
                continue;  // 被信号中断，重试
            }
            perror("epoll_wait error");
            break;
        }
        
        // 处理就绪事件
        for (int i = 0; i < nfds; i++) {
            int sockfd = events[i].data.fd;
            
            // 监听套接字：新连接
            if (sockfd == listenfd) {
                cli_len = sizeof(cli_addr);
                connfd = accept(listenfd, (struct sockaddr*)&cli_addr, &cli_len);
                
                if (connfd == -1) {
                    perror("accept error");
                    continue;
                }
                
                // 打印客户端信息
                inet_ntop(AF_INET, &cli_addr.sin_addr, cli_ip, sizeof(cli_ip));
                printf("New connection from %s:%d\n", 
                       cli_ip, ntohs(cli_addr.sin_port));
                
                // 设置为非阻塞（可选）
                // int flags = fcntl(connfd, F_GETFL, 0);
                // fcntl(connfd, F_SETFL, flags | O_NONBLOCK);
                
                // 添加到 epoll
                event.events = EPOLLIN;
                event.data.fd = connfd;
                
                if (epoll_ctl(epfd, EPOLL_CTL_ADD, connfd, &event) == -1) {
                    perror("epoll_ctl error");
                    close(connfd);
                }
            } 
            // 客户端套接字：数据到达
            else {
                n = read(sockfd, buffer, BUFFER_SIZE - 1);
                
                if (n == -1) {
                    if (errno == EINTR) {
                        continue;
                    }
                    perror("read error");
                    close(sockfd);
                    epoll_ctl(epfd, EPOLL_CTL_DEL, sockfd, NULL);
                } else if (n == 0) {
                    // 客户端关闭连接
                    printf("Client %d closed connection\n", sockfd);
                    close(sockfd);
                    epoll_ctl(epfd, EPOLL_CTL_DEL, sockfd, NULL);
                } else {
                    buffer[n] = '\0';
                    printf("Received from client %d: %s", sockfd, buffer);
                    
                    // 转大写
                    for (int j = 0; j < n; j++) {
                        buffer[j] = toupper(buffer[j]);
                    }
                    
                    // 发送回客户端
                    write(sockfd, buffer, n);
                }
            }
        }
    }
    
    close(listenfd);
    close(epfd);
    return 0;
}
```

### 💡 epoll ET 模式服务器（非阻塞）

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/epoll.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <ctype.h>
#include <errno.h>
#include <fcntl.h>

#define PORT 8888
#define MAX_EVENTS 1024
#define BUFFER_SIZE 4096

// 设置非阻塞
void set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main() {
    int listenfd, connfd, epfd, nfds;
    struct sockaddr_in serv_addr, cli_addr;
    socklen_t cli_len;
    struct epoll_event event, events[MAX_EVENTS];
    char buffer[BUFFER_SIZE];
    ssize_t n;
    char cli_ip[INET_ADDRSTRLEN];
    
    // 创建监听套接字
    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd == -1) {
        perror("socket error");
        exit(1);
    }
    
    // 允许端口重用
    int opt = 1;
    setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    
    // 绑定地址
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    
    if (bind(listenfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr)) == -1) {
        perror("bind error");
        close(listenfd);
        exit(1);
    }
    
    // 监听
    if (listen(listenfd, 128) == -1) {
        perror("listen error");
        close(listenfd);
        exit(1);
    }
    
    // 设置监听套接字为非阻塞(1)
    set_nonblocking(listenfd);
    
    // 创建 epoll
    epfd = epoll_create(MAX_EVENTS);
    if (epfd == -1) {
        perror("epoll_create error");
        close(listenfd);
        exit(1);
    }
    
    // 添加监听套接字（ET 模式）(2)
    event.events = EPOLLIN | EPOLLET;
    event.data.fd = listenfd;
    
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, listenfd, &event) == -1) {
        perror("epoll_ctl error");
        close(listenfd);
        close(epfd);
        exit(1);
    }
    
    printf("Epoll ET server started on port %d\n", PORT);
    
    // 主循环
    while (1) {
        // 等待事件
        nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        
        if (nfds == -1) {
            if (errno == EINTR) {
                continue;  // 被信号中断，重试
            }
            perror("epoll_wait error");
            break;
        }
        
        // 处理就绪事件
        for (int i = 0; i < nfds; i++) {
            int sockfd = events[i].data.fd;
            
            // 监听套接字：新连接（需要循环 accept）(3)
            if (sockfd == listenfd) {
                while (1) {
                    cli_len = sizeof(cli_addr);
                    connfd = accept(listenfd, (struct sockaddr*)&cli_addr, &cli_len);
                    
                    if (connfd == -1) {
                        if (errno == EAGAIN || errno == EWOULDBLOCK) {
                            // 所有连接已处理完
                            break;
                        }
                        perror("accept error");
                        break;
                    }
                    
                    // 打印客户端信息
                    inet_ntop(AF_INET, &cli_addr.sin_addr, cli_ip, sizeof(cli_ip));
                    printf("New connection from %s:%d\n", 
                           cli_ip, ntohs(cli_addr.sin_port));
                    
                    // 设置为非阻塞(4)
                    set_nonblocking(connfd);
                    
                    // 添加到 epoll（ET 模式）(5)
                    event.events = EPOLLIN | EPOLLET;
                    event.data.fd = connfd;
                    
                    if (epoll_ctl(epfd, EPOLL_CTL_ADD, connfd, &event) == -1) {
                        perror("epoll_ctl error");
                        close(connfd);
                    }
                }
            } 
            // 客户端套接字：数据到达（需要循环 read）(6)
            else if (events[i].events & EPOLLIN) {
                while (1) {
                    n = read(sockfd, buffer, BUFFER_SIZE - 1);
                    
                    if (n == -1) {
                        if (errno == EAGAIN || errno == EWOULDBLOCK) {
                            // 数据已读完
                            break;
                        }
                        perror("read error");
                        close(sockfd);
                        epoll_ctl(epoll_ctl_DEL, sockfd, NULL);
                        break;
                    } else if (n == 0) {
                        // 客户端关闭连接
                        printf("Client %d closed connection\n", sockfd);
                        close(sockfd);
                        epoll_ctl(epfd, EPOLL_CTL_DEL, sockfd, NULL);
                        break;
                    } else {
                        buffer[n] = '\0';
                        printf("Received from client %d: %s", sockfd, buffer);
                        
                        // 转大写
                        for (int j = 0; j < n; j++) {
                            buffer[j] = toupper(buffer[j]);
                        }
                        
                        // 发送回客户端（也可以注册写事件）
                        write(sockfd, buffer, n);
                    }
                }
            }
        }
    }
    
    close(listenfd);
    close(epfd);
    return 0;
}
```

## 📚 六、epoll 反应堆模型（Reactor）

### 💡 epoll ET + 非阻塞 + 回调函数

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/epoll.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <ctype.h>
#include <errno.h>
#include <fcntl.h>

#define PORT 8888
#define MAX_EVENTS 1024
#define BUFFER_SIZE 4096

// 事件结构体
typedef struct {
    int fd;                    // 文件描述符
    int events;                // 监听的事件
    void (*callback)(int fd, int events, void *arg);  // 回调函数
    void *arg;                 // 回调函数参数
    char buffer[BUFFER_SIZE];  // 缓冲区
    int len;                   // 缓冲区中数据长度
    int offset;                // 写入偏移量（用于部分写入）
} Event;

// 全局 epoll fd
int epfd;

// 设置非阻塞
void set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

// 添加事件
void event_add(Event *ev) {
    struct epoll_event event;
    event.events = ev->events;
    event.data.ptr = ev;
    
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, ev->fd, &event) == -1) {
        perror("epoll_ctl ADD error");
    }
}

// 修改事件
void event_mod(Event *ev) {
    struct epoll_event event;
    event.events = ev->events;
    event.data.ptr = ev;
    
    if (epoll_ctl(epfd, EPOLL_CTL_MOD, ev->fd, &event) == -1) {
        perror("epoll_ctl MOD error");
    }
}

// 删除事件
void event_del(Event *ev) {
    if (epoll_ctl(epfd, EPOLL_CTL_DEL, ev->fd, NULL) == -1) {
        perror("epoll_ctl DEL error");
    }
}

// 接受连接回调函数
void accept_callback(int fd, int events, void *arg) {
    struct sockaddr_in cli_addr;
    socklen_t cli_len = sizeof(cli_addr);
    int connfd;
    char cli_ip[INET_ADDRSTRLEN];
    
    while (1) {
        connfd = accept(fd, (struct sockaddr*)&cli_addr, &cli_len);
        
        if (connfd == -1) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                break;  // 所有连接已处理完
            }
            perror("accept error");
            break;
        }
        
        // 打印客户端信息
        inet_ntop(AF_INET, &cli_addr.sin_addr, cli_ip, sizeof(cli_ip));
        printf("New connection from %s:%d\n", cli_ip, ntohs(cli_addr.sin_port));
        
        // 设置为非阻塞
        set_nonblocking(connfd);
        
        // 创建客户端事件
        Event *ev = malloc(sizeof(Event));
        if (!ev) {
            perror("malloc error");
            close(connfd);
            continue;
        }
        
        ev->fd = connfd;
        ev->events = EPOLLIN | EPOLLET;
        ev->callback = client_callback;  // 提前绑定统一回调函数
        ev->arg = ev;
        ev->len = 0;
        ev->offset = 0;
        
        // 注册读事件
        event_add(ev);
    }
}

// 客户端统一回调函数（处理读和写）
void client_callback(int fd, int events, void *arg) {
    Event *ev = (Event*)arg;
    
    // 检查错误或关闭
    if (events & (EPOLLERR | EPOLLHUP)) {
        printf("Client %d error or closed\n", fd);
        close(fd);
        event_del(ev);
        free(ev);
        return;
    }
    
    // 可读事件
    if (events & EPOLLIN) {
        handle_read(ev);
    }
    
    // 可写事件
    if (events & EPOLLOUT) {
        handle_write(ev);
    }
}

// 处理读取
void handle_read(Event *ev) {
    ssize_t n;
    
    while (1) {
        n = read(ev->fd, ev->buffer + ev->len, BUFFER_SIZE - ev->len - 1);
        
        if (n == -1) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // 数据已读完
                if (ev->len > 0) {
                    ev->buffer[ev->len] = '\0';
                    printf("Received from client %d: %s", ev->fd, ev->buffer);
                    
                    // 转大写
                    for (int i = 0; i < ev->len; i++) {
                        ev->buffer[i] = toupper(ev->buffer[i]);
                    }
                    
                    // 准备发送，修改为写事件
                    ev->offset = 0;
                    ev->events = EPOLLOUT | EPOLLET;
                    event_mod(ev);
                }
                break;
            }
            perror("read error");
            close(ev->fd);
            event_del(ev);
            free(ev);
            break;
        } else if (n == 0) {
            // 客户端关闭连接
            printf("Client %d closed connection\n", ev->fd);
            close(ev->fd);
            event_del(ev);
            free(ev);
            break;
        } else {
            ev->len += n;
            // 检查缓冲区是否已满
            if (ev->len >= BUFFER_SIZE - 1) {
                ev->buffer[ev->len] = '\0';
                printf("Received from client %d: %s", ev->fd, ev->buffer);
                
                // 转大写
                for (int i = 0; i < ev->len; i++) {
                    ev->buffer[i] = toupper(ev->buffer[i]);
                }
                
                // 准备发送
                ev->offset = 0;
                ev->events = EPOLLOUT | EPOLLET;
                event_mod(ev);
                break;
            }
        }
    }
}

// 处理写入（带循环，处理部分写入）
void handle_write(Event *ev) {
    ssize_t n;
    
    while (ev->offset < ev->len) {
        n = write(ev->fd, ev->buffer + ev->offset, ev->len - ev->offset);
        
        if (n == -1) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // 缓冲区满，等待下次可写事件
                event_mod(ev);  // 确保继续监听写事件
                return;
            }
            // 真正的错误
            perror("write error");
            close(ev->fd);
            event_del(ev);
            free(ev);
            return;
        }
        
        // 成功写入
        ev->offset += n;
    }
    
    // 全部写完，重置状态，切回读事件
    printf("Sent response to client %d\n", ev->fd);
    ev->len = 0;
    ev->offset = 0;
    ev->events = EPOLLIN | EPOLLET;
    event_mod(ev);
}

int main() {
    int listenfd;
    struct sockaddr_in serv_addr;
    struct epoll_event events[MAX_EVENTS];
    int nfds;
    
    // 创建监听套接字
    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd == -1) {
        perror("socket error");
        exit(1);
    }
    
    // 允许端口重用
    int opt = 1;
    setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    
    // 绑定地址
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    
    if (bind(listenfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr)) == -1) {
        perror("bind error");
        close(listenfd);
        exit(1);
    }
    
    // 监听
    if (listen(listenfd, 128) == -1) {
        perror("listen error");
        close(listenfd);
        exit(1);
    }
    
    // 设置为非阻塞
    set_nonblocking(listenfd);
    
    // 创建 epoll
    epfd = epoll_create(MAX_EVENTS);
    if (epfd == -1) {
        perror("epoll_create error");
        close(listenfd);
        exit(1);
    }
    
    // 创建监听事件
    Event *listen_ev = malloc(sizeof(Event));
    if (!listen_ev) {
        perror("malloc error");
        close(listenfd);
        close(epfd);
        exit(1);
    }
    
    listen_ev->fd = listenfd;
    listen_ev->events = EPOLLIN | EPOLLET;
    listen_ev->callback = accept_callback;  // 提前绑定
    listen_ev->arg = listen_ev;
    listen_ev->len = 0;
    listen_ev->offset = 0;
    
    // 添加监听事件
    event_add(listen_ev);
    
    printf("Epoll Reactor server started on port %d\n", PORT);
    printf("Waiting for connections...\n");
    
    // 主循环
    while (1) {
        nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        
        if (nfds == -1) {
            if (errno == EINTR) {
                continue;
            }
            perror("epoll_wait error");
            break;
        }
        
        // 处理就绪事件
        for (int i = 0; i < nfds; i++) {
            Event *ev = (Event*)events[i].data.ptr;
            
            // 直接调用回调函数（已经提前设置好）
            if (ev->callback) {
                ev->callback(ev->fd, events[i].events, ev->arg);
            }
        }
    }
    
    // 清理资源
    event_del(listen_ev);
    free(listen_ev);
    close(listenfd);
    close(epfd);
    
    return 0;
}
```


## 📚 七、select、poll、epoll 对比总结

### 📊 性能对比表

|特性|select|poll|epoll|
|---|---|---|---|
|跨平台|✅|✅|❌ (Linux only)|
|文件描述符限制|1024|无限制|无限制|
|数据结构|位图 (fd_set)|数组 (pollfd)|红黑树 + 链表|
|内核拷贝|每次调用都拷贝|每次调用都拷贝|仅修改时拷贝|
|事件通知|返回所有 fd|返回所有 fd|仅返回就绪 fd|
|时间复杂度|O(n)|O(n)|O(1)|
|触发模式|LT|LT|LT + ET|
|适用场景|小规模并发|中等规模并发|大规模高并发|

### 💡 选择建议

select：跨平台需求，文件描述符较少（< 1024）

poll：文件描述符较多，但不需要极致性能

epoll：Linux 平台，高并发场景（推荐）

## 📚 八、突破 1024 文件描述符限制

### 📌 查看和修改限制

```
# 查看系统最大文件描述符数
cat /proc/sys/fs/file-max

# 查看当前用户限制
ulimit -n

# 临时修改（当前会话）
ulimit -n 65536

# 永久修改
sudo vim /etc/security/limits.conf

# 添加以下内容
* soft nofile 65536
* hard nofile 100000

# 重启后生效
```

### 📌 代码中设置

```
#include <sys/resource.h>

void set_max_fds() {
    struct rlimit rl;
    
    // 获取当前限制
    getrlimit(RLIMIT_NOFILE, &rl);
    printf("Current max fds: %ld\n", rl.rlim_cur);
    
    // 设置新限制
    rl.rlim_cur = 65536;
    rl.rlim_max = 100000;
    
    if (setrlimit(RLIMIT_NOFILE, &rl) == -1) {
        perror("setrlimit error");
    } else {
        printf("Max fds set to: %ld\n", rl.rlim_cur);
    }
}
```

## 📚 九、常见错误和注意事项

### ⚠️ 1. select 的 nfds 参数

```
// ❌ 错误：nfds 应该是最大 fd + 1
FD_SET(100, &readfds);
select(10, &readfds, NULL, NULL, NULL);  // 不会监听 fd 100

// ✅ 正确
FD_SET(100, &readfds);
select(101, &readfds, NULL, NULL, NULL);  // 正确
```

### ⚠️ 2. ET 模式必须非阻塞

```
// ❌ 错误：ET 模式 + 阻塞 IO 会导致卡死
event.events = EPOLLIN | EPOLLET;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &event);
// fd 是阻塞的

// ✅ 正确：必须设置为非阻塞
set_nonblocking(fd);
event.events = EPOLLIN | EPOLLET;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &event);
```

### ⚠️ 3. epoll_wait 返回后要循环读取

```
// ❌ 错误：只读一次
if (events[i].events & EPOLLIN) {
    read(fd, buf, sizeof(buf));  // 可能漏读数据
}

// ✅ 正确：循环读取直到 EAGAIN
if (events[i].events & EPOLLIN) {
    while (1) {
        n = read(fd, buf, sizeof(buf));
        if (n == -1 && (errno == EAGAIN || errno == EWOULDBLOCK)) {
            break;  // 数据已读完
        }
        // 处理数据
    }
}
```