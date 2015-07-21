第一节 套接字和网络编程
=======================

winsock 编程采用 socket 实现，即套接字。
根据应用场合的不同，分为服务端和客户端编程。

1.1 winsock 服务端代码示例
--------------------------

.. code-block:: C++

    #ifndef UNICODE
    #define UNICODE
    #endif

    #include <winsock2.h>
    #include <stdio.h>
    #include <windows.h>

    // Need to link with Ws2_32.lib
    #pragma comment(lib, "Ws2_32.lib")

    int wmain(void)
    {

        //----------------------
        // Initialize Winsock.
        WSADATA wsaData;
        int iResult = WSAStartup(MAKEWORD(2, 2), &wsaData);
        if (iResult != NO_ERROR) {
            wprintf(L"WSAStartup failed with error: %ld\n", iResult);
            return 1;
        }
        //----------------------
        // Create a SOCKET for listening for
        // incoming connection requests.
        SOCKET ListenSocket;
        ListenSocket = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
        if (ListenSocket == INVALID_SOCKET) {
            wprintf(L"socket failed with error: %ld\n", WSAGetLastError());
            WSACleanup();
            return 1;
        }
        //----------------------
        // The sockaddr_in structure specifies the address family,
        // IP address, and port for the socket that is being bound.
        sockaddr_in service;
        service.sin_family = AF_INET;
        service.sin_addr.s_addr = inet_addr("127.0.0.1");
        service.sin_port = htons(27015);

        if (bind(ListenSocket,
                 (SOCKADDR *) & service, sizeof (service)) == SOCKET_ERROR) {
            wprintf(L"bind failed with error: %ld\n", WSAGetLastError());
            closesocket(ListenSocket);
            WSACleanup();
            return 1;
        }
        //----------------------
        // Listen for incoming connection requests.
        // on the created socket
        if (listen(ListenSocket, 1) == SOCKET_ERROR) {
            wprintf(L"listen failed with error: %ld\n", WSAGetLastError());
            closesocket(ListenSocket);
            WSACleanup();
            return 1;
        }
        //----------------------
        // Create a SOCKET for accepting incoming requests.
        SOCKET AcceptSocket;
        wprintf(L"Waiting for client to connect...\n");

        //----------------------
        // Accept the connection.
        AcceptSocket = accept(ListenSocket, NULL, NULL);
        if (AcceptSocket == INVALID_SOCKET) {
            wprintf(L"accept failed with error: %ld\n", WSAGetLastError());
            closesocket(ListenSocket);
            WSACleanup();
            return 1;
        } else
            wprintf(L"Client connected.\n");

        // No longer need server socket
        closesocket(ListenSocket);

        WSACleanup();
        return 0;
    }

代码解析：

line 1 ~ 3        防止文件重复载入

line 5 ~ 7        头文件加载

line 10           加载静态库
                  
                  * Windows 平台：Ws2_32.lib
                  * WINCE 平台：Ws2.lib

line 17 ~ 21      使用 Windows socket 时，必须先调用 WSAStartup() 函数。
                  该函数的代码实现在 dll 中，作用是初始化网络编程环境。

line 26 ~ 32      定义监听套接字

line 36 ~ 47      bind() 的函数功能是将本地地址和一个套接字绑定。
                  有三个参数：

                  * 参数一是套接字
                  * 参数二为 sockaddr 结构体，描述本地地址
                    
                    注意服务端绑定时进行了强制类型转换，将 sockaddr_in 类型转为 SOCKADDR 类型

                  * 参数三表示参数二的字节数
                  
                  成功则返回 0，否则返回 SOCKET_ERROR；可以通过 WSAGetLastError() 获取错误码

line 51 ~ 56      listen() 函数将 socket 置为监听状态，用于等待即将到来的连接

line 59 ~ 71      通过监听套接字，接收一个连接；这之后就可以根据连接的套接字，收发数据

line 74           关闭套接字

line 76           网络环境清理

为获取客户端发起的连接，需要先创建一个套接字，也就是示例代码中的 ListenSocket。
将套接字绑定到本地地址上，这个过程通过 bind() 函数完成。
但是做完这些仍然不够，还需要将套接字设为监听状态，用于监听连接。
通过 listen() 函数实现。
需要注意的是 listen() 函数的第二个参数 backlog。
backlog 的中文意思表示积压，这里表示连接队列的最大长度。
也就是说在连接到来之后，如果没有立刻取用，则会暂时存在队列里。
队列的长度由 listen() 第二个参数设定；
该参数可以取宏值 SOMAXCONN，表示由底层服务的提供者决定连接队列的最大长度。
在 Windows socket 2 中，这个最大值默认是一个很大的值（一般是几百或者更多）。
在蓝牙应用程序中调用 listen() 函数时，强烈推荐使用小的 backlog（一般是 2~4）。
因为它只能接受几个客户端连接而已。

listen() 函数一般用于同时接受多个连接请求的服务器中。
如果一个连接请求到达之后，队列已满，则客户端会收到一个错误码，SWACONNREFUSED。

如果套接字不可用，则 listen() 会继续执行该函数。
如果套接字可用，则之后调用 listen() 或者 accept() 会重新填充连接队列，重启监听连接。

而如果 listen() 调用时，传入的 socket 已经是监听状态，则会返回成功，且不会改变 backlog 参数的值。
但如果 backlog 设为 0，则操作无效。

.. note:: listen() 函数使用的 socket 是面向连接的，套接字类型为 SOCK_STREAM。
 这个调用是阻塞式的，Winsock 可能需要等待某个网络事件发生之后才会返回。
 该等待过程可以通过相同线程中的异步调用打断。
 如果该异步调用内部还处理了一个阻塞式 Winsock 调用；
 这种做法会导致不确定的后果，因此 winsock 客户端应该绝对禁止的。

调用函数 accept() 获取监听的连接。
accept() 提取连接队列(pending connnections queque)的第一个连接。
创建一个新的 socket 描述符。
新生成的套接字用于解析实际上的连接，它和 ListenSocket 有相同的属性。
包括由函数 WSAAsyncSelect() 或者 WSAEventSelect() 注册的异步网络事件。

如果连接队列里没有连接可用，accept() 会阻塞调用者，并将 ListenSocket 标记为阻塞。
如果该套接字是非阻塞的，且队列里没有连接，则 accept 函数返回错误码。
accept() 成功返回一个套接字描述符之后，该不能接受其他的连接。
而原始的 socket （ListenSocket）依然可以接收新的连接请求。
和 listen() 一样，accept() 调用也可能会需要等到某个网络事件发生才返回。
该等待过程可以通过相同线程下 winsock 异步调用打断。

listen() 成功返回 0，否则返回 SOCKET_ERROR
accept() 成功返回句柄，否则返回 INVALID_SOCKET

1.2 winsock 客户端代码示例
--------------------------

以下例子来自于 MSDN:

.. code-block:: C++

    #ifndef UNICODE
    #define UNICODE
    #endif

    #define WIN32_LEAN_AND_MEAN

    #include <winsock2.h>
    #include <Ws2tcpip.h>
    #include <stdio.h>

    #pragma comment(lib, "Ws2_32.lib")

    #define DEFAULT_BUFLEN 512
    #define DEFAULT_PORT 27015

    int main() {

        //----------------------
        // Declare and initialize variables.
        int iResult;
        WSADATA wsaData;

        SOCKET ConnectSocket = INVALID_SOCKET;
        struct sockaddr_in clientService; 

        int recvbuflen = DEFAULT_BUFLEN;
        char *sendbuf = "Client: sending data test";
        char recvbuf[DEFAULT_BUFLEN] = "";

        //----------------------
        // Initialize Winsock
        iResult = WSAStartup(MAKEWORD(2,2), &wsaData);
        if (iResult != NO_ERROR) {
            wprintf(L"WSAStartup failed with error: %d\n", iResult);
            return 1;
        }

        //----------------------
        // Create a SOCKET for connecting to server
        ConnectSocket = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
        if (ConnectSocket == INVALID_SOCKET) {
            wprintf(L"socket failed with error: %ld\n", WSAGetLastError());
            WSACleanup();
            return 1;
        }

        //----------------------
        // The sockaddr_in structure specifies the address family,
        // IP address, and port of the server to be connected to.
        clientService.sin_family = AF_INET;
        clientService.sin_addr.s_addr = inet_addr( "127.0.0.1" );
        clientService.sin_port = htons( DEFAULT_PORT );

        //----------------------
        // Connect to server.
        iResult = connect( ConnectSocket, (SOCKADDR*) &clientService, sizeof(clientService) );
        if (iResult == SOCKET_ERROR) {
            wprintf(L"connect failed with error: %d\n", WSAGetLastError() );
            closesocket(ConnectSocket);
            WSACleanup();
            return 1;
      }

        //----------------------
        // Send an initial buffer
        iResult = send( ConnectSocket, sendbuf, (int)strlen(sendbuf), 0 );
        if (iResult == SOCKET_ERROR) {
            wprintf(L"send failed with error: %d\n", WSAGetLastError());
            closesocket(ConnectSocket);
            WSACleanup();
            return 1;
        }

        printf("Bytes Sent: %d\n", iResult);

        // shutdown the connection since no more data will be sent
        iResult = shutdown(ConnectSocket, SD_SEND);
        if (iResult == SOCKET_ERROR) {
            wprintf(L"shutdown failed with error: %d\n", WSAGetLastError());
            closesocket(ConnectSocket);
            WSACleanup();
            return 1;
        }

        // Receive until the peer closes the connection
        do {

            iResult = recv(ConnectSocket, recvbuf, recvbuflen, 0);
            if ( iResult > 0 )
                wprintf(L"Bytes received: %d\n", iResult);
            else if ( iResult == 0 )
                wprintf(L"Connection closed\n");
            else
                wprintf(L"recv failed with error: %d\n", WSAGetLastError());

        } while( iResult > 0 );


        // close the socket
        iResult = closesocket(ConnectSocket);
        if (iResult == SOCKET_ERROR) {
            wprintf(L"close failed with error: %d\n", WSAGetLastError());
            WSACleanup();
            return 1;
        }

        WSACleanup();
        return 0;
    }

代码解析：

line 1 ~ 3        C/C++ 编程惯用的技巧，用于防止重复载入。

line 7 ~ 9        头文件加载；需要注意的是网络编程需要加入头文件 winsock.h 或
                  winsock2.h，这两个头文件不能同时出现。

line 11           加载静态库 Ws2_32.lib，这是 Windows 平台下的 winsock 
                  使用的库；WINCE 下的库为 Ws2.lib

line 5, 13 ~ 14   宏定义

line 20 ~ 28      变量定义

line 32 ~ 36      在使用 winsock 的 dll 时必须调用 WSAStartup()
                  接口完成初始化工作。该函数有两个参数，第一个参数是 windows socket 的版本号，值为 0x0202，也就是 2.2；第二个参数是 WSADATA 结构体，用于获取 windows socket 的实现细节。初始化成功返回0，即 NO_ERROR

line 40 ~ 45      实例化一个 SOCKET 对象，如果失败则返回 INVALID_SOCKET
                  socket() 函数有三个参数：

                  * AF_INET 表示 IPv4 地址族；AF_INET6 表示 IPv6 地址族
                  * SOCK_STREAM 套接字类型，表示提供面向连接的流；可以使用 SOCK_DGRAM 表示无连接的报文。
                  * IPPROTO_TCP 网络协议， 对应地址族 AF_INET or AF_INET6 且类型为 SOCK_STREAM；如果是 SOCK_DGRAM，则协议为 IPPROTO_UDP

line 50 ~ 62      连接到服务器，使用 sockaddr_in 结构体指定服务器地址。
                  connect() 的作用是将服务器和特定套接字建立连接。

                  * 参数一，套接字描述符
                  * 参数二，sockaddr_in 结构体指针。
                  
                  sockaddr_in 的成员分别为

                  * sin_family 地址族(u_short)
                  * sin_port 端口号(u_short)，大端 u_short
                  * IP 地址(u_long)，大端 u_long
                  * 保留字段(char[8])。
                  
                  inet_addr() 将字符串变为大端长整型数
                  htons() 将整数变为大端整数（网络序列数NS)

                  * 参数三，参数二的字节数
                  
                 成功返回 0， 否则返回 SOCKET_ERROR。

line 66 ~ 72     发送数据，这个过程是阻塞的，维持到服务器接收完数据为止。

line 77 ~ 83     停止发送或者接收
                 
                 SD_SEND，表示停止发送
                 SD_RECEIVE，表示停止接收
                 SD_BOTH，表示停止发送和接收

line 86 ~ 96     接收来自服务器端的反馈数据。
                
                 recv() 函数的作用是从一个连接的或无连接的 socket 中接收数据。
                 需要注意的是，只要还有数据，则一直可以调用 recv() 函数，直到连接关闭，recv() 函数返回0 为止。调用几次一般受限于传入的 buffer 大小。
                 
line 100 ~ 104   关闭 socket

line 107         通知 windows socket 的 dll 做清理工作

1.3 网络事件
------------

WSAEventSelect()
^^^^^^^^^^^^^^^^

1. 函数原型

.. code-block:: C++

    int WSAEventSelect(
      _In_ SOCKET   s,
      _In_ WSAEVENT hEventObject,
      _In_ long     lNetworkEvents
    );

2. 功能
   
   将事件对象和特定的网络事件集关联起来；
   目的是捕捉发生在指定套接字上的网络事件。
   网络事件是一系列 Winsock 定义的宏，类似于 FD_XXX。

3. 参数说明
   
   +----------------+---------------------------------------+
   | 参数名         | 说明                                  |
   +================+=======================================+
   | s              | 套接字描述符                          |
   +----------------+---------------------------------------+
   | hEventObject   | 事件对象句柄                          |
   +----------------+---------------------------------------+
   | lNetworkEvents | 网络事件集（异或）                    |
   +----------------+---------------------------------------+
   | 返回值         | 0 表示成功，SOCKET_ERROR 表示失败。   |
   |                | 可以通过 SWAGetLastError() 获取错误码 |
   +----------------+---------------------------------------+

4. 细节描述
   
   WSAEventSelect() 常用在判断数据传输何时能够立即成功。
   一个健壮的应用程序在设置了事件对象之后，能够处理 winsock 函数调用产生的 WSAWOULDBLOCK 错误。
   比如，依照发生的前后次序，可能会遇到这样的情况：

   * 数据到达套接字 s；winsock 设置事件对象；
   * 程序处理其他事物
   * 在处理事务过程中，程序通过 ioctlsocket 注意到了数据的到达
   * 程序执行 recv() 接收数据
   * 程序等待事件对象（WSAEventSelect() 返回表明数据准备好，事件对象变为有信号，程序结束等待）
   * 程序执行 recv() 失败，返回错误 WSAWOULDBLOCK

   Ws2_32.dll 将发生的网络事件记录（通过网络事件比特位设置）并向对应对象发信号之后，不再有其他动作。
   也就是说事件记录会一直保留，且对应的对象收到信号后也会持续不变。
   一直持续到程序调用某函数重新触发了新的网络事件并重新向事件对象发信号为止。
   这种函数称为重置函数
   以下列举了不同网络事件的重置函数：

   +-----------------------------+-------------------------------------------------+----------+
   | 网络事件                    | 重置函数                                        | 触发方式 |
   +=============================+=================================================+==========+
   | FD_READ                     | recv, recvfrom, WSARecv, WSARecvEx, WSARevcFrom | 条件触发 |
   +-----------------------------+-------------------------------------------------+----------+
   | FD_WRITE                    | send, sendto, WSASend, WSASendTo                | [1]_     |
   +-----------------------------+-------------------------------------------------+----------+
   | FD_OOB                      | recv, recvfrom, WSARecv, WSARecvEx, WSARecvFrom | 条件触发 |
   +-----------------------------+-------------------------------------------------+----------+
   | FD_ACCEPT                   | accept, AcceptEx, WSAAccept                     | 条件触发 |
   +-----------------------------+-------------------------------------------------+----------+
   | FD_CONNECT                  | None                                            |          |
   +-----------------------------+-------------------------------------------------+----------+
   | FD_CLOSE                    | None                                            |          |
   +-----------------------------+-------------------------------------------------+----------+
   | FD_QOS                      | WSAIoctl(SIO_GET_QOS)                           | 边沿触发 |
   +-----------------------------+-------------------------------------------------+----------+
   | FD_GROUP_QOS                | 保留                                            | 边沿触发 |
   +-----------------------------+-------------------------------------------------+----------+
   | FD_ROUTING_INTERFACE_CHANGE | WSAIoctl(SIO_ROUTING_INTERFACE_CHANGE)          | 边沿触发 |
   +-----------------------------+-------------------------------------------------+----------+
   | FD_ADDRESS_LIST_CHANGE      | WSAIoctl(SIO_ADDRESS_LIST_CHANGE)               | 边沿触发 |
   +-----------------------------+-------------------------------------------------+----------+

   .. note:: 程序收到 FD_CLOSE 事件时，应该检验数据的完整性，以避免数据的丢失。

5. 后续操作
   
   在设定事件对象之后，将事件对象传给 WSAEnumNetworkEvents() 枚举所有发生的网络事件。
   需要注意的是，这个过程会重置事件对象状态和事件记录，该过程是原子的。

6. 条件触发
   
   当套接字 s 收到数据块时，会引起 Ws2_32.dll 记录事件 FD_READ 并向对应事件对象发送信号。
   而程序调用 recv() 函数之后，如果还有数据，即网络条件还有效，则会再次触发 FD_READ 事件。

7. 时效性
   
   调用 WSAEventSelect() 或者重置函数时，网络事件已经发生，则网络事件依然会被记录，且会发信号给相关事件对象。
   也就是说网络事件不是瞬时的，而是有一定的实效性的。

8. 不叠加
   
   WSAEventSelect() 调用没有叠加性，后一次调用会覆盖前一次

9. 清理
    
   WSACloseEvent()
10. 代码示例
    
    摘自 MSDN:

    .. code-block:: C++
    
        #ifndef UNICODE
        #define UNICODE
        #endif

        #define WIN32_LEAN_AND_MEAN

        #include <windows.h>

        #include <winsock2.h>
        #include <Ws2tcpip.h>
        #include <stdio.h>

        // Link with ws2_32.lib
        #pragma comment(lib, "Ws2_32.lib")

        int main()
        {
        //-------------------------
        // Declare and initialize variables
            WSADATA wsaData;
            int iResult;

            SOCKET SocketArray[WSA_MAXIMUM_WAIT_EVENTS], ListenSocket;
            WSAEVENT EventArray[WSA_MAXIMUM_WAIT_EVENTS];
            WSANETWORKEVENTS NetworkEvents;
            sockaddr_in InetAddr;
            DWORD EventTotal = 0;
            DWORD Index;
            DWORD i;
            
            HANDLE NewEvent = NULL; 

            // Initialize Winsock
            iResult = WSAStartup(MAKEWORD(2, 2), &wsaData);
            if (iResult != 0) {
                wprintf(L"WSAStartup failed with error: %d\n", iResult);
                return 1;
            }

        //-------------------------
        // Create a listening socket
            ListenSocket = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
            if (ListenSocket == INVALID_SOCKET) {
                wprintf(L"socket function failed with error: %d\n", WSAGetLastError() );
                return 1;
            }
            
            InetAddr.sin_family = AF_INET;
            InetAddr.sin_addr.s_addr = htonl(INADDR_ANY);
            InetAddr.sin_port = htons(27015);

        //-------------------------
        // Bind the listening socket
            iResult = bind(ListenSocket, (SOCKADDR *) & InetAddr, sizeof (InetAddr));
            if (iResult != 0) {
                wprintf(L"bind failed with error: %d\n", WSAGetLastError() );
                return 1;
            }

        //-------------------------
        // Create a new event
            NewEvent = WSACreateEvent();
            if (NewEvent == NULL) {
                wprintf(L"WSACreateEvent failed with error: %d\n", GetLastError() );
                return 1;
            }    

        //-------------------------
        // Associate event types FD_ACCEPT and FD_CLOSE
        // with the listening socket and NewEvent
            iResult = WSAEventSelect(ListenSocket, NewEvent, FD_ACCEPT | FD_CLOSE);
            if (iResult != 0) {
                wprintf(L"WSAEventSelect failed with error: %d\n", WSAGetLastError() );
                return 1;
            }

        //-------------------------
        // Start listening on the socket
            iResult = listen(ListenSocket, 10);
            if (iResult != 0) {
                wprintf(L"listen failed with error: %d\n", WSAGetLastError() );
                return 1;
            }

        //-------------------------
        // Add the socket and event to the arrays, increment number of events
            SocketArray[EventTotal] = ListenSocket;
            EventArray[EventTotal] = NewEvent;
            EventTotal++;

        //-------------------------
        // Wait for network events on all sockets
            Index = WSAWaitForMultipleEvents(EventTotal, EventArray, FALSE, WSA_INFINITE, FALSE);
            Index = Index - WSA_WAIT_EVENT_0;

        //-------------------------
        // Iterate through all events and enumerate
        // if the wait does not fail.
            for (i = Index; i < EventTotal; i++) {
                Index = WSAWaitForMultipleEvents(1, &EventArray[i], TRUE, 1000, FALSE);
                if ((Index != WSA_WAIT_FAILED) && (Index != WSA_WAIT_TIMEOUT)) {
                    WSAEnumNetworkEvents(SocketArray[i], EventArray[i], &NetworkEvents);
                }
            }

        //...
            return 0;

.. [1] FD_WRITE 的触发方式不太相同.
 首次调用 connect 系列函数，会触发 FD_WRITE 事件（客户端）;
 成功调用 accept 系列函数也会触发 FD_WRITE 事件（服务端）。
 从首次接收到 FD_WRITE 事件到发生 WSAWOULDBLOCK 错误之时，这段时间可以发送数据。
 在收到该错误之后，只有当 FD_WRITE 事件再次产生之时，才能继续发数据。