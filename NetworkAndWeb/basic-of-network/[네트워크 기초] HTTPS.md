## http 약점

### 평문(암호화x)

http 메시지는 암호화되지 않은 메시지(평문)이므로 누구나 읽을 수 있다.<br>
tcp/ip 구조의 통신 내용은 전부 통신 경로 도중에 읽을 수 있음. 인터넷은 전세계 경유 네트워크이므로.

세그먼트 통신을 도청하는 건 쉬운 일 => 네트워크 상을 흐르고 있는 패킷 수집하면 됨 (패킷 해석하는 packet capture, packet sniffer 등)

wireshark 같은 것도 packet capture 도구 중 하나.

#### 암호화 기술

1. 통신 암호화

http 자체는 암호화 구조가 없지만, ssl(Secure Socket Layer) 혹은 tls(Transport Layer Security) 같이 다른 프토콜을 조합해서 http 통신 내용 암호화

흔히 https라고 함.

2. 콘텐츠 암호화

콘텐츠 내용 자체를 암호화하는 방법. http에는 암호화 기능이 없으므로, http를 사용해서 운반하는 내용 자체를 암호화.

콘텐츠 암호화하려면 클라이언트와 서버가 콘텐츠 암호화, 복호화 구조를 가져야 함.

### 통신 상대 확인x이므로 위장 가능

http 통신에서 request, response는 통신 상대를 확인x

누구나 request 가능하므로 => 의미없는 request더라도 수신하게 됨 (대량 request로 공격하는 DoS 공격)

#### SSL을 사용

http에서는 통신 상대 확인x지만 SSL로는 상대 확인할 수 있음.

SSL은 암호화 뿐만 아니라 상대를 확인하는 수단으로 증명서 제공

증명서 위조는 기술적으로 어려움.

### 완전성 증명x이므로 변조 가능

완전성 == 정보의 정확성.

완정성 증명x == 정보 정확한지 확인x

reqeust/response가 발신된 후 상대가 수신할 때까지 그 사이에 변조되더라도 알아차릴 수 없음.<br>
즉, 발신한 request/response가 수신한 reqeust/response가 다를 수도 있음.

이렇게 도중에 request/response를 빼앗아 변조하는 공격을 `중간자 공격(Man-in-the-Middle)`이라고 부름.

#### 변조 방지하려면?

완벽한 방법은 아직x

그나마 `MD5`나 `SHA-1` 등 해시 값을 확인하거나 파일의 디지털 서명을 확인하는 방법을 자주 사용함.

확실히 방지하려면 https 사용. => SSL에는 인증, 암호화, 그리고 다이제스트 기능 제공

## https

`http + 통신 암호화 + 증명서 + 완전성 보호 = https`

https는 http 통신하는 소켓 부분을 ssl, tls라는 프로토콜로 대체함.

보통 http는 tcp와 직접 통신하지만,<br>
https에서는 http가 ssl과 통신/ssl이 tcp와 통신

ssl은 http와 독립된 프로토콜.

### 공개키 암호화 방식

ssl에서는 공개키 암호화 방식 채용.

암호화/복호화에 key를 사용.

#### 공통키 암호 딜레마

암호화/복호화에 1개의 key를 사용하는 방식이 공통키 암호.

key를 상대방에게 넘겨줘야 하는데, 통신 도중에 key를 뺏길 수 있음.

#### 2개의 key를 사용하는 공개키 암호

`공통키 암호` 문제를 해결하는 게 `공개키 암호`

서로 다른 2개의 key pair를 사용. 한쪽은 비밀키(private key), 다른 하나는 공개키(public key).

암호를 보내는 측이 상대의 공개키를 사용해서 암호화. => 암호화된 정보를 받은 상대는 자신의 비밀키로 복호화.

- 비밀key를 통신으로 보낼 필요x
- 공개key는 누구에게나 넘겨줘도 괜찮음.

#### https는 하이브리드 암호 시스템

공통키 암호와 공개키 암호 양쪽 성질을 가진 하이브리드 암호 시스템.

공개키 암호는 공통키 암호에 비해 처리 속도가 늦음. => 모든 통신에 공개키 암호를 사용하는 건 비효율적임.

공개키 암호, 공통키 암호 방식을 조합해서 사용.

#### 공개키의 문제점

공개키가 진짜인지 증명이 불가능하다는 단점.

암호를 사용해서 통신 시작할 때 => 수신한 공개키가 본래 의도한 서버가 발행한 공개키인지 증명 어려움.

이 문제를 해결하기 위해 인증 기관(CA: Certificate Authority)과 그 기관이 발행한 공개키 증명서가 사용됨.

- 인증 기관이라느 클라이언트와 서버가 모두 신뢰하는 제3기관. 유명한 게 VeriSign 사.

#### 인증기관의 공개키로 확인

서버 운영자가 인증 기관에 공개키 제출.

- => 인증기관은 제출된 공개키에 디지털 서명 => 서명이 끝난 공개키 만듦
- => 공개키 인증서에 서명 끝난 공개키 담음.
- => 서버는 이 인증 기관에 의해 작성된 공개키 인증서를 클라이언트에 보내고, 공개키 암호로 통신
- => 공개키 인증서는 디지털 증명서(혹은 증명서)라고도 함
- => 증명서 받은 클라이언트는 증명 기관의 공개키 사용해서 -> 서버 공개키가 신뢰할만하다고 판단
- => 증명기관의 공개키는 안전하게 전달되어야 하므로, 보통은 브라우저가 주요 인증 기관의 공개키를 사전에 내장하고 있음.

#### 조직의 실제성 증명하는 EV SSL 증명서

증명서 역할은 서버가 올바른 통신 상대임을 증명하는 것.<br>

++ 상대방이 실제로 있는 기업인지 확인하는 역할. 이런 역할 가진 증명서가 EV SSL 증명서.

EV SSL은 세계 표준 인정 가이드라인에 의해 발행되는 증명서.

브라우저는 EV SSL로 인증된 웹사이트인지 알려주는 장치들을 가지고 있음. 시각적으로도 알려줌.

#### 클라이언트 확인하는 클라이언트 증명서

https에서는 클라이언트 증명서도 이용할 수 있음. => 서버가 통신하는 클라이언트가, 의도한 클라이언트인지 확인하는 용도.

클라이언트 증명서 몇가지 문제점.

- 증명서 입수와 배포 : 유저가 설치해야 해서 비효율적임. 인터넷 뱅킹 같은 데서 쓰기도 함.
- 클라이언트 실재만 증명할 뿐, 사용자 존재 유무 증명x

#### 인증기관의 신용

ssl은 인증 기관을 신용할 수 있다는 전제가 필요.

OpenSSL 등 소프트웨어를 사용하면 누구든 인증기관 구축해서 서버 증명서 발행 가능. => 의미 없음.

## https 통신 구조

1. 클라이언트가 client hello 송신 => ssl 통신 시작 (클라이언트가 제공하는 ssl 버전 명시, 암호 스위트(cipher suite)로 불리는 리스트 등이 메시지에 포함)
2. 서버가 ssl 통신 가능한 경우 server hello 응답. 클라이언트처럼 ssl 버전과 암호 스위트 포함. 서버가 보낸 암호 스위트에는 클라이언트가 보낸 암호 스위트 목록 중 선택한 것임.
3. 서버가 certificate 메시지 송신. 메시지에 공개키 증명서 포함.
4. 서버가 server hello done 메시지 송신. 최초의 ssl negotiation 끝났다고 통지.
5. 최초의 ssl negotiation 끝나면, 클라이언트가 client key exchange 메시지로 응답. 메시지에는 통신 암호화하는 데 사용하는 Pre-Master secret이 포함되어 있음. 이 메시지는 3번에서 받은 공개키 증명서에서 꺼낸 공개키로 암호화되어 있음.
6. 클라이언트는 Change Cipher Spec 메시지 송신. 이 메시지는 이후 통신을 암호키 사용해서 통신한다는 의미.
7. 클라이언트가 Finished 메시지 송신. 이 메시지는 접속 전체의 체크 값 포함. 이후 서버가 이 메시지를 올바르게 복호화할 수 있으면 negotiation 성공.
8. 서버에서도 마찬가지로 Change Cipher Spec 메시지 송신.
9. 서버에서도 마찬가지로 Finished 메시지 송신.
10. 서버와 클라이언트가 Finished 메시지 교환 완료되면 ssl에 의해서 접속 확립. 이제부터 어플리케이션 계층의 프로토콜(http)에 의해 통신. 즉 http reqeust 송신.
11. http response 받음. (애플리케이션 계층의 프로토콜에 의한 통신)
12. 마지막으로 클라이언트가 접속 끊음. close_notify 메시지 전송. => 이후 TCP FIN 메시지 보내고 TCP 통신 종료

또한, 애플리케이션 계층의 데이터 송신할 때 MAC(Message Authentication Code)라 불리는 메시지 다이제스트 덧붙일 수 있음.

MAC을 잉요해서 변조 감지 가능하므로 완전성 보호 실현 가능.

### SSL과 TLS

ssl은 본래 브라우저 개발 회사였던 넷스케이프 커뮤니케이션사가 내놓은 프로토콜. ssl 3.0까지 같은 회사에서 개발 => 현재는 `IETF`로 옮겨짐

`SSL3.0`을 기반으로 한 `TLS1.0`이 책정되고 이후 버전이 높아짐. TSL는 SSL을 바탕으로 한 프로토콜.

#### SSL은 느린가?

1. https는 서버, 클라이언트 모두 암호화/복호화 처리가 필요하므로 cpu나 메모리 등 하드웨어 리소스 필요로 함.
2. http 통신에 비해 ssl 통신만큼 네트워크 리소스를 더 소비함. 또한 ssl 통신만큼 통신 처리에 시간 걸림.

http보다 https가 느림. => ssl 엑셀레이터라는 하드웨어를 사용해서 문제를 해결하기도 함.

암호화 통신은 평문 통신에 비해 cpu, 메모리 등 리소스가 많이 소요되므로 굳이 암호화가 필요하지 않다면 http도 많이 씀.

- 모든 콘텐츠를 암호화하는 게 아니라 숨겨야 할 콘텐츠만 암호화해서 리소스를 줄인다거나...
- 증명서 구입 비용도...
