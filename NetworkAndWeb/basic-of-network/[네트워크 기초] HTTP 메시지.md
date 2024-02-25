### http message

HTTP에서 교환하는 정보는 `HTTP 메시지`라고 부른다.<br>
HTTP 메시지는 복수 행(개행 문자는 CR+LF)의 데이터로 구성된 텍스트 문자열이다.

- HTTP 메시지는 `메시지 header`와 `메시지 body`로 구분되며, 최초에 나타나는 개행 문자(CR+LF)로 구분된다.
- CR(Carriage return: 16진수 0x0d), LF(Line Feed: 16진수 0x0a)

메시지(request/response) header는 다음과 같이 구성된다.<br>
line(reqeust/status), header field(request/response), normal header field, entity header filed, 그 외 HTTP RFC에는 없는 헤더 필드(쿠키 등)가 추가적으로 포함된다.

- request line : request에 사용하는 method와 request uri, http version
- status line : status code, response description, http version

http 통신 시, 그대로 전송하지 않고 인코딩을 통해 전송 효율을 높인다. (단, 컴퓨터에서 인코딩을 처리하므로 cpu 리소스가 쓰인다.)

### message, entity

http message body의 역할은 request, response에 관한 entity body를 운반하는 것.<br>
기본적으로 message body과 entity body는 동일하지만, `전송코딩`이 적용될 경우 달라진다.

- `message` : http 통신의 기본 단위, Octet sequence로 구성되고 통신을 통해 전송됨
- `entity` : request/response의 payload로 전송되는 정보로, entity header field와 entity body로 구성됨

### http content codings

content coding은 entity에 적용하는 인코딩을 가리킨다.<br>
entity 정보를 유지한 채로 압축하며, content coding된 entity는 수신한 client 측에서 decoing한다.

content codings 종류

- gzip(GNU zip)
- compress(Unix의 표준 압축)
- deflate(zlib)
- idenity(no encoding)

### chunked transfer coding

http 통신 시에, 요청했던 리소스 전부 중 entity body의 전송이 완료되지 않으면 브라우저 표시되지 않음.<br>
사이즈가 큰 데이터를 전송할 경우, data를 분할해서 조금씩 표시하는 방법을 사용한다.

entity body를 분할하는 기능을 chunked tranfer coding이라고 한다.

- 다음 청크 사이즈를 16진수를 사용해 단락을 표시하고 entity body 끝에 0(CR+LF)를 기록해둔다.

### multipart

MIME(Multipurpose Internet Mail Extensions) => 메일 같은 경우, 복수의 첨부 파일과 함께 메일을 보내곤 한다.<br>
MIME는 이미지 등의 바이너리 데이터를 아스키 문자열에 인코딩하는 방법과 데이터 종류 나타내는 방법 등을 규정한다.

MIME는 multipart라고 하는 여러 종류 데이터 수용하는 방법을 사용하는 것.

http도 multipart에 대응하고 있어서, 하나의 message body 내부에 entity를 여러 개 포함시켜 보낼 수 있다. 주로 이미지나 텍스트 파일 등 업로드할 때 사용한다.

- `multipart/form-data` : web form으로부터 파일 업로드
- `multipart/byteranges` : status code 206(partial content) response message가 복수 범위 내용 포함할 때 사용

http message로 multipart를 사용할 때는 content-type header field를 사용한다.

- multipart 각각의 entity를 구분하기 위해 "boundary"라는 문자열을 사용한다. 각 entity 시작과 끝에 `--`를 붙인다.
- multipart는 파트마다 header filed가 포함된다.

[RFC 2046](https://datatracker.ietf.org/doc/html/rfc2046)

### range request

요즘처럼 광대역 네트워크 사용하지 않을 때는, 대용량 데이터 다운로드 시 한 번 끊어지면 다시 다운로드 받아야했다.<br>
이를 해결하기 위해 `resume` 기능을 통해 이전에 다운로드한 곳을 기억해 재개할 수 있게 했다.

resume 기능을 구현하려면, entity 범위를 지정해서 다운로드해야 하며, 이를 `range request`라고 한다.

range request를 할 때는, range header field를 사용해서 `byte range`를 지정한다.

range request에 대한 response는 status code 206 (partial content)라는 메시지를 응답받는다.

- 서버가 range request를 지원하지 않을 경우 200 ok 응답과 함께 완전한 entity가 돌아온다.

### content negotiation

header field

- Accept
- Accept-Charset
- Accept-Encoding
- Accept-Language
- Content-Language

content negotiation 종류

- server-driven negotiation
- agent-driven negotiation
- transparent negotiation
