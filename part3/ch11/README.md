# 실전: 책임의 분리

## 예제의 시나리오와 요구 사항
```
1. 비즈니스 논리 서버의 /login 엔드 포인트를 호출해 사용자 이름과 암호를 인증하면 무작위로 생성된 OTP 를 얻는다.
2. 사용자 이름과 OTP 를 이용해 /login 엔드포인트를 호출하면 토큰을 얻을 수 있다.
3. 2 단계에서 얻은 토큰을 http 요청의 Authorization 헤더에 추가하고 다른 엔드포인트를 호출한다.

로그인과 OTP 를 이용한 인증 방식을 다단계 인증이라고 한다 277p

일반적인 경우 인증 서버만 암호를 공유해야한다 예제를 위해 비즈니스 논리 서버에서도 암호를 공유한다. 278p
```

## 토큰의 구현과 이용

### 토큰이란?

토큰은 이론상 일종의 출입 카드다. 호텔에 방문하면 안내 데스크에 신원을 밝힌 후(인증) 출입 카드(토큰) 을 

받을 수 있다. 이 카드로 허용되는 문을 열 수 있지만 모든 문을 열 수있는 것은 아니다!

토큰은 접근 권한을 부여하고 특정한 작업이 허용되는지 결정한다.

```
* 토큰의 장점

토큰을 이용하면 요청할 때마다 자격 증명을 공유할 필요가 없다.
-> HTTP Baisc 을 이용해서 아이디와 비밀번호를 인코딩해서 보내지 않는 대신 토큰을 사용하면 된다.

토큰은 수명을 지정할 수 있다.
-> 토큰이 탈취 당하더라도 토큰은 영원하지 않다! 토큰으로 시스템 침입 방법을 찾기 전에 토큰이 만료될 가능성이 크다.

클라이언트가 요청할 때 보내야하는 사용자 권한과 같은 세부 정보를 토큰에 저장할 수도 있다.
-> 토큰에 사용자의 권한과 역할 같은 세부 정보를 저장하면 서버 쪽 세션을 클라이언트 쪽 세션으로 대체하여 
수평 확장을 위한 높은 유연성을 달성할 수 있다 (12~15 장에서 알아봄)

토큰일 이용하면 인증 책임을 시스템의 다른 구성 요소에 위임할 수 있다.
-> 사용자를 직접 관리하지 않고 다른 플랫폼에서 가지고 있는 계정의 자격 증명으로 인증할 수 있다.
```

### JSON 웹 토큰이란? (JWT) 
```
웹 요청을 위한 JSON 형식으로 구현된 토큰,JWT 는 세 부분으로 구성되고 각 부분은 마침표로 구분된다

ASDFDSAFSDFA.asdfdsagads.sadfsadfsaf // 예시

첫 두 부분은 헤더와 본문으로 JSON 형식으로 만들어지며 Base64 로 인코딩된다. 
헤더에는 토큰과 관련된 메타 데이터를 저장한다, 본문은 권한 부여에 필요한 세부 정보를 저장할 수 있다.

토큰은 가급적 짧게 유지하고 본문에 너무 많은 데이터를 추가하지 않아야 한다.
토큰이 길면 요청 속도가 느려지고, 토큰에 서명하는 경우 토큰이 길수록 암호화 알고리즘이 서명하는 시간이 길어진다.

토큰의 마지막 부분은 디지털 서명으로 이 부분은 생략할 수도 있다. 보통 헤더와 본문에 서명하고 나중에 서명을 
이용해 내용이 변경되지 않았는지 확인할 수 있다.

서명이 없으면 네트워크에서 토큰을 전송할 때 누군가가 토큰을 가로채고 내용을 변경하지 않았는지 확신할 수 없다.
```
## 실습 예제 구현 하기

### 인증 서버 구현하기
```
인증 서버는 사용자 자격 증명과 요청 인증 이벤트로 생성된 OTP 가 저장된 데이터베이스에 연결한다.
비즈니스 논리 서버와 인증 서버 간의 인증 구현해보기 287p

authenticationserver 참고
```
### 비즈니스 논리 서버 구현하기

ex 11 businesslogicserver 참고 , JWT 의존성 추가 xml 참고 

비즈니스 논리 서버는 /test 엔드포인트 하나만 가지며 나머지 부분은 이 엔드 포인트를 보호하기 위한 것이다. 
```
* 인증 서버를 통해서 비즈니스 논리 서버에 접근하는 단계

1. 사용자는 인증을 위한 자격 증명을 보내고 인증 서버는 사용자를 인증하고 OTP 코드를 포함한 SMS 를 보낸다.
2. 첫 번째 인증 단계에서 받은 OTP 코드를 보내면 인증 서버는 OTP 코드를 검증한 후 토큰을 클라이언트로 보낸다.
3. 사용자는 인증 서버에서 얻은 토큰으로 비즈니스 논리 서버가 노출하는 리소스 ("/test") 에 접근할 수 있다.
```
```
* 비즈니스 서버 아키텍처 설계하기 301p

두 개의 필터를 가지는 비즈니스 서버 (InitailAuthenticationFilter, JwtAuthenticationFilter)

비즈니스 서버의 InitailAuthentiationFilter(인증 필터) 는 인증 서버를 사용하기 위해 두 개의 인증 공급자를 가진다.

공급자 1 사용자 이름과 암호로 사용자를 인증하는 
공급자 2 OTP 로 사용자를 인증한다.

보안 컨텍스트로 JWT AuthenticationFiler 를 호출한다. 이 필터는 /login 을 제외한 모든 경로에 적용되며 
엔드 포인트를 호출할 수 있도록 JWT 를 검증한다.
```
```
* Authnetication 객체 구현하기

비즈니스 논리 서버를 구축하는 데 필요한 두(일반 인증, OTP 인증에서 사용할) Authentication 객체 구현하기

UsernamePasswordAuthentication
OtpAuthentication  참고 

처음 인증을 호출할 때 매개변수가 두 개인 생성자를 호출하고, 인증 후 매개변수가 세 개인 생성자를 호출해서 
인증 상태를 true 로 만든다. 
```
```
* 인증 서버에 대한 프록시 구현

AuthenticationProvider 를 구현하기 전 인증 서버가 노출하는 REST API 를 호출하는 프록시 구현하기

인증 서버의 REST API 를 호출하기위한 작업

1. 인증 서버가 노출하는 REST 서비스를 호출하는 데 이용할 모델 클래스 User 를 정의한다.
2. REST 엔드포인트를 호출하는 데 이용할 RestTemplate 을 사용한다.
3. 이름/암호 인증 이름/otp 인증을 수행하는 메서드 두 개를 프록시 클래스에 만든다! 

businesslogicserver AuthenticationServerProxy 참고
```
```
* AuthenticationProvider 인터페이스 구현 

인증 공급자를 구현하고 맞춤형 인증 논리를 만들기! businesslogicserver Authentication 참고
공급자를 직접 구현하는 경우 인증 매니저도 직접 구현해야 한다! ch5 참고


앞서 인증을 호출하는 메서드와, OTP 를 호출하는 메서드 두 개를 만들었다. 공급자도 여기에 맞춰 두 개의 
클래스를 만들어서 맞춤형 인증 논리를 구현한다.
```
```
* 필터 구현

맞춤형 필터 구현하기! 요청을 가로채고 인증 논리를 적용하는 것! JWT 기반 인증을 위한 필터 만들기
 
InitialAuthenticationFilter 가 인증 필터를 대체하고, JwtAuthenticationFilter 가 권한 부여 필터를 보완한다. 
171p 그림 참고 
필터에서 요청을 허용할지 결정한다.

InitailAuthenticationFilter -> SecurityContext <- JwtAuthenticationFilter -> Contorller 

OncePerRequestFilter: 한번만 호출되는 필터
shouldNotFilter 메서드를 재정의해서 필터 적용 여부를 결정할 수 있다.

businesslogicserver filter 참고
```
```
* InitailAuthenticationFilter 참고

비즈니스 논리서버는 JWT 토큰의 키 값을 가진다. 비즈니스 논리서버에서 키 값으로 토큰에 서명하고 클라이언트가
엔드포인트를 호출할 때 키를 이용해서 토큰을 검증한다. 

예제에서는 간단하게 구현하기 위해 모든 사용자에 하나의 JWT 토큰 키를 사용했지만, 실제 시나리오에서는
사용자별로 다른 토큰 키를 사용해야 한다.

사용자별 다른 토큰 키를 사용하면 토큰을 무효화 해야 할때 필요한 사용자의 토큰만 무효화 할 수 있다는 것이다.

jwt.signing.key=ymLTU8rq83.. // JWT 토큰 키 properties 참고 

Tip
예제에서는 실습을 위해 최소한의 구현으로 작성했지만, 프로덕션 환경에서는 예외 처리 및 이벤트 기록 같은
세부적인 사항을 만들어야 한다.
```
```
* JwtAuthenticationFiler 참고 

로그인 외의 다른 모든 경로에 대한 요청을 처리하는 필터, 이 필터는 요청의 Authorization 헤더에 JWT 가 있다고
가정하고 서명을 확인해 JWT 를 검증한 후 인증된 Authentciation 객체를 만들고 SecurityContext 에 추가한다. 314p

```

### 비즈니스 논리 서버 보안 구성 작성하기
```
* business-logic-server SecurityConfig 참고

인증 관리자를 만들고 공급자를 등록했다. 다른 출처를 이용할 때 적용되는 사항이 아니므로 csrf 보호를 비활성화 한다.
JWT 토큰 이용해서 CSRF 토큰을 통한 검증을 대신했다. 
```

## 전체 시스템 테스트

```
헤더 값으로 로그인 아이디와 비밀번호로 post localhost:9090/login 을 호출해서 otp 코드를 받는다
헤더 값으로 username, code 를 전달해서 otp 인증을 받으면 JWT 토큰을 받을 수 있다.

Authorization 토큰을 헤더에 넣고 원하는 엔드포인트를 호출하면 된다! localhost:8080/test 

authentication-server, business-logic-server 참고.
```


