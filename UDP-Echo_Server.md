# Echo Server - UDP


> ## WSAStartup()
  > - Winsock API를 사용하기 위해 호출하는 함수, Winsock을 초기화 목적으로 사용  
  > - 정상 = 0 반환 이외의 값을 반환하면 오류

    int WSAAPI WSAStartup
    (
        [in]  WORD      wVersionRequested,
        [out] LPWSADATA lpWSAData
    );

wVersionRequested :  Winsock 버전을 지정 보통 2,2 사용  
lpWSAData : WSAStartup 함수에서 반환되는 정보를 저장하는 구조체 <br/><br/><br/>
> ## socket()
 > 소켓을 생성하는 함수, 소켓 = 데이터 송수신을 위한 인터페이스  
 > 정상 = 생성된 소켓 디스크립터 반환, INVALID_SOCKET의 경우 오류
 

    SOCKET WSAAPI socket
    (
        [in] int af,
        [in] int type,
        [in] int protocol
    );

af : 주소체계를 말함, AF_INET = ipv4 / AF_INET6 = ipv6  
type : 소켓 타입을 지정함 SOCK_STREAM은 TCP 소켓, SOCK_DGRAM은 UDP 소켓  
protocol : type에 맞는 프로토콜 사용 TCP = IPPROTO_TPC / UDP = IPPROTO_UDP <br/><br/><br/>

> ## shared_ptr< type > name
 - 여러 개의 포인터가 하나의 객체를 참조할 때 메모리 누수나 잘못된 참조를 방지하는 데 사용
 - 동적으로 할당된 객체를 가리키는 포인터를 관리하며, 해당 객체가 더 이상 필요하지 않을 때 자동으로 메모리를 해제 <br/><br/><br/>


> ## make_shared< class_type >( Constructor )
 - 주로 shared_ptr객체의 생성자 초기화 목적으로 사용 <br/><br/><br/>



> ## bind()
 > socket 과 local address( ip + port )를 결합하는 함수.   
 > - 정상 = 0 반환, 실패 = SOCKET_ERROR  
 > - 일반적으로 로컬 어드레스가 이미 다른 소켓에 할당되어 있거나, 사용 권한이 부족한 경우 실패함

    int bind
    (
    SOCKET         s,
    const sockaddr *name,
    int            namelen
    );
s : 생성한 소켓  
*name : ( sockaddr * )구조체의 포인터, 보통 SOCKADDR_IN 타입으로 정보 넣고 대표타입으로 사용  
namelen : *name가 가르키는것, 즉 sockaddr 구조체의 크기   

> 아래 코드 clasee :: bind() 내부  
  
    int UDPSocket::Bind(const SocketAddress& inBindAddress)
    {
        int error = bind(mSocket, &inBindAddress.mSockAddr, inBindAddress.GetSize());
        if (error != 0)
        {
            cout << "UDPSocket::Bind" << endl;
            return WSAGetLastError();
        }

        return NO_ERROR;
    }

<br/><br/><br/>

> ## recvfrom()
>  - UDP프로토콜을 사용하는 소켓 통신에서 데이터를 받는 함수   
>  - 데이터를 수신하고 송신자의 ip주소와 port번호를 확인 할 수 있음  
*from.sin_addr = 송신자 ip , *from.sin_port = 송신자의 port
>  - 성공 = 수신된 데이터의 길이, 실패 0(정상종료) 또는 SOCKET_ERRPR

    int WSAAPI recvfrom
    (
    [in]                SOCKET   s,
    [out]               char     *buf,
    [in]                int      len,
    [in]                int      flags,
    [out]               sockaddr *from,
    [in, out, optional] int      *fromlen
    );  

s : 수신 받을 소켓 지정  
*buf : 수신 데이터를 저장할 버퍼  
len : 버퍼의 크기  
flags : 소켓옵션 보통0(기본동작)으로 사용  
*from : 송신자의 ip주소와 port를 저장할 구조체 (sockar *) & sockaddr_in타입 형태로 사용가능  
*fromlen : *from 포인터가 가르키는 구조체의 크기 즉 sockaddr_in타입으로 만든 구조체의 크기 sizeof() 사용하면 됨.  <br/> 
> 아래 코드 clasee :: udp_socket_ptr->ReceiveFrom() 내부   <br/> 

    int UDPSocket::ReceiveFrom(void* inToReceive, int inMaxLength, SocketAddress& outFromAddress)
    {
        socklen_t fromLength = outFromAddress.GetSize();

        int readByteCount = recvfrom(mSocket,
            static_cast<char*>(inToReceive),
            inMaxLength,
            0, &outFromAddress.mSockAddr, &fromLength);
        if (readByteCount >= 0)
        {
            return readByteCount;
        }
        else
        {
            int error = WSAGetLastError();

            if (error == WSAEWOULDBLOCK)
            {
                return 0;
            }
            else if (error == WSAECONNRESET)
            {
                return -WSAECONNRESET;
            }
            else
            {
                cout << "UDPSocket::ReceiveFrom" << endl;
                return -error;
            }
        }
    }
<br/><br/><br/>

> ## sendto()
> -  UDP프로토콜을 사용하는 소켓 통신에서 데이터를 보내는 함수  

    int sendto(
    [in] SOCKET         s,
    [in] const char*    buf,
    [in] int            len,
    [in] int            flags,
    [in] const sockaddr*to,
    [in] int            tolen
    );

s : 데이터를 송신할 소켓  
*buf : 송신할 데이터가 저장된 버퍼  
len : 송신할 데이터의 길이  
flags : 옵션 보통 0셋팅  
*to : 송신자의 ip주소와 port를 저장할 구조체 (sockar *) & sockaddr_in타입 형태로 사용가능   
tolen : *to 포인터가 가르키는 구조체의 크기 즉 sockaddr_in타입으로 만든 구조체의 크기 sizeof() 사용하면 됨. <br/>
> 아래 코드 clasee :: udp_socket_ptr->ReceiveFrom() 내부   <br/> 

    int UDPSocket::SendTo(const void* inToSend, int inLength, const SocketAddress& inToAddress)
    {
        int byteSentCount = sendto(mSocket,
            static_cast<const char*>(inToSend),
            inLength,
            0, &inToAddress.mSockAddr, inToAddress.GetSize());
        if (byteSentCount <= 0)
        {
            //we'll return error as negative number to indicate less than requested amount of bytes sent...
            cout<<"UDPSocket::SendTo"<<endl;
            return -WSAGetLastError();
        }
        else
        {
            return byteSentCount;
        }
    }

 <br/><br/>

 # Echo server code

    #pragma comment (lib,"ws2_32.lib")  
    #include "SocketAddress.h"  
    #include "UDPSocket.h"  
    
    void show_error(const char*);

    int main()
    {  
        WSADATA wsa;
        if (WSAStartup(MAKEWORD(2, 2), &wsa)) 
        {
            show_error("WSAStartup()");
            return -1;
        }
        
        SOCKET gate_socket = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
        if (gate_socket == INVALID_SOCKET)
        {
            show_error("create_socket");
            WSACleanup();
            return-1;
        }

        SocketAddressPtr sockaddrptr = make_shared<SocketAddress>(INADDR_ANY, 8000);    // SocketAddress객체 생성
        
        UDPSocketPtr udp_socket_ptr = make_shared<UDPSocket>(gate_socket);              // UDPSocket객체 생성
        
        if (udp_socket_ptr->Bind(*sockaddrptr)) // bind() ip주소와 prot번호를 넘김
        {
            show_error("bind()");
            return -1;
        }
        
        char message_buffer[80];    // 수신받은 데이터가 저장될 공간
        SocketAddress send_info;    // 송신자의 정보가 저장될 공간
        while (true)
        {
            int response = udp_socket_ptr->ReceiveFrom(message_buffer, 80, send_info);
            if (response == 0)
            {
                cout << "Connection with the client has been lost" << endl;
                break;
            }
            else if (response < 0)
            {
                break;
            }

            message_buffer[response] = NULL;    // 수신받은 데이터를 문자열로 변환
            cout << "ip : " << send_info.ToString() <<"\n"<<"port : " << send_info.GetPort() <<"\n"<<"message : "<< message_buffer << endl;

            udp_socket_ptr->SendTo(message_buffer, response, send_info);
        }
        return 0;
    }


    void show_error(const char* error)
    {
        LPVOID error_message = NULL;
        FormatMessage
        (
            FORMAT_MESSAGE_ALLOCATE_BUFFER | FORMAT_MESSAGE_FROM_SYSTEM,
            NULL,
            WSAGetLastError(),
            MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT),
            (LPSTR)&error_message,
            0,
            NULL
        );

        cout << error << " : " << (LPSTR)error_message << endl;
        LocalFree(error_message);
    }
