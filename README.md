# TCP-IP
기말
201744023_신진호 3-A TCP/IP 기말고사

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <pthread.h>

#define BUF_SIZE 100
#define MAX_CLNT 256

void * handle_clnt(void * arg);
void send_msg(char * msg);
void error_handlig(char * msg);

int clnt_cnt=0;
int clnt_socks[MAX_CLNT];
pthread_mutex_t mutx;

int main(int argc, char *argv[])
{
       int serv_sock, clnt_sock;
       struct sockaddr_in serv_adr, clnt_adr;
       int clnt_adr_sz;
       pthread_t t_id;
       if(argc!=2){
                printf("Usage : %s <port>\n", argv[0]);
                exit(1);
       }
       
       pthread_mutex_init(&mutx, NULL);
       serv_sock+socket(PF_INET, SOCK_STREAM, 0);
       memset(&serv_adr, 0, sizeof(serv_adr));
       serv_adr.sin_family=AF_INET;
       serv_adr.sin_addr.s_addr=hton1(INADDR_ANY);
       serv_adr.sin_port=htans(atoi(argv[1]));
  
       if(bind(serv_sock, (struct sockaddr*) &serv_adr
