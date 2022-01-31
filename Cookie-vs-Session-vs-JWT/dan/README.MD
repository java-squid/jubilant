## 공통

* HTTP 는 Connectionless, Stateless 를 지향하다보니 생기는 단점을 보완하기 위해 사용.

## Cookie

* 보통 session cookie 를 말합니다.
* key, value, expires, domain, path 로 구성되며, 서버가 생성하여 클라이언트 로컬에 저장됩니다.
* 만료 시간에 따라 브라우저를 종료해도 남아있을 수 있습니다.
* 보통 장바구니와 같은 개인화 정보에 사용됩니다.
* 보안에 취약합니다.
    - 암호화를 사용하기도 합니다.
* request 에 포함됨으로 cookie 가 늘어날수록 네트워크 패킷의 크기가 커집니다.
    - 압축을 하기도 합니다.

## Session

* 가장 기본적인 방법으로 jsession 이 있습니다.
    - Tomcat 에서 세션을 유지하기 위해 발급하는 키로 쿠키에 저장 됩니다.
        - ex) Set-Cookie: JSESSIONID=3CB361E0BE1A9A7DE7DB926DF0772BAE
* 서버가 생성하며 서버에 저장됩니다.
    - 실무에서는 주로 Redis 를 사용합니다.
    - 서버가 원할 때 만료시킬 수 있습니다.
* 브라우저를 종료하거나, 서버에서 의도적으로 만료시키는 경우 삭제됩니다.
* 다중 서버 환경에서의 Session 관리
    - 과거에는 web server 에서 Sticky Session 을 구현했지만, 최근에는 AWS 의 LB 를 사용해서 관리합니다.
    - Sticky Session
        - 쿠키 또는 세션을 사용하여 트래픽을 분산하는 기술.
        - 주로 LB 를 사용하여 구현하며 LB 에 부하가 집중되며, 분배 정책에 따라 특정 서버로 트래픽이 집중될 수 있는 단점이 있습니다. (디폴트 정책 - RR)
        - 서버가 뻗는 경우 세션이 유실되고, 스케일 아웃에 약합니다.
    - Session clustering
        - 여러 대의 서버에서 세션을 공유합니다.
        - 장애와 유실에 강합니다.
        - 세션 복제로 인한 메모리 낭비와 동기화, 락 등의 단점이 있습니다.
    - Session Storage
        - 세션을 별도의 공간에 관리합니다.
        - 보통 휘발성의 inmemory DB 인 Redis 를 사용하지만, 필요에 따라 indisk DB 를 사용하기도 합니다.

## JWT (Json Web Token)

* 다중 서버 환경에 가장 적절한 방식입니다.
* header
    - typ: 토큰의 타입을 지정 (jwt)
    - alg: 해싱 알고리즘을 지정
* payload
    - claims 이 저장되는 공간으로 key, value 로 이루어져 있습니다.
        - java 에서는 보통 hashmap 으로 생성합니다.
        - Registered Claims
            - iss: 토큰 발급자 (issuer)
            - sub: 토큰 제목 (subject)
            - aud: 토큰 대상자 (audience)
            - exp: 토큰의 만료시간 (expiraton)
            - nbf: 토큰 활성 날짜
            - iat: 토큰이 발급된 시간 (issued at)
            - jti: 중복 방지를 위한 JWT의 고유 식별자
        - Public Claim
            - 사용자 정의 claims 이지만 충돌을 방지하기 위해 key 값을 URL 형태로 작성합니다.
        - Private Claims
            - 사용자 정의 claims 으로 클라이언트와 약속된 key 만을 사용합니다.
    - 보통 Registered, Private Claims 를 많이 사용합니다.
    - claims 는 토큰 진위 여부나 복호화 없이 해석할 수 있음으로 민감 정보가 들어가서는 안됩니다.
* signature
    - signature는 토큰을 인코딩하거나 유효성 검증을 할 때 사용하는 고유한 암호화 코드입니다.
    - header, payload 값을 각각 BASE64로 인코딩   
      → secret 키를 이용해 header에서 정의한 alg로 해싱   
      → BASE64로 인코딩하여 생성
* 한번 발급된 토큰은 만료되기 전에는 서버에서 만료 시킬 수 없습니다.
    - 유일한 방법은 secret 키 변경 뿐 입니다.
        - 이를 방지하기 위해서 만료 시간을 작게 주고 refreshToken 을 사용합니다.
    - 서버에서 강제 만료를 시키기 위해 redis 로 관리하는 방법도 있습니다.  
      session 과의 차이점은 최소 정보만 관리한다는 것입니다.
* 보안성 강화를 위한 방법
    - secret key 관리: key rotation, key rolling
    - 만료 시간이 짧은 accessToken과 만료 시간이 상대적으로 긴 refreshToken을 사용하는 전략

## 기타

* 보안 취약점
* JWT 구현 샘플 [https://github.com/Hyune-c/common-util](https://github.com/Hyune-c/common-util)
