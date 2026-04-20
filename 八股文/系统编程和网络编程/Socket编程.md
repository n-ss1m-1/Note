下面我按照头文件、函数原型、参数说明、返回值、特性、代码示例的格式，详细整理和补充 Socket 编程的 API。

## 📚 一、Socket 基础函数

### 📌 1. socket () - 创建套接字

#### 头文件

c

运行

```
#include <sys/socket.h>
```

#### 函数原型

c

运行

```
int socket(int domain, int type, int protocol);
```

#### 参数说明

表格

|参数|说明|
|---|---|
|domain|协议族：<br><br>AF_INET - IPv4<br><br>AF_INET6 - IPv6<br><br>AF_UNIX - 本地套接字|
|type|套接字类型：<br><br>SOCK_STREAM - TCP（流式）<br><br>SOCK_DGRAM - UDP（数据报）<br><br>SOCK_RAW - 原始套接字|
|protocol|协议：<br><br>0 - 默认协议（推荐）<br><br>IPPROTO_TCP - TCP<br><br>IPPROTO_UDP - UDP|

#### 返回值

表格

|返回值|说明|
|---|---|
|> 0|成功：返回套接字文件描述符|
|-1|失败：返回 -1，设置 errno|

#### 特性

创建一个通信端点

返回的文件描述符用于后续的 bind ()、connect ()、listen () 等操作

需要手动关闭：close (fd)

#### 代码示例

c

运行

```
#include <sys/socket.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>

int main() {
    int sockfd;
    
    // 创建TCP套接字
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd == -1) {
        perror("socket error");
        exit(1);
    }
    printf("TCP socket created, fd = %d\n", sockfd);
    
    // 创建UDP套接字
    int udpfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (udpfd == -1) {
        perror("socket error");
        exit(1);
    }
    printf("UDP socket created, fd = %d\n", udpfd);
    
    close(sockfd);
    close(udpfd);
    
    return 0;
}
```

### 📌 2. bind () - 绑定地址

#### 头文件

c

运行

```
#include <sys/socket.h>
```

#### 函数原型

c

运行

```
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

#### 参数说明

表格

|参数|说明|
|---|---|
|sockfd|socket () 返回的文件描述符|
|addr|指向地址结构的指针（struct sockaddr_in 或 struct sockaddr_in6）|
|addrlen|地址结构的大小|

#### 返回值

表格

|返回值|说明|
|---|---|
|0|成功|
|-1|失败，设置 errno|

#### 特性

将套接字与特定的 IP 地址和端口号绑定

服务器必须调用 bind ()，客户端可选（隐式绑定）

INADDR_ANY 表示绑定到所有可用的网络接口

#### 代码示例

c

运行

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define PORT 8888

int main() {
    int sockfd;
    struct sockaddr_in serv_addr;
    
    // 创建套接字
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd == -1) {
        perror("socket error");
        exit(1);
    }
    
    // 初始化地址结构
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(PORT);  // 本地->网络字节序
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);  // 绑定到所有接口
    
    // 绑定地址
    if (bind(sockfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr)) == -1) {
        perror("bind error");
        close(sockfd);
        exit(1);
    }
    
    printf("Socket bound to port %d\n", PORT);
    
    close(sockfd);
    return 0;
}
```

### 📌 3. listen () - 监听连接

#### 头文件

c

运行

```
#include <sys/socket.h>
```

#### 函数原型

c

运行

```
int listen(int sockfd, int backlog);
```

#### 参数说明

表格

|参数|说明|
|---|---|
|sockfd|已绑定的套接字文件描述符|
|backlog|连接队列的最大长度（通常设为 128）|

#### 返回值

表格

|返回值|说明|
|---|---|
|0|成功|
|-1|失败，设置 errno|

#### 特性

将套接字设置为被动模式（监听模式）

仅用于 TCP 服务器

backlog 指定了未完成连接队列的最大长度

#### 代码示例

c

运行

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define PORT 8888
#define BACKLOG 128

int main() {
    int sockfd;
    struct sockaddr_in serv_addr;
    
    // 创建套接字
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd == -1) {
        perror("socket error");
        exit(1);
    }
    
    // 初始化地址结构
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    
    // 绑定地址
    if (bind(sockfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr)) == -1) {
        perror("bind error");
        close(sockfd);
        exit(1);
    }
    
    // 监听连接
    if (listen(sockfd, BACKLOG) == -1) {
        perror("listen error");
        close(sockfd);
        exit(1);
    }
    
    printf("Server listening on port %d, backlog = %d\n", PORT, BACKLOG);
    
    close(sockfd);
    return 0;
}
```

### 📌 4. accept () - 接受连接

#### 头文件

c

运行

```
#include <sys/socket.h>
```

#### 函数原型

c

运行

```
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

#### 参数说明

表格

|参数|说明|
|---|---|
|sockfd|监听套接字的文件描述符|
|addr|传出参数，客户端地址结构|
|addrlen|传入传出参数，地址结构的大小|

#### 返回值

表格

|返回值|说明|
|---|---|
|> 0|成功：返回新的通信套接字文件描述符|
|-1|失败，设置 errno|

#### 特性

阻塞等待客户端连接

成功时返回一个新的套接字，用于与客户端通信

原来的监听套接字继续监听新的连接

#### 代码示例

c

运行

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 8888

int main() {
    int listenfd, connfd;
    struct sockaddr_in serv_addr, cli_addr;
    socklen_t cli_len;
    char cli_ip[INET_ADDRSTRLEN];
    
    // 创建监听套接字
    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd == -1) {
        perror("socket error");
        exit(1);
    }
    
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
    
    printf("Server listening on port %d...\n", PORT);
    
    // 接受连接
    cli_len = sizeof(cli_addr);
    connfd = accept(listenfd, (struct sockaddr*)&cli_addr, &cli_len);
    if (connfd == -1) {
        perror("accept error");
        close(listenfd);
        exit(1);
    }
    
    // 打印客户端信息
    inet_ntop(AF_INET, &cli_addr.sin_addr, cli_ip, sizeof(cli_ip));
    printf("Client connected: IP=%s, Port=%d\n", 
           cli_ip, ntohs(cli_addr.sin_port));
    
    // 关闭套接字
    close(connfd);
    close(listenfd);
    
    return 0;
}
```

### 📌 5. connect () - 建立连接

#### 头文件

c

运行

```
#include <sys/socket.h>
```

#### 函数原型

c

运行

```
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

#### 参数说明

表格

|参数|说明|
|---|---|
|sockfd|客户端套接字文件描述符|
|addr|服务器地址结构|
|addrlen|地址结构的大小|

#### 返回值

表格

|返回值|说明|
|---|---|
|0|成功|
|-1|失败，设置 errno|

#### 特性

客户端主动发起连接

阻塞直到连接建立或失败

TCP 三次握手在此函数中完成

#### 代码示例

c

运行

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define SERVER_IP "127.0.0.1"
#define SERVER_PORT 8888

int main() {
    int sockfd;
    struct sockaddr_in serv_addr;
    
    // 创建套接字
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd == -1) {
        perror("socket error");
        exit(1);
    }
    
    // 初始化服务器地址
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERVER_PORT);
    
    // 转换IP地址
    if (inet_pton(AF_INET, SERVER_IP, &serv_addr.sin_addr) <= 0) {
        perror("inet_pton error");
        close(sockfd);
        exit(1);
    }
    
    // 连接服务器
    if (connect(sockfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr)) == -1) {
        perror("connect error");
        close(sockfd);
        exit(1);
    }
    
    printf("Connected to server %s:%d\n", SERVER_IP, SERVER_PORT);
    
    close(sockfd);
    return 0;
}
```

## 📚 二、数据传输函数

### 📌 1. read () /recv () - 接收数据

#### 头文件

c

运行

```
#include <unistd.h>      // read()
#include <sys/socket.h>  // recv()
```

#### 函数原型

c

运行

```
ssize_t read(int fd, void *buf, size_t count);
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
```

#### 参数说明

表格

|参数|说明|
|---|---|
|fd/sockfd|套接字文件描述符|
|buf|接收缓冲区|
|count/len|缓冲区大小|
|flags|标志（recv () 特有，通常为 0）|

#### 返回值

表格

|返回值|说明|
|---|---|
|> 0|成功：实际接收到的字节数|
|0|对端关闭连接|
|-1|失败，设置 errno|

#### 特性

read () 是通用的文件读取函数

recv () 是专门用于套接字的函数，支持更多标志位

阻塞模式下会等待数据到达

#### 代码示例

c

运行

```
#include <sys/socket.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define BUFFER_SIZE 1024

void handle_client(int connfd) {
    char buffer[BUFFER_SIZE];
    ssize_t n;
    
    while (1) {
        // 接收数据
        n = read(connfd, buffer, BUFFER_SIZE - 1);
        
        if (n == -1) {
            perror("read error");
            break;
        } else if (n == 0) {
            printf("Client closed connection\n");
            break;
        }
        
        buffer[n] = '\0';  // 添加字符串结束符
        printf("Received %zd bytes: %s", n, buffer);
        
        // 处理数据（例如：转大写）
        for (int i = 0; i < n; i++) {
            buffer[i] = toupper(buffer[i]);
        }
        
        // 发送回客户端
        write(connfd, buffer, n);
    }
}
```

### 📌 2. write () /send () - 发送数据

#### 头文件

c

运行

```
#include <unistd.h>      // write()
#include <sys/socket.h>  // send()
```

#### 函数原型

c

运行

```
ssize_t write(int fd, const void *buf, size_t count);
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
```

#### 参数说明

表格

|参数|说明|
|---|---|
|fd/sockfd|套接字文件描述符|
|buf|发送缓冲区|
|count/len|要发送的数据大小|
|flags|标志（send () 特有，通常为 0）|

#### 返回值

表格

|返回值|说明|
|---|---|
|> 0|成功：实际发送的字节数|
|-1|失败，设置 errno|

#### 特性

write () 是通用的文件写入函数

send () 是专门用于套接字的函数，支持更多标志位

可能只发送部分数据，需要循环发送

#### 代码示例

c

运行

```
#include <sys/socket.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define BUFFER_SIZE 1024

int send_all(int sockfd, const char *buf, size_t len) {
    size_t total = 0;
    ssize_t n;
    
    while (total < len) {
        n = send(sockfd, buf + total, len - total, 0);
        
        if (n == -1) {
            if (errno == EINTR) {
                // 被信号中断，重试
                continue;
            }
            perror("send error");
            return -1;
        }
        
        total += n;
    }
    
    return total;
}

int main() {
    int sockfd;
    char *message = "Hello, Server!";
    
    // 假设已经建立了连接
    // sockfd = socket(...);
    // connect(...);
    
    // 发送数据
    if (send_all(sockfd, message, strlen(message)) == -1) {
        fprintf(stderr, "Failed to send message\n");
        return 1;
    }
    
    printf("Message sent successfully\n");
    
    return 0;
}
```

### 📌 3. recvfrom () - UDP 接收数据

#### 头文件

c

运行

```
#include <sys/socket.h>
```

#### 函数原型

c

运行

```
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                 struct sockaddr *src_addr, socklen_t *addrlen);
```

#### 参数说明

表格

|参数|说明|
|---|---|
|sockfd|UDP 套接字文件描述符|
|buf|接收缓冲区|
|len|缓冲区大小|
|flags|标志（通常为 0）|
|src_addr|传出参数，发送方地址|
|addrlen|传入传出参数，地址结构大小|

#### 返回值

表格

|返回值|说明|
|---|---|
|> 0|成功：实际接收到的字节数|
|-1|失败，设置 errno|

#### 特性

用于 UDP 套接字接收数据

同时获取发送方的地址信息

无连接，不需要 accept ()

#### 代码示例

c

运行

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define PORT 8888
#define BUFFER_SIZE 1024

int main() {
    int sockfd;
    struct sockaddr_in serv_addr, cli_addr;
    socklen_t cli_len;
    char buffer[BUFFER_SIZE];
    ssize_t n;
    char cli_ip[INET_ADDRSTRLEN];
    
    // 创建UDP套接字
    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd == -1) {
        perror("socket error");
        exit(1);
    }
    
    // 绑定地址
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    
    if (bind(sockfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr)) == -1) {
        perror("bind error");
        close(sockfd);
        exit(1);
    }
    
    printf("UDP server listening on port %d...\n", PORT);
    
    while (1) {
        cli_len = sizeof(cli_addr);
        
        // 接收数据
        n = recvfrom(sockfd, buffer, BUFFER_SIZE - 1, 0,
                     (struct sockaddr*)&cli_addr, &cli_len);
        
        if (n == -1) {
            perror("recvfrom error");
            continue;
        }
        
        buffer[n] = '\0';
        
        // 打印客户端信息
        inet_ntop(AF_INET, &cli_addr.sin_addr, cli_ip, sizeof(cli_ip));
        printf("Received from %s:%d - %s", 
               cli_ip, ntohs(cli_addr.sin_port), buffer);
        
        // 转大写并发送回客户端
        for (int i = 0; i < n; i++) {
            buffer[i] = toupper(buffer[i]);
        }
        
        sendto(sockfd, buffer, n, 0, 
               (struct sockaddr*)&cli_addr, cli_len);
    }
    
    close(sockfd);
    return 0;
}
```

### 📌 4. sendto () - UDP 发送数据

#### 头文件

c

运行

```
#include <sys/socket.h>
```

#### 函数原型

c

运行

```
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
               const struct sockaddr *dest_addr, socklen_t addrlen);
```

#### 参数说明

表格

|参数|说明|
|---|---|
|sockfd|UDP 套接字文件描述符|
|buf|发送缓冲区|
|len|要发送的数据大小|
|flags|标志（通常为 0）|
|dest_addr|目标地址结构|
|addrlen|地址结构大小|

#### 返回值

表格

|返回值|说明|
|---|---|
|> 0|成功：实际发送的字节数|
|-1|失败，设置 errno|

#### 特性

用于 UDP 套接字发送数据

需要指定目标地址

无连接，不需要 connect ()

#### 代码示例

c

运行

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define SERVER_IP "127.0.0.1"
#define SERVER_PORT 8888
#define BUFFER_SIZE 1024

int main() {
    int sockfd;
    struct sockaddr_in serv_addr;
    char buffer[BUFFER_SIZE];
    ssize_t n;
    
    // 创建UDP套接字
    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd == -1) {
        perror("socket error");
        exit(1);
    }
    
    // 初始化服务器地址
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERVER_PORT);
    
    if (inet_pton(AF_INET, SERVER_IP, &serv_addr.sin_addr) <= 0) {
        perror("inet_pton error");
        close(sockfd);
        exit(1);
    }
    
    printf("UDP client started, sending to %s:%d\n", SERVER_IP, SERVER_PORT);
    
    while (1) {
        printf("Enter message: ");
        fgets(buffer, BUFFER_SIZE, stdin);
        
        // 去除换行符
        buffer[strcspn(buffer, "\n")] = '\0';
        
        if (strcmp(buffer, "quit") == 0) {
            break;
        }
        
        // 发送数据
        n = sendto(sockfd, buffer, strlen(buffer), 0,
                   (struct sockaddr*)&serv_addr, sizeof(serv_addr));
        
        if (n == -1) {
            perror("sendto error");
            continue;
        }
        
        printf("Sent %zd bytes\n", n);
        
        // 接收响应
        n = recvfrom(sockfd, buffer, BUFFER_SIZE - 1, 0, NULL, NULL);
        if (n == -1) {
            perror("recvfrom error");
            continue;
        }
        
        buffer[n] = '\0';
        printf("Received: %s\n", buffer);
    }
    
    close(sockfd);
    return 0;
}
```

## 📚 三、网络字节序转换函数

### 📌 1. 字节序转换函数

#### 头文件

c

运行

```
#include <arpa/inet.h>
```

#### 函数原型

c

运行

```
uint32_t htonl(uint32_t hostlong);   // host to network long (32位)
uint16_t htons(uint16_t hostshort);  // host to network short (16位)
uint32_t ntohl(uint32_t netlong);    // network to host long
uint16_t ntohs(uint16_t netshort);   // network to host short
```

#### 参数说明

表格

|参数|说明|
|---|---|
|hostlong/hostshort|本地字节序的值|
|netlong/netshort|网络字节序的值|

#### 返回值

返回转换后的值

#### 特性

网络字节序是大端序（Big-Endian）

本地字节序可能是小端序（Little-Endian）或大端序

用于端口号和 IP 地址的转换

#### 代码示例

c

运行

```
#include <arpa/inet.h>
#include <stdio.h>
#include <stdint.h>

int main() {
    uint32_t ip = 0x12345678;  // 本地字节序
    uint16_t port = 8888;      // 本地字节序
    
    printf("Local IP: 0x%08x\n", ip);
    printf("Local Port: %d\n", port);
    
    // 转换为网络字节序
    uint32_t net_ip = htonl(ip);
    uint16_t net_port = htons(port);
    
    printf("Network IP: 0x%08x\n", net_ip);
    printf("Network Port: %d\n", net_port);
    
    // 转换回本地字节序
    uint32_t local_ip = ntohl(net_ip);
    uint16_t local_port = ntohs(net_port);
    
    printf("Back to Local IP: 0x%08x\n", local_ip);
    printf("Back to Local Port: %d\n", local_port);
    
    return 0;
}
```

### 📌 2. IP 地址转换函数

#### 头文件

c

运行

```
#include <arpa/inet.h>
```

#### 函数原型

c

运行

```
int inet_pton(int af, const char *src, void *dst);
const char *inet_ntop(int af, const void *src, char *dst, socklen_t size);
```

#### 参数说明

表格

|函数|参数|说明|
|---|---|---|
|inet_pton|af|地址族：AF_INET 或 AF_INET6|
||src|字符串形式的 IP 地址（如 "192.168.1.1"）|
||dst|传出参数，存储转换后的二进制地址|
|inet_ntop|af|地址族：AF_INET 或 AF_INET6|
||src|二进制形式的 IP 地址|
||dst|传出参数，存储转换后的字符串地址|
||size|dst 缓冲区的大小|

#### 返回值

表格

|函数|返回值|说明|
|---|---|---|
|inet_pton|1|成功|
||0|无效的 IP 地址|
||-1|失败，设置 errno|
|inet_ntop|非 NULL|成功，返回 dst|
||NULL|失败，设置 errno|

#### 特性

inet_pton：字符串 -> 二进制（Presentation to Network）

inet_ntop：二进制 -> 字符串（Network to Presentation）

支持 IPv4 和 IPv6

#### 代码示例

c

运行

```
#include <arpa/inet.h>
#include <stdio.h>
#include <string.h>

int main() {
    struct in_addr ip_addr;
    char ip_str[INET_ADDRSTRLEN];
    
    // 字符串 -> 二进制
    const char *ip_string = "192.168.1.100";
    
    if (inet_pton(AF_INET, ip_string, &ip_addr) <= 0) {
        perror("inet_pton error");
        return 1;
    }
    
    printf("String IP: %s\n", ip_string);
    printf("Binary IP: 0x%08x\n", ip_addr.s_addr);
    
    // 二进制 -> 字符串
    if (inet_ntop(AF_INET, &ip_addr, ip_str, sizeof(ip_str)) == NULL) {
        perror("inet_ntop error");
        return 1;
    }
    
    printf("Converted back: %s\n", ip_str);
    
    return 0;
}
```

## 📚 四、完整 TCP 服务器和客户端示例

### 💡 TCP 服务器完整示例

c

运行

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <ctype.h>
#include <signal.h>
#include <sys/wait.h>

#define PORT 8888
#define BACKLOG 128
#define BUFFER_SIZE 1024

// 信号处理函数（回收僵尸进程）
void sigchld_handler(int sig) {
    while (waitpid(-1, NULL, WNOHANG) > 0);
}

// 处理客户端连接
void handle_client(int connfd) {
    char buffer[BUFFER_SIZE];
    ssize_t n;
    
    while (1) {
        // 接收数据
        n = read(connfd, buffer, BUFFER_SIZE - 1);
        
        if (n == -1) {
            if (errno == EINTR) {
                continue;  // 被信号中断，重试
            }
            perror("read error");
            break;
        } else if (n == 0) {
            printf("Client closed connection\n");
            break;
        }
        
        buffer[n] = '\0';
        printf("Received from client: %s", buffer);
        
        // 转大写
        for (int i = 0; i < n; i++) {
            buffer[i] = toupper(buffer[i]);
        }
        
        // 发送回客户端
        if (write(connfd, buffer, n) == -1) {
            perror("write error");
            break;
        }
    }
    
    close(connfd);
}

int main() {
    int listenfd, connfd;
    struct sockaddr_in serv_addr, cli_addr;
    socklen_t cli_len;
    pid_t pid;
    char cli_ip[INET_ADDRSTRLEN];
    
    // 注册信号处理
    signal(SIGCHLD, sigchld_handler);
    
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
    if (listen(listenfd, BACKLOG) == -1) {
        perror("listen error");
        close(listenfd);
        exit(1);
    }
    
    printf("TCP server started on port %d\n", PORT);
    
    // 主循环
    while (1) {
        cli_len = sizeof(cli_addr);
        
        // 接受连接
        connfd = accept(listenfd, (struct sockaddr*)&cli_addr, &cli_len);
        if (connfd == -1) {
            if (errno == EINTR) {
                continue;  // 被信号中断，重试
            }
            perror("accept error");
            continue;
        }
        
        // 打印客户端信息
        inet_ntop(AF_INET, &cli_addr.sin_addr, cli_ip, sizeof(cli_ip));
        printf("New connection from %s:%d\n", 
               cli_ip, ntohs(cli_addr.sin_port));
        
        // 创建子进程处理客户端
        pid = fork();
        
        if (pid == -1) {
            perror("fork error");
            close(connfd);
            continue;
        }
        
        if (pid == 0) {
            // 子进程
            close(listenfd);  // 子进程不需要监听套接字
            handle_client(connfd);
            exit(0);
        } else {
            // 父进程
            close(connfd);  // 父进程不需要通信套接字
        }
    }
    
    close(listenfd);
    return 0;
}
```

### 💡 TCP 客户端完整示例

c

运行

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define SERVER_IP "127.0.0.1"
#define SERVER_PORT 8888
#define BUFFER_SIZE 1024

int main() {
    int sockfd;
    struct sockaddr_in serv_addr;
    char send_buf[BUFFER_SIZE];
    char recv_buf[BUFFER_SIZE];
    ssize_t n;
    
    // 创建套接字
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd == -1) {
        perror("socket error");
        exit(1);
    }
    
    // 初始化服务器地址
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERVER_PORT);
    
    if (inet_pton(AF_INET, SERVER_IP, &serv_addr.sin_addr) <= 0) {
        perror("inet_pton error");
        close(sockfd);
        exit(1);
    }
    
    // 连接服务器
    if (connect(sockfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr)) == -1) {
        perror("connect error");
        close(sockfd);
        exit(1);
    }
    
    printf("Connected to server %s:%d\n", SERVER_IP, SERVER_PORT);
    printf("Enter messages (type 'quit' to exit):\n");
    
    // 交互循环
    while (1) {
        printf("Client: ");
        fgets(send_buf, BUFFER_SIZE, stdin);
        
        // 去除换行符
        send_buf[strcspn(send_buf, "\n")] = '\0';
        
        if (strcmp(send_buf, "quit") == 0) {
            break;
        }
        
        // 发送数据
        n = write(sockfd, send_buf, strlen(send_buf));
        if (n == -1) {
            perror("write error");
            break;
        }
        
        // 接收响应
        n = read(sockfd, recv_buf, BUFFER_SIZE - 1);
        if (n == -1) {
            perror("read error");
            break;
        } else if (n == 0) {
            printf("Server closed connection\n");
            break;
        }
        
        recv_buf[n] = '\0';
        printf("Server: %s\n", recv_buf);
    }
    
    close(sockfd);
    printf("Disconnected from server\n");
    
    return 0;
}
```

## 📚 五、完整 UDP 服务器和客户端示例

### 💡 UDP 服务器完整示例

c

运行

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <ctype.h>

#define PORT 8888
#define BUFFER_SIZE 1024

int main() {
    int sockfd;
    struct sockaddr_in serv_addr, cli_addr;
    socklen_t cli_len;
    char buffer[BUFFER_SIZE];
    ssize_t n;
    char cli_ip[INET_ADDRSTRLEN];
    
    // 创建UDP套接字
    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd == -1) {
        perror("socket error");
        exit(1);
    }
    
    // 允许端口重用
    int opt = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    
    // 绑定地址
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(PORT);
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    
    if (bind(sockfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr)) == -1) {
        perror("bind error");
        close(sockfd);
        exit(1);
    }
    
    printf("UDP server started on port %d\n", PORT);
    
    // 主循环
    while (1) {
        cli_len = sizeof(cli_addr);
        
        // 接收数据
        n = recvfrom(sockfd, buffer, BUFFER_SIZE - 1, 0,
                     (struct sockaddr*)&cli_addr, &cli_len);
        
        if (n == -1) {
            perror("recvfrom error");
            continue;
        }
        
        buffer[n] = '\0';
        
        // 打印客户端信息
        inet_ntop(AF_INET, &cli_addr.sin_addr, cli_ip, sizeof(cli_ip));
        printf("Received from %s:%d - %s", 
               cli_ip, ntohs(cli_addr.sin_port), buffer);
        
        // 转大写
        for (int i = 0; i < n; i++) {
            buffer[i] = toupper(buffer[i]);
        }
        
        // 发送回客户端
        if (sendto(sockfd, buffer, n, 0, 
                   (struct sockaddr*)&cli_addr, cli_len) == -1) {
            perror("sendto error");
            continue;
        }
    }
    
    close(sockfd);
    return 0;
}
```

### 💡 UDP 客户端完整示例

c

运行

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define SERVER_IP "127.0.0.1"
#define SERVER_PORT 8888
#define BUFFER_SIZE 1024

int main() {
    int sockfd;
    struct sockaddr_in serv_addr;
    char send_buf[BUFFER_SIZE];
    char recv_buf[BUFFER_SIZE];
    ssize_t n;
    
    // 创建UDP套接字
    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd == -1) {
        perror("socket error");
        exit(1);
    }
    
    // 初始化服务器地址
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERVER_PORT);
    
    if (inet_pton(AF_INET, SERVER_IP, &serv_addr.sin_addr) <= 0) {
        perror("inet_pton error");
        close(sockfd);
        exit(1);
    }
    
    printf("UDP client started, sending to %s:%d\n", SERVER_IP, SERVER_PORT);
    printf("Enter messages (type 'quit' to exit):\n");
    
    // 交互循环
    while (1) {
        printf("Client: ");
        fgets(send_buf, BUFFER_SIZE, stdin);
        
        // 去除换行符
        send_buf[strcspn(send_buf, "\n")] = '\0';
        
        if (strcmp(send_buf, "quit") == 0) {
            break;
        }
        
        // 发送数据
        n = sendto(sockfd, send_buf, strlen(send_buf), 0,
                   (struct sockaddr*)&serv_addr, sizeof(serv_addr));
        
        if (n == -1) {
            perror("sendto error");
            continue;
        }
        
        printf("Sent %zd bytes\n", n);
        
        // 接收响应
        n = recvfrom(sockfd, recv_buf, BUFFER_SIZE - 1, 0, NULL, NULL);
        if (n == -1) {
            perror("recvfrom error");
            continue;
        }
        
        recv_buf[n] = '\0';
        printf("Server: %s\n", recv_buf);
    }
    
    close(sockfd);
    printf("Client stopped\n");
    
    return 0;
}
```

## 📚 六、多线程并发服务器示例

c

运行

```
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <ctype.h>
#include <pthread.h>

#define PORT 8888
#define BACKLOG 128
#define BUFFER_SIZE 1024

// 客户端信息结构
typedef struct {
    int connfd;
    struct sockaddr_in cli_addr;
} ClientInfo;

// 处理客户端的线程函数
void* handle_client(void* arg) {
    ClientInfo* client = (ClientInfo*)arg;
    int connfd = client->connfd;
    char cli_ip[INET_ADDRSTRLEN];
    char buffer[BUFFER_SIZE];
    ssize_t n;
    
    // 打印客户端信息
    inet_ntop(AF_INET, &client->cli_addr.sin_addr, cli_ip, sizeof(cli_ip));
    printf("[Thread %lu] Handling client %s:%d\n", 
           pthread_self(), cli_ip, ntohs(client->cli_addr.sin_port));
    
    // 释放内存
    free(client);
    
    // 设置线程分离
    pthread_detach(pthread_self());
    
    // 处理客户端请求
    while (1) {
        n = read(connfd, buffer, BUFFER_SIZE - 1);
        
        if (n == -1) {
            if (errno == EINTR) {
                continue;
            }
            perror("read error");
            break;
        } else if (n == 0) {
            printf("[Thread %lu] Client %s:%d closed connection\n", 
                   pthread_self(), cli_ip, ntohs(client->cli_addr.sin_port));
            break;
        }
        
        buffer[n] = '\0';
        printf("[Thread %lu] Received from %s:%d - %s", 
               pthread_self(), cli_ip, ntohs(client->cli_addr.sin_port), buffer);
        
        // 转大写
        for (int i = 0; i < n; i++) {
            buffer[i] = toupper(buffer[i]);
        }
        
        // 发送回客户端
        if (write(connfd, buffer, n) == -1) {
            perror("write error");
            break;
        }
    }
    
    close(connfd);
    printf("[Thread %lu] Client handler exiting\n", pthread_self());
    
    return NULL;
}

int main() {
    int listenfd, connfd;
    struct sockaddr_in serv_addr, cli_addr;
    socklen_t cli_len;
    pthread_t tid;
    ClientInfo* client;
    
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
    if (listen(listenfd, BACKLOG) == -1) {
        perror("listen error");
        close(listenfd);
        exit(1);
    }
    
    printf("Multi-threaded TCP server started on port %d\n", PORT);
    
    // 主循环
    while (1) {
        cli_len = sizeof(cli_addr);
        
        // 接受连接
        connfd = accept(listenfd, (struct sockaddr*)&cli_addr, &cli_len);
        if (connfd == -1) {
            if (errno == EINTR) {
                continue;
            }
            perror("accept error");
            continue;
        }
        
        // 分配客户端信息
        client = malloc(sizeof(ClientInfo));
        if (client == NULL) {
            perror("malloc error");
            close(connfd);
            continue;
        }
        
        client->connfd = connfd;
        client->cli_addr = cli_addr;
        
        // 创建线程处理客户端
        if (pthread_create(&tid, NULL, handle_client, client) != 0) {
            perror("pthread_create error");
            free(client);
            close(connfd);
            continue;
        }
    }
    
    close(listenfd);
    return 0;
}
```

## 📚 七、错误处理函数封装

c

运行

```
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <unistd.h>

// 通用错误处理
void perror_exit(const char *msg) {
    perror(msg);
    exit(EXIT_FAILURE);
}

// Socket错误处理
void socket_error(const char *msg) {
    fprintf(stderr, "Socket error: %s - %s\n", msg, strerror(errno));
    exit(EXIT_FAILURE);
}

// 读取数据（处理各种错误情况）
ssize_t read_with_error_handling(int fd, void *buf, size_t count) {
    ssize_t n;
    
    while (1) {
        n = read(fd, buf, count);
        
        if (n == -1) {
            if (errno == EINTR) {
                // 被信号中断，重试
                continue;
            } else if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // 非阻塞模式，没有数据可读
                return -2;
            } else if (errno == ECONNRESET) {
                // 对端重置连接
                fprintf(stderr, "Connection reset by peer\n");
                return -1;
            } else {
                perror("read error");
                return -1;
            }
        }
        
        return n;
    }
}

// 发送所有数据
ssize_t send_all(int sockfd, const void *buf, size_t len) {
    size_t total = 0;
    ssize_t n;
    
    while (total < len) {
        n = send(sockfd, (char*)buf + total, len - total, 0);
        
        if (n == -1) {
            if (errno == EINTR) {
                continue;
            }
            perror("send error");
            return -1;
        }
        
        total += n;
    }
    
    return total;
}

// 接收所有数据
ssize_t recv_all(int sockfd, void *buf, size_t len) {
    size_t total = 0;
    ssize_t n;
    
    while (total < len) {
        n = recv(sockfd, (char*)buf + total, len - total, 0);
        
        if (n == -1) {
            if (errno == EINTR) {
                continue;
            }
            perror("recv error");
            return -1;
        } else if (n == 0) {
            // 对端关闭连接
            break;
        }
        
        total += n;
    }
    
    return total;
}
```

## 📊 Socket 函数总结表

表格

|函数|用途|适用协议|阻塞 / 非阻塞|
|---|---|---|---|
|socket()|创建套接字|TCP/UDP|-|
|bind()|绑定地址|TCP/UDP|-|
|listen()|监听连接|TCP|-|
|accept()|接受连接|TCP|阻塞|
|connect()|建立连接|TCP|阻塞|
|read()/recv()|接收数据|TCP/UDP|阻塞|
|write()/send()|发送数据|TCP/UDP|阻塞|
|recvfrom()|接收数据 + 地址|UDP|阻塞|
|sendto()|发送数据 + 地址|UDP|阻塞|
|close()|关闭套接字|TCP/UDP|-|

## ⚠️ 常见错误和注意事项

1. **端口被占用**

c

运行

```
// 解决方案：设置 SO_REUSEADDR 选项
int opt = 1;
setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```

2. **读取返回 0**

c

运行

```
// 表示对端关闭连接
if (n == 0) {
    printf("Client closed connection\n");
    close(connfd);
}
```

3. **被信号中断**

c

运行

```
// 处理 EINTR 错误
if (errno == EINTR) {
    continue;  // 重试
}
```

4. **僵尸进程**

c

运行

```
// 多进程服务器：注册 SIGCHLD 信号处理
signal(SIGCHLD, sigchld_handler);

void sigchld_handler(int sig) {
    while (waitpid(-1, NULL, WNOHANG) > 0);
}
```

5. **资源泄漏**

c

运行

```
// 确保关闭所有套接字
close(listenfd);
close(connfd);
```