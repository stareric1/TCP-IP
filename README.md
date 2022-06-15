# TCP-IP
기말
201744023_신진호 3-A TCP/IP 기말고사

<<< chat_server.c >>>
#include <stdio.h>       //표준입출력
#include <stdlib.h>      //표준 라이브러리
#include <unistd.h>      //유닉스 라이브러리
#include <string.h>      //문자열 처리
#include <arpa/inet.h>   //인터넷 프로토콜
#include <sys/socket.h>  //소켓함수
#include <netinet/in.h>  //인터넷 주소 체계
#include <pthread.h>     //쓰레드 구현하기 위해

#define BUF_SIZE 100      //채팅할때 메세지 최대길이
#define MAX_CLNT 256      //최대 동시 접속자 수

void * handle_clnt(void * arg);  //클라이언트 쓰레드용 함수(함수포인터)
void send_msg(char * msg);       //메세지 전달 함수
void error_handlig(char * msg);  //예외 처리 함수

int clnt_cnt=0;               //현제 접속중인 사용자 수
int clnt_socks[MAX_CLNT];     //메세지 전달 함수
pthread_mutex_t mutx;         //예외 처리 함수

int main(int argc, char *argv[])   // 인자로 포트번호를 받음
{
       int serv_sock, clnt_sock;               //소켓 통신용 서버 소켓과 임시 클라이언트 소켓
       struct sockaddr_in serv_adr, clnt_adr;  //서버 주소, 클라이언트 주소 구조체
       int clnt_adr_sz;                        //클라이언트 주소 구조체
       pthread_t t_id;                         //클라이언트 쓰레드용 ID
       // 포트 입력 안했으면
       if(argc!=2){
                printf("Usage : %s <port>\n", argv[0]);   //사용법을 알려준다
                exit(1);                                  //프로그램 비정상 종료
       }
       
       pthread_mutex_init(&mutx, NULL);                   //커널에서 Mutex쓰기위해 얻어 온다.
       serv_sock+socket(PF_INET, SOCK_STREAM, 0);         //TCP용 소켓 생성
       memset(&serv_adr, 0, sizeof(serv_adr));            //서버 주소 구조체 초기화
       serv_adr.sin_family=AF_INET;                       //인터넷 통신
       serv_adr.sin_addr.s_addr=hton1(INADDR_ANY);        //현제 IP를 이용하고
       serv_adr.sin_port=htans(atoi(argv[1]));            //포트는 사용자가 지정한 포트사용
  
       //서버 소켓에 주소를 할당한다
       if(bind(serv_sock, (struct sockaddr*) &serv_adr, sizeof(serv_adr))==-1)
             error_handling("bind() error");
       
       //서버 소켓을 서버로 사용
       if(listen(serv_sock, 5)==-1)
             error_handling("listen() error");
       
       while(1) //무한루프 돌면서
       {
             clnt_adr_sz=sizeof(clnt_adr);    // 클라이언트 구조체의 크기를 얻고
             
             //클라이언트의 접속을 받아들이기 위해 Block 된다.(멈춘다)
             clnt_sock=accept(serv_sock, (struct sockaddr*)&clnt_adr,&clnt_adr_sz);
             
             pthread_mutex_lock(&mutx);
             clnt_socks[clnt_cnt++]=clnt_sock;
             pthread_mutex_unlock(&mutx);
       
             // 클라이언트 구조체의 주소를 쓰레드에 넘긴다.(포트 포함됨)
             pthread_create(&t_id, NULL, handle_clnt, (void*)&clnt_sock);    //쓰레드 시작
             pthread_detach(t_id);                                           //쓰레드가 종료되면 스스로 소멸되게 함
             
             //접속된 클라이언트 IP를 화면에 찍어준다.
             printf("Connected client IP: %s \n", inet_ntoa(clnt_adr.sin_addr));
       }
       close(serv_sock);    // 쓰레드가 끝나면 소켓을 닫아준다.
       return 0;
}
// 쓰레드용 함수
void * handle_clnt(void * arg)   // 소켓을 들고 클라이언트와 통신하는 함수
{
       int clnt_sock=*((int*)arg);   // 쓰레드가 통신할 클라이언트 소켓 변수
       int str_len=0, i;
       char msg[BUF_SIZE];           // 메시지 버퍼
       
       // 클라이언트가 통신을 끊지 않느한 계속 서비스를 제공한다.
       while((str_len=read(clnt_sock, msg, sizeof(msg)))!=0)
              send_msg(msg, str_len);
       
       pthread_mutex_lock(&mutx);       // 임계 영역 시작
       for(i=0; i<clnt_cnt; i++) // remove disconnected client
{
                 if(clnt_sock==clnt_socks[i])
                 {
                         while(i++<clnt_cnt-1)
                                 clnt_socks[i]=clnt_socks[i+1];
                         break;
                 }
       }
       clnt_cnt--;
       pthread_mutex_unlock(&mutx);    // 임계 영역 끝
       close(clnt_sock);               // 소켓을 닫고
       return NULL;                    // 서비스 종료
}
void send_msg(char * msg, int len)  //send to all
{
       int i;
       pthread_mutex_lock(&mutx);
       for(i=0; i<clnt_cnt; i++)
                 write(clnt_socks[i], msg, len);
       pthread_mutex_unlock(&mutx);
}
// 예외 처리 함수
void error_handling(char *msg)
{
       fputs(msg, stderr);
       fputc('\n', stderr);
       exit(1);
}
                            
                            
    <<< chat_client.c >>>
#include <stdio.h>	// 표준 입출력
#include <stdlib.h>	// 표준 라이브러리
#include <unistd.h>	// 유닉스 표준
#include <string.h>	// 문자열 처리
#include <arpa/inet.h>	// 인터넷 프로토콜
#include <sys/socket.h>	// 소켓함수
#include <pthread.h>	
#define BUF_SIZE 100		// 메시지 버퍼 길이
#define NAME_SIZE 20		// 이름의 길이	
       
void * send_msg(void * arg);	// 쓰레드 함수 - 송신용
void * recv_msg(void * arg);	// 쓰레드 함수 - 수신용
void error_handling(char * msg);	// 예외처리 함수	
       
char name[NAME_SIZE]="[DEFAULT]";	// 방 이름 - 초기값은 "[DEFAULT]"
char msg[BUF_SIZE];			// 메시지 버퍼
       
// 인자로 IP와 port 를 받는다
int main(int argc, char *argv[])
{	
       int sock;				// 통신용 소켓 파일 디스크립터	
       struct sockaddr_in serv_addr;	// 서버 주소 구조체 변수	
       pthread_t snd_thread, rcv_thread;	// 송신 쓰레드, 수신 쓰레드	
       void * thread_return;			// 쓰레드 반환 갑인가	
       if(argc!=4)		// 사용자가 프로그램 잘 못 시작시켰으면	
       {			
              printf("Usage : %s <IP> <port> <name>\n", argv[0]); // 사용법 안내		
              exit(1);						// 프로그램 비정상 종료	 
       }
       
       sprintf(name, "[%s]", argv[3]);			// 사용자의 이름	
       sock=socket(PF_INET, SOCK_STREAM, 0);	// TCP 통신 소켓 설정		
       
       memset(&serv_addr, 0, sizeof(serv_addr));	// 접속할 서버 주소 설정	
       serv_addr.sin_family=AF_INET;			// 인터넷 통신	
       serv_addr.sin_addr.s_addr=inet_addr(argv[1]);		
       serv_addr.sin_port=htons(atoi(argv[2]));	// 빅엔디안을 리틀엔디안으로	 	
      
       
       // 서버에 접속 시도	
       if(connect(sock, (struct sockaddr*)&serv_addr, sizeof(serv_addr))==-1)		
              error_handling("connect() error");		
       
       pthread_create(&snd_thread, NULL, send_msg, (void*)&sock);	// 송신 쓰레드 시작	
       pthread_create(&rcv_thread, NULL, recv_msg, (void*)&sock);	//  수신 쓰레드 시작
       pthread_join(snd_thread, &thread_return);	// 메인 함수는 아무것도 안하고 기다린다.	
       pthread_join(rcv_thread, &thread_return);	// 통신은 2개의 쓰레드가 처리한다.	
       close(sock);  		// 프로그램 종료할 때 소켓 소멸 시키고 종료	
       return 0;
}	

void * send_msg(void * arg)   // send thread main
{	
       int sock=*((int*)arg);		// 소켓을 받는다.	
       char name_msg[NAME_SIZE+BUF_SIZE];		
       while(1) 		// 무한 루프 돌면서	
       {		
              fgets(msg, BUF_SIZE, stdin);		// 키보드 입력을 받아서		
              if(!strcmp(msg,"q\n")||!strcmp(msg,"Q\n")) 	// q나 Q면 종료하고		
              {			
                    close(sock);			
                    exit(0);		
              }		
              // 정상 적인 입력을 하면		
              sprintf(name_msg,"%s %s", name, msg);	// 이름 + 공백 + 메시지 순으로 전송		
              write(sock, name_msg, strlen(name_msg));	// 서버로 메시지 보내기	
       }	
       return NULL;
}
// 수신쪽 쓰레드 함수
void * recv_msg(void * arg)   // read thread main
{	
       int sock=*((int*)arg);		// 통신용 소켓을 받고	
       char name_msg[NAME_SIZE+BUF_SIZE];	// 이름 + 메시지 버퍼	
       int str_len;				// 문자열 길이		
       while(1)				// 무한 루프 돌면서	
       {		
               // 메시지가 들어오면		
               str_len=read(sock, name_msg, NAME_SIZE+BUF_SIZE-1);		
               if(str_len==-1) 		// 만약 통신이 끊겼으면			
                      return (void*)-1;	// 쓰레드 종료		
               name_msg[str_len]=0;	// 메시지끝을 설정		
               fputs(name_msg, stdout);	// 화면에 수신된 메시지 표시	
       }	
       return NULL;
}
// 예외 처리 함수
void error_handling(char *msg)
{	
       fputs(msg, stderr);	// 오류 메시지를 표시하고	
       fputc('\n', stderr);	
       exit(1);	       // 비정상 종료 처리
}
