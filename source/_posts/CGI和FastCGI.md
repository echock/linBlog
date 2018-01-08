---
title: CGI和FastCGI
---


# CGI简介
全称是“通用网关接口”，它描述了客户端和程序之间传输数据的一种标准。
## CGI的运行原理
1. 客户端访问某个 URL 地址之后，通过 GET/POST/PUT 等方式提交数据，并通过 HTTP 协议向 Web 服务器发出请求。
2. 服务器端的 HTTP Daemon（守护进程）启动一个子进程。然后在子进程中，将 HTTP 请求里描述的信息通过标准输入 stdin 和环境变量传递给 URL 指定的 CGI 程序，并启动此应用程序进行处理，处理结果通过标准输出 stdout 返回给 HTTP Daemon 子进程。
3. 再由 HTTP Daemon 子进程通过 HTTP 协议返回给客户端。

下面以一次请求为例
1. 客户端访问 http://127.0.0.1:9003/cgi-bin/user?id=1
2. 127.0.0.1 上监听 9003 端口的守护进程接受到该请求
3. 通过解析 HTTP 头信息，得知是 GET 请求，并且请求的是 /cgi-bin/ 目录下的 user 文件。
4. 将 uri 里的 id=1 通过存入 QUERY_STRING 环境变量。
5. Web 守护进程 fork 一个子进程，然后在子进程中执行 user 程序，通过环境变量获取到id。
6. 执行完毕之后，将结果通过标准输出返回到子进程。
7. 子进程将结果返回给客户端。

---
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <string.h>
 
#define SERV_PORT 9003
 
char *str_join(char *str1, char *str2);
 
char *html_response(char *res, char *buf);
 
int main(void) {
    int lfd, cfd;
    struct sockaddr_in serv_addr, clin_addr;
    socklen_t clin_len;
    char buf[1024], web_result[1024];
    int len;
    FILE *cin;
 
    if ((lfd = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
        perror("create socket failed");
        exit(1);
    }
 
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_addr.sin_port = htons(SERV_PORT);
 
    if (bind(lfd, (struct sockaddr *) &serv_addr, sizeof(serv_addr)) == -1) {
        perror("bind error");
        exit(1);
    }
 
    if (listen(lfd, 128) == -1) {
        perror("listen error");
        exit(1);
    }
 
    signal(SIGCLD, SIG_IGN);
 
    while (1) {
        clin_len = sizeof(clin_addr);
        if ((cfd = accept(lfd, (struct sockaddr *) &clin_addr, &clin_len)) == -1) {
            perror("接收错误\n");
            continue;
        }
 
        cin = fdopen(cfd, "r");
        setbuf(cin, (char *) 0);
        fgets(buf, 1024, cin); //读取第一行
        printf("\n%s", buf);
 
        //============================ cgi 环境变量设置演示 ============================
 
        // 例如 "GET /cgi-bin/user?id=1 HTTP/1.1";
 
        char *delim = " ";
        char *p;
        char *method, *filename, *query_string;
        char *query_string_pre = "QUERY_STRING=";
 
        method = strtok(buf, delim);         // GET
        p = strtok(NULL, delim);             // /cgi-bin/user?id=1 
        filename = strtok(p, "?");           // /cgi-bin/user
 
        if (strcmp(filename, "/favicon.ico") == 0) {
            continue;
        }
 
        query_string = strtok(NULL, "?");    // id=1
        putenv(str_join(query_string_pre, query_string));
 
        //============================ cgi 环境变量设置演示 ============================
 
        int pid = fork();
 
        if (pid > 0) {
            close(cfd);
        }
        else if (pid == 0) {
            close(lfd);
            FILE *stream = popen(str_join(".", filename), "r");
            fread(buf, sizeof(char), sizeof(buf), stream);
            html_response(web_result, buf);
            write(cfd, web_result, sizeof(web_result));
            pclose(stream);
            close(cfd);
            exit(0);
        }
        else {
            perror("fork error");
            exit(1);
        }
    }
 
    close(lfd);
 
    return 0;
}
 
char *str_join(char *str1, char *str2) {
    char *result = malloc(strlen(str1) + strlen(str2) + 1);
    if (result == NULL) exit(1);
    strcpy(result, str1);
    strcat(result, str2);
 
    return result;
}
 
char *html_response(char *res, char *buf) {
    char *html_response_template = "HTTP/1.1 200 OK\r\nContent-Type:text/html\r\nContent-Length: %d\r\nServer: mengkang\r\n\r\n%s";
 
    sprintf(res, html_response_template, strlen(buf), buf);
 
    return res;
}
---

# FastCGI 简介
FastCGI是web服务器和处理程序之间通信的一种协议，是CGI的一种改进方案，它常驻进程，可以一直执行，在请求达到时不会话费时间去fork一个进程来处理，将CGI解释器进程保存在内存中并接受FASTCGI进程管理器调度，可以提供好的性能。
## 工作流程
1. FastCGI 进程管理器自身初始化，启动多个 CGI 解释器进程，并等待来自 Web Server 的连接。
2. Web 服务器与 FastCGI 进程管理器进行 Socket 通信，通过 FastCGI 协议发送 CGI 环境变量和标准输入数据给 CGI 解释器进程。
3. CGI 解释器进程完成处理后将标准输出和错误信息从同一连接返回 Web Server。
4. CGI 解释器进程接着等待并处理来自 Web Server 的下一个连接。

## 比较
FastCGI 与传统 CGI 模式的区别之一则是 Web 服务器不是直接执行 CGI 程序了，而是通过 Socket 与 FastCGI 响应器（FastCGI 进程管理器）进行交互，也正是由于 FastCGI 进程管理器是基于 Socket 通信的，所以也是分布式的，Web 服务器可以和 CGI 响应器服务器分开部署。Web 服务器需要将数据 CGI/1.1 的规范封装在遵循 FastCGI 协议包中发送给 FastCGI 响应器程序。






