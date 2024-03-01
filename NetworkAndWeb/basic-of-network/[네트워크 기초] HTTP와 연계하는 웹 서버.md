## 가상 호스트(virtual host)

http/1.1에서는 하나의 http 서버가 여러 개의 웹 사이트 실행 가능<br>

이때 domain name을 다르게 하더라도, 결국 request가 서버에 도착할 때는 ip 주소를 기준으로 엑세스하므로 (같은 ip 주소에서 여러 개의 웹사이트를 실행하는 가상 호스트 시스템이므로)

- => http request를 보낼 때 호스트명과 도메인명을 완전하게 포함하는 uri를 지정하거나 반드시 host header field를 지정해야 함

## 통신 중계 : 프록시, 게이트웨이, 터널

http는 클라이언트와 서버 이외에 proxy, gateway, tunnel과 같이 통신 중계하는 프로그램과 서버 연계하는 것 가능.

### proxy : 서버와 클라이언트 양쪽 중계 (클라이언트로부터 request를 서버로 전송, 서버로부터 response를 클라이언트로 전송)

클라이언트로부터 받은 request uri를 변경하지 않고 => 리소스 가진 서버로 전송<br>
리소스 본체를 가진 서버를 origin server라고 칭함.

http 통신 시, proxy server를 여러 대 경유 가능. (여러 대 공유해서 request, response 중계 시 via header field에 경유한 호스트 정보 추가해야 함)

proxy server 사용 목적

- 캐시, 엑세스 제한, 엑세스 로그 획득, ...

proxy server 사용 방법 2가지 분류

- 캐시 사용 여부로 구분, 메시지 변경 여부

#### Caching Proxy

proxy server 상에 resource cache를 보관하는 타입.

- 같은 리소스 request인 경우, origin server로부터 리소스 획득하지 않고 cache response 전송

#### Transparent Proxy

메시지 변경하지 않는 타입을 투명 프록시라고 함. 메시지 변경하면 non transparent proxy

### gateway : 다른 서버 중계 (클라이언트로부터 수신한 request를 리소스 보유한 서버인 것처럼 수신. 클라이언트가 gateway라는 걸 인지 못하는 경우도 존재)

게이트웨이 동작은 프록시와 유사함. 다만, 게이트웨이는 프록시와 달리 origin server가 http protocol 이외의 통신을 제공하는 서버가 됨.

db에 접속해서 sql query를 사용해 데이터 얻을 때, 결제 시스템 연동할 때 => 클라이언트와 그 사이를 암호화하는 등의 통신 안정성 높이는 목적

### tunnel : 클라이언트와 서버 접속 중계

tunnel은 요구에 따라 다른 서버와의 통신 경로 확립 목적. (클라이언트는 ssl 같은 암호화 통신을 통해 서버와 안전하게 통신하기 위해 사용)<br>
tunnel 자체는 http request를 해석x. 그대로 다음 서버에 중계.

tunnel은 통신하고 있는 양쪽 끝이 끊어지면 종료.

## 리소스 보관하는 캐시

cache는 proxy server와 클라이언트의 로컬 디스크에 보관된 리소스 복사본을 가리킴.

리소스 복사본을 통해 => 통신량과 통신 시간 절약 가능

캐시 서버는 proxy server 중 하나인 caching proxy로 분류됨.

### 캐시 유효기간

cache server에 cache가 있더라도, 항상 캐시된 리소스를 돌려주지 않음. (캐시된 리소스의 유효성)

캐시를 가지고 있더라도, 클라이언트가 요구하거나 캐시의 유효기간 등에 의해 origin server에 리소스 유효성 확인하거나 새로운 리소스 획득하러 감.

### 클라이언트에 있는 캐시

브라우저도 캐시 존재.

브라우저가 유효한 캐시 가지고 있으면, 같은 리소스 엑세스를 서버가 아닌 로컬 디스크로부터 함.

## http 등장 이전 프로토콜

ftp (file transfer protocol)

- tcp/ip 이전에 등장. 지금도 많이 씀.

nntp (network news transfer protocol)

- netnews라는 전자 회의실에서 메시지 보낼 때 쓰던. 웹 쓰면서 이제 잘 안 씀

archie

wais(wide area information servers)

gopher
