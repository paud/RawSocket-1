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
- 프로토콜의 넘버는 0x88b5니다.

아래 그림과 같은 결과를 얻을 수 있습니다.

(그림)

### 소스코드
소스코드는 Ubuntu 16 LTS환경에서 작성되어씁니다.

```

```
