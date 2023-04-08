# Echo Server - TCP  


> ## listen()
> 소켓을 수신 대기 상태로 만듦, 연결 요청 대기열을 생성
> - TCP 소켓에서 사용함
> - listen함수가 호출되면 소켓이 수신 대기 상태로 변경됨  

    int WSAAPI listen
    (
        [in] SOCKET s,
        [in] int    backlog
    );
s : 수신 대기할 소켓 핸들  
backlog : accept함수 호출전 클라이언트 연결 요청을 저장하는 큐(queue) 일반적으로 SOMAXCOON을 사용
 > if (tcp_socket->Listen() != NO_ERROR) 코드 내부

    int TCPSocket::Listen(int inBackLog)
    {
        int err = listen(mSocket, inBackLog);
        if (err < 0)
        {
            cout << "TCPSocket::Listen" << endl;
            return -WSAGetLastError();
        }
        return NO_ERROR;
    }
inBackLog : 현재 기본값 32 셋팅 

<br/><br/><br/>

> ## aceept()
> 실질적으로 클라이언트와 통신 서비스를 수락하는 함수.
>  - 실제 통신에 사용하는 소켓 디스크립터가 생성됨  
> -> Session_socket  

    SOCKET WSAAPI accept
    (
    [in]      SOCKET   s,
    [out]     sockaddr *addr,
    [in, out] int      *addrlen
    );
s : listen()함수의 대기 큐 에 있는 소켓  
*addr  : 클라이언트의 주소 정보를 저장하기 위한 구조체 sockaddr_IN 타입으로 생성  
*addrlen : addr구조체의 크기

> SocketAddress client_addr_info;  
TCPSocketPtr session_socket = tcp_socket->Accept(client_addr_info); 코드 내부

    TCPSocketPtr TCPSocket::Accept(SocketAddress& inFromAddress)
    {
        socklen_t length = inFromAddress.GetSize();
        SOCKET newSocket = accept(mSocket, &inFromAddress.mSockAddr, &length);

        if (newSocket != INVALID_SOCKET)
        {
            return TCPSocketPtr(new TCPSocket(newSocket));
        }
        else
        {
            cout << "TCPSocket::Accept" << endl;
            return nullptr;
        }
    }
SocketAddress client_addr_info : 클라이언트의 주소 정보가 저장될 변수  
GetSize() == sizeof() 동일한 역할

<br/><br/><br/>

> ## recv()
> TCP소켓을 이용한 통신에서 클라이언트로 부터 데이터를 수신받는 함수.
> - 클라이언트로 부터 0 byte를 수신받을 경우 정상종료임.

    int recv
    (
    [in]  SOCKET s,
    [out] char   *buf,
    [in]  int    len,
    [in]  int    flags
    );
    
s : accept()함수를 통해 생성된 Session_socket  
*buf : 수신 데이터를 저장 할 버퍼  
len : 버퍼의 크기  
flags : 옵션 MSG_OOB, MSG_PEEK, 사용 안할려면 0  
>char buffer[80];  
int recvlen;  
 recvlen = session_socket->Receive(buffer, sizeof(buffer)); 내부코드

    int32_t	TCPSocket::Receive(void* inData, size_t inLen)
    {
        int bytesReceivedCount = recv(mSocket, static_cast<char*>(inData), inLen, 0);
        if (bytesReceivedCount < 0)
        {
            cout<<"TCPSocket::Receive"<<endl;
            return -WSAGetLastError();
        }
        return bytesReceivedCount;
    }
char buffer[80] : 수신받은 데이터가 저장되는 공간   
int recvlen :  수신 데이터의 바이트 크기

<br/><br/><br/>

> ## send()
> TCP 소켓 통신에서 데이터를 송신하는 함수
> - blocking 모드로 호출될때 송신버퍼가 채워 질때까지 대기함
> - 만약 송신 버퍼가 가득 차서 데이터를 전송할 수 없는 경우, 블로킹되어 호출자는 다른 작업을 수행할 수 없음  

    int WSAAPI send
    (
    [in] SOCKET     s,
    [in] const char *buf,
    [in] int        len,
    [in] int        flags
    );
s : 실제 통신하는 Session_socket  
*buf : 전송할 데이터가 저장된 버퍼  
len : 전송할 데이터의 크기  
flags : 옵션, 일반적으로 0 사용  

> session_socket->Send(buffer, recvlen); 코드 내부  

    int32_t	TCPSocket::Send(const void* inData, size_t inLen)
    {
        int bytesSentCount = send(mSocket, static_cast<const char*>(inData), inLen, 0);
        if (bytesSentCount < 0)
        {
            cout << "TCPSocket::Send" << endl;
            return -WSAGetLastError();
        }
        return bytesSentCount;
    }


<br/><br/><br/>

 # TCP - Echo server code
    #pragma comment (lib,"ws2_32.lib")
    #include "SocketAddress.h"
    #include "TCPSocket.h"

    int main()
    {
        WSADATA wsa;
        if (WSAStartup(MAKEWORD(2, 2), &wsa))
        {
            return -1;
        }

        SOCKET gate_socket = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
        if (gate_socket == INVALID_SOCKET)
        {
            return -1;
        }
        
        SocketAddressPtr socket_address = make_shared<SocketAddress>(INADDR_ANY, 8000);

        TCPSocketPtr tcp_socket = make_shared<TCPSocket>(gate_socket);

        if (tcp_socket->Bind(*socket_address) != NO_ERROR)
        {
            return -1;
        }

        if (tcp_socket->Listen() != NO_ERROR)
        {
            return -1;
        }

        SocketAddress client_addr_info;
        TCPSocketPtr session_socket = tcp_socket->Accept(client_addr_info);
        if (session_socket == nullptr)
        {
            return -1;
        }
        else
        {
            cout << "Connected to the client \n\n" << endl;
        }

        char buffer[80];
        int recvlen;

        while (true)
        {
            recvlen = session_socket->Receive(buffer, sizeof(buffer));
            if (recvlen == 0)
            {
                cout << "Connection with the client has been lost" << endl;
                break;
            }
            else if (recvlen < 0)
            {
                break;
            }

            buffer[recvlen] = NULL;
            cout << "[Client IP : " << client_addr_info.ToString() << "  |  " << "Client Port : " << ntohs(client_addr_info.GetPort()) <<"]"<< endl;
            cout << "\nMessage : " << buffer << endl;

            session_socket->Send(buffer, recvlen);
        }

        WSACleanup();

        return 0;
    }

