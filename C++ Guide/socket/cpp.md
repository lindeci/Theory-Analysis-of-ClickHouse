- [c++ Scoket编程](#c-scoket编程)
  - [一、socket编程流程](#一socket编程流程)
  - [二、简单的服务端](#二简单的服务端)
  - [三、简单客户端例子](#三简单客户端例子)

# c++ Scoket编程

## 一、socket编程流程
**1.** 创建socket套接字  
**2.** 定义服务端地址和端口  
**3.** 服务端绑定地址和端口  
**4.** 服务端进行端口监听  
**5.** 服务端与客户端进行连接  
**6.** 服务端接收客户端发送的信息  
**7.** 服务端返回信息给客户端  
**8.** 关闭套接字

## 二、简单的服务端
``` cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <netinet/in.h>
#include <sys/types.h>

#include <errno.h>


int
main(void){
        char send_message[255] = "hello i am server";
        int server_sock = socket(AF_INET, SOCK_STREAM, 0);

        // address
        struct sockaddr_in server_addr;
        server_addr.sin_addr.s_addr = INADDR_ANY;
        server_addr.sin_port = htons(9001);
        server_addr.sin_family = AF_INET; 

        //bind
        if ( (bind(server_sock, (struct sockaddr*) &server_addr, sizeof(server_addr))) != 0 ){
                fprintf(stderr, "couldn't bind: %s\n", strerror(errno));
                exit(EXIT_FAILURE);
        }

        //listen
        if ((listen(server_sock, 10)) != 0){
                fprintf(stderr, "couldn't listen: %s\n", strerror(errno));
                exit(EXIT_FAILURE);
        }

        int client_sock;
        //accept
        if( (client_sock = accept(server_sock, NULL, NULL)) == -1 ){
                fprintf(stderr, "couldn't accept: %s\n", strerror(errno));
                exit(EXIT_FAILURE);
        }

        send(client_sock, send_message, sizeof(send_message), 0);

        close(server_sock);
        close(client_sock);
}
```
## 三、简单客户端例子
``` cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <sys/types.h>
#include <sys/socket.h>
#include <unistd.h>

#include <netinet/in.h>

#include <errno.h>

int main(void){
        //create sock
        int sock = socket(AF_INET, SOCK_STREAM, 0);
        if(sock < 0){
                fprintf(stderr, "can't create socket because of %s\n", strerror(errno));
                exit(EXIT_FAILURE);
        }

        //create address
        struct sockaddr_in server_addr;
        server_addr.sin_addr.s_addr = INADDR_ANY;
        server_addr.sin_port = htons(9001);
        server_addr.sin_family = AF_INET;

        //connect
        if( (connect(sock, (struct sockaddr*)&server_addr, sizeof(server_addr))) == -1  ){
                perror("connect error because of: ");
                close(sock);
                exit(EXIT_FAILURE);

        }

        //create recv buffer and recv
        char recv_buffer[255];
        recv(sock, recv_buffer, sizeof(recv_buffer), 0);

        //print
        printf("printf recv: %s\n", recv_buffer);

        close(sock);
        return 0;
}
```