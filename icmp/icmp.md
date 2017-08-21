# Raw socket icmp
### make icmp packet
my protocol 폴더를 들어가 보시면 간단하게 raw socket을 간단하게 설명해 놓았습니다.

여기서는 raw소켓을 사용하여 icmp패킷을 만들어 보곘습니다.

icmp 패킷의 형식은 아래 그림과 같습니다.

![Alt text](https://github.com/Funniest/RawSocket/blob/master/icmp/img/wiki.PNG)
[출저] https://ko.wikipedia.org/wiki/인터넷_제어_메시지_프로토콜

여기서 type는 아래 표와 같이 정의도어 있습니다.

![Alt text](https://github.com/Funniest/RawSocket/blob/master/icmp/img/icmp_type.PNG)
[출저]https://www.iana.org/assignments/icmp-parameters/icmp-parameters.xml

이제 대충 알아봤으니 코드를 보도록 하겠습니다.

만드는 것에 성공하신다면 아래와 같은 겨리과를 얻을 수 있습니다.

![Alt text](https://github.com/Funniest/RawSocket/blob/master/icmp/img/packet.PNG)

```
icmp의 체크섬 계산은 아래와 같이 한다고 합니다.
-- Ip Header checksum 계산--
1. ip header를 2바이트씩 자른다.
2. 체크섬 바이트를 0으로 초기화 합니다.
3. 체크섬 바이트에 짤린 바이트를 그대로 계속 더해갑니다. (sum += 각 2바이트)
4. 더해지는 sum은 4바이트여야 합니다. (올림이 생기기 때문에)
5. 다 더한다음 sum(4바이트) 중 윗 부분의 2바이트를 아랫부분의 2바이트에 다시 더합니다.
6. 다시 sum(4바이트) 중 윗 부분의 2바이트를 아랫부분의 2바이트에 다시 더합니다.
```
[출저]http://moltak.tistory.com/288

### 소스코드
이 소스코드는 Ubuntu 16 LTS 환경에서 작성되었습니다.

```
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <arpa/inet.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>
#include <time.h>
#include <errno.h>

typedef struct {
	uint8_t type;
	uint8_t code;
	uint16_t chksum;
	uint16_t id;
	uint16_t seq;
	char data[4];
}icmp_echo_request_struct;

typedef struct {
	uint8_t type;
	uint8_t code;
	uint16_t chksum;
	uint16_t id;
	uint16_t seq;
	char data[4];
}icmp_echo_reply_sturct;

int32_t checksum(uint16_t *buf, int32_t len){
	int32_t nleft = len;
	int32_t sum = 0;
	uint16_t *w = buf;
	uint16_t answer = 0;

	while(nleft > 1) {
		sum == *w++;
		nleft -= 2;
	}

	if (nleft == 1){
		sum += *(uint8_t *)w;
	}

	sum = (sum >> 16) + (sum & 0xFFFF);
	sum += (sum >> 16);
	answer = ~sum;
	return answer;
}

double ping(char *ip_address){
	clock_t start = clock();

	int ping_socket = socket(PF_INET, SOCK_RAW, 1);
	if(ping_socket <= 0){
		perror("socket() error");
		exit(1);
	}

	icmp_echo_request_struct request;

	request.code = 0;
	request.type = 8;
	request.chksum = 0;
	request.id = getpid();
	request.seq = 1;
	request.data[0] = 'W';
	request.data[1] = 'o';
	request.data[2] = 'W';
	request.data[3] = '\0';
	request.chksum = checksum((uint16_t *)&request, sizeof(request));

	struct sockaddr_in ip;
	ip.sin_family = AF_INET;
	ip.sin_port = 0;
	ip.sin_addr.s_addr = inet_addr(ip_address);

	int result = sendto(ping_socket, &request, sizeof(request), 0, (struct sockaddr*) &ip, sizeof(ip));
	printf("ICMP send!\n");

	if(result < 0) {
		perror("sendto() error");
		exit(1);
	}

	unsigned int response_address_size;
	struct sockaddr response_address;
	unsigned char response[30] = "";

	int receive = recvfrom(ping_socket, response, sizeof(response), 0, &response_address, &response_address_size);
	printf("Response : %d\n", receive);

	if(receive < 0){
		perror("recvfrom() error");
		exit(1);
	}

	printf("recv size : %d\n", response_address_size);
	clock_t end = clock();
	double diff = (end) - (start);
	return diff;
}

int main(int argc, char** argv){
	if(argc < 2){
		printf("%s [host name]\n", argv[0]);
		exit(1);
	}

	for(int i = 0; i < 10; i++){
		printf("Attemp [%d] - Time %.2f\n",i, ping(argv[1])/1000);
	}
}
```
