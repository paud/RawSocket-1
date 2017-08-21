# Raw socket
### What is raw socket?
로우 소켓이란, 소켓의 데이터 하나하나를 조작할 수 있게 만드는 소켓입니다.

좀 더 자세하게 설명하자면, 새로운 프로토콜을 만들거나 패킷을 세밀하게 조작할 때 주로 사용됩니다.

예를 들어 TCP/UDP Socket을 사용하면, 만들어진 틀에 목적지와 데이터만 실어 보내면 됩니다.

하지만 로우소켓으로 TCP Packet을 보낼때에는 목적지와 데이터 뿐 아니라 틀도 정의를 전부 해주어야 합니다.

### Fuctions
로우소켓을 사용할 때 필요한 함수들에 대해서 알아보겠습니다.

socket(AF_INET, SOCK_RAW, protocol)입니다.

socket의 중간 인자는 ***SOCK_RAW***를 주어야 합니다.

protocol은 <netinet/net.h>를 포함한 정의도니 IPPROTO_ICMP같은 형식의 상수중의 하나여야 한다고 합니다.

다음은 setsockopt인데, 사실 이건 상황에 따라 써두되고 안써도 되는 것 같습니다.

마지막으로는 sendto입니다.

만들어진 raw socket을 보낼떄 sendto함수를 사용해서 보낼 수 있습니다.

### Make

간단하게 커스텀 소켓을 만들어 보겠습니다.

- 프로토콜 형식은 data length, data를 가집니다.
- 프로토콜의 넘버는 0x88b5d입니다.

아래 그림과 같은 결과를 얻을 수 있습니다.

[!Alt](https://github.com/Funniest/RawSocket/blob/master/img/packet.png)

### 소스코드
소스코드는 Ubuntu 16 LTS환경에서 작성되어씁니다.

```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <sys/ioctl.h>
#include <netinet/in.h>
#include <netinet/ip.h>
#include <net/if.h>
#include <net/ethernet.h>
#include <arpa/inet.h>
#include <linux/if_ether.h>
#include <linux/if_packet.h>
#include <bits/ioctls.h>
#include <errno.h>

typedef struct _custom_protocol{
	uint16_t cmd_number;
	uint16_t data_len;
	uint8_t data[4];
}custom_protocol;

#define CUSTOM_PROTOCOL_LEN 4
#define ETH_HDRLEN 12

void *allocate_strmem(int, char);

int main(int argc, char* argv[]){
	int fd, frame_length, bytes;
	char *interface, *target, *src_ip;
	uint8_t *src_mac, *dst_mac, *ether_frame, *data;
	struct ifreq ifr;
	custom_protocol my_protocol;
	struct sockaddr_ll device;

	if(argc != 3){
		perror("Usage file [network deivce name] [data]");
		exit(1);
	}

	src_mac = (uint8_t *)allocate_strmem(6, 'u');
	dst_mac = (uint8_t *)allocate_strmem(6, 'u');

	interface = (char *)allocate_strmem(40, 'c');
	target = (char *)allocate_strmem(40, 'c');
	ether_frame = (char *)allocate_strmem(IP_MAXPACKET, 'c');
	src_ip = (char *)allocate_strmem(INET_ADDRSTRLEN, 'c');
	
	// Get Mac address
	if((fd = socket(PF_PACKET, SOCK_RAW, IPPROTO_RAW)) < 0){
		perror("socket() error");
		goto end; // goto main end line
	}

	strcpy(interface, argv[1]);
	
	memset(&ifr, 0x00, sizeof(ifr));
	snprintf(ifr.ifr_name, sizeof(ifr.ifr_name), "%s", interface);
	if(ioctl(fd, SIOCGIFHWADDR, &ifr) < 0){
		perror("ioctl() failed to get source MAC Address");
		goto end; // goto main end line
	}
	close(fd); //close fd
	printf("debug0\n");

	memcpy(src_mac, ifr.ifr_hwaddr.sa_data, 6 * sizeof(uint8_t));
	printf("MAC Address for interface %s is ", interface);
	for(int i = 0; i < 5; i++){
		printf("%02x:",src_mac[i]);
	}
	printf("%02x\n", src_mac[5]);

	memset(&device, 0x00, sizeof(device));
	if((device.sll_ifindex = if_nametoindex(interface)) == 0){
		perror("nametoindex() error");
		goto end; // goto main end line
	}
	printf("index of interface %s is %i\n", interface, device.sll_ifindex);

	memset(dst_mac, 0xff, 6 * sizeof(uint8_t)); // dest mac address 0xff
	
	device.sll_family = AF_PACKET;
	memcpy(device.sll_addr, src_mac, 6 * sizeof(uint8_t));
	device.sll_halen = 6;

	strcpy(src_ip ,"192.168.1.1");
	strcpy(target, "127.0.0.1");
	
	// mac address setting in ether_frame
	memcpy(ether_frame, dst_mac, 6 * sizeof(uint8_t));
	memcpy(ether_frame + 6, src_mac, 6 * sizeof(uint8_t));
	
	my_protocol.cmd_number = 0xB588;
	my_protocol.data_len = 4;//(strlen(argv[2]) * sizeof(uint8_t)) + 1;
	
	/*
	data = (uint8_t *)malloc(my_protocol.data_len);
	memset(data, 0x00, my_protocol.data_len);
	for(int i = 0; i < my_protocol.data_len - 1; i++){
		data[i] = argv[2][i];
	}
	printf("testing : %s\n", data);
	*/

	my_protocol.data[0] = 'W';
	my_protocol.data[1] = 'o';
	my_protocol.data[2] = 'W';
	my_protocol.data[3] = '\0';
	//printf("data's : %s\n", my_protocol.data);
	//printf("data lens : %d\n", my_protocol.data_len);

	memcpy(ether_frame, dst_mac, 6 * sizeof(uint8_t));
	memcpy(ether_frame + 6, src_mac, 6 * sizeof(uint8_t));
	
	memcpy(ether_frame + ETH_HDRLEN, &my_protocol, CUSTOM_PROTOCOL_LEN + 4);//(CUSTOM_PROTOCOL_LEN * sizeof(uint8_t)) + my_protocol.data_len);
	
	frame_length = 6 + 6 + 8;//((CUSTOM_PROTOCOL_LEN * sizeof(uint8_t)) + my_protocol.data_len);
	
	if((fd = socket(PF_PACKET, SOCK_RAW, htons(ETH_P_ALL))) , 0){
		perror("socket() error");
		goto end;
	}
	
	if((bytes = sendto(fd, ether_frame, frame_length, 0, (struct sockaddr *)&device, sizeof(device))) <= 0){
		perror("sendto() error");
		goto end;
	}

	close(fd);
end:
	free(interface);
	free(target);
	free(ether_frame);
	free(src_ip);
	free(src_mac);
	free(dst_mac);
	return 0;
}

void *allocate_strmem(int len, char type){
	void *tmp;
	int sizeof_num;

	if (len <= 0){
		perror("lens error");
		exit(1);
	}
	
	switch(type){
		case 'c' :
			tmp = (char *)malloc(len * sizeof(char));
			sizeof_num = sizeof(char);
			break;
		case 'u':
			tmp = (char *)malloc(len * sizeof(uint8_t));
			sizeof_num = sizeof(uint8_t);
			break;
	}

	if(tmp != NULL){
		memset(tmp, 0, len * sizeof_num);
		return tmp;
	}
	else {
		perror("malloc() error");
		exit(1);
	}
}
```
