상태 코드 3자리 숫자 중 첫 번째 자리는 response의 클래스를 의미하며, 나머지 2자리는 분류가 없다.

- 1XX : informational (request를 처리 중)
- 2XX : success (request 정상 처리)
  - 200 OK : request를 서버가 정상 처리
  - 204 Not Content : reqeust 정상 처리, 하지만 response에 entity body를 포함하지 않음 (클라이언트가 서버로 정보 보내고 끝낼 때)
  - 206 Partial Content : range request에 의해 부분적 request 받음 (response에 Content-Range로 지정된 범위의 entity 포함)
- 3XX : redirection (request 완료를 위해 추가 동작 필요)
  - 301 Moved Permanently : request된 리소스에 새로운 uri가 부여되어 있을 떄 알려주는 용도 (보통 슬래시(/) 누락한 경우)
  - 302 Found : reqeuest된 리소스에 새로운 uri 할당되어 있음을 알림 (301과 비슷, 301은 영구적인 게 차이)
  - 303 See Other : request된 리소스는 다른 uri에 있으므로 get으로 추가로 얻어야 함을 알림 (redirect 장소를 get으로 얻어야 한다고 명시한다는 점에서 302와 다름)
  - 301,302,303을 받으면 브라우저는 post를 get으로 바꿔서 request entity body 삭제 후 자동으로 request 재송신하도록 되어 있음. (스펙상으로는 301,302는 post를 get으로 변경 불가능)
  - 304 Not Modified : 조건부 request일 경우, 리소스 엑세스는 허락, 조건 미충족 안내 => response body에 어떤 것도 포함x 여야 함. 30x지만, redirect와 관련x
  - 307 Temporay Redirect : 302 Found과 같은 의미, but 브라우저 사량에 따라 post => get으로 치환하지 않음.
- 4XX : client error (서버가 request 이해 불가능)
  - 400 Bad Request : request 구문 오류. 브라우저는 이를 200 ok와 같은 취급.
  - 401 Unauthorized : request에 http 인증(basic 인증, digest 인증 정보)가 필요하다고 안내. (401인 경우, request된 리소스에 적용되는 challenge를 포함한 WWW-Authenticate header field 포함할 필요 있음)
  - 403 Forbidden : reqeust된 리소스의 엑세스 거부. 거부 이유를 entity body에 명확히 기재해서 표시해야 함.
  - 404 Not Found : request한 리소스가 서버 상에 없음.
- 5XX : server error (서버가 request 처리 실패)
  - 500 Internal Server Error : request 처리 도중 서버 에러
  - 503 Service Unavaliable : 일시적 서버 과부하 혹은 점검 중. 시간 걸리는 경우 Retry-After 헤더 필드에 전달

위 클래스 정의만 지킨다면 [RFC2616](https://datatracker.ietf.org/doc/html/rfc2616)에서 정의된 상태 코드를 변경하거나 독자적인 상태 코드를 만들어도 된다.

WebDAV([RFC4918](https://datatracker.ietf.org/doc/html/rfc4918), [RFC5842](https://www.rfc-editor.org/rfc/rfc5842.html)), [Additional HTTP Status Codes](https://datatracker.ietf.org/doc/html/rfc6585)
