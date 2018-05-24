# Integrating with Melon using Authorization Code Grant

### Prerequisites
**authorization credentials 생성**  
클라이언트 혹은 제휴사가(이하 '파트너') OAuth2를 이용해 Melon Resource API를 사용하려면,  
먼저 파트너를 식별할 수 있는 authorization credentials(client_id/client_secret 기반)를 발급 받아야 합니다.  
현재, 수동적인 프로세스를 통해 authorization credentials을 발급하고 있습니다.  
아래 내용을 포함한 정보를 담당자(파트너사와 협업하는 Melon의 담당자)에게 전달해주시면 됩니다.  

1. 애플리케이션 유형 : (웹애플리케이션/Android/Chrome앱/iOS/기타)
2. 파트너 담당자 이메일 주소 :
3. 사용자에게 표시 되는 애플리케이션 명(서비스 명) :
4. Redirect Uris : authorization code를 전달 받을 수 있는 redirect_uris. 복수설정 가능합니다. ex) https://your.site.com/redirect_oauth)


![#f03c15](https://placehold.it/15/f03c15/000000?text=+) `발급된 authorization credential은 외부 유출이 되지 않도록 별도 보관하셔야 합니다.  
(만약, credential이 유출되었다고 판단되는 경우 담당자에게 연락주시면 client_secret을 변경해드립니다.)`


**scope 정의**  
Melon Resource Apis는 scope을 통해 세부적으로 제어 됩니다.  
파트너는 scope에 따른 '약관 동의'를 사용자로부터 승인 받아야 하며, 승인된 scope의 resource API만 접근 허용됩니다.  

OAuth 2.0 integration을 진행하기 전에, 멜론의 제휴담당자 혹은 기획담당자와 협의하여 사용될 서비스에 대한 scope을 식별하시길 권장합니다.  

현재 정의된 Melon OAuth 2.0 API scope은 다음과 같습니다.

Scope               | Description                             | Required
:-----------------------| :---------------------------------------| :-----------
user-private-read       | 사용자 "개인정보"(name, nickname) read 권한  | **Yes**
streaming               | 음원재생 권한                              | No
user-playlist-read      | 사용자 "플레이리스트" read 권한               | No
user-like-read          | 사용자 "좋아요" read 권한                   | No
user-like-modify        | 사용자 "좋아요" write/delete 권한           | No   
user-pay-amount-read    | 사용자 "결제 요청 금액" read 권한             | No

### OAuth2 access tokens 발급 flow
Melon의 OAuth2 access token을 발급받는 과정은 크게 *4단계*로 나뉩니다.

**Step 1: Authorization Endpoint을 이용한 authorization_code 발급 요청**  
첫번째로, 파트너앱에서 웹뷰를 로드하여 authorization HTTP request를 요청합니다.  
request에는 파트너앱을 식별할 수 있는 정보와 부여받고자 하는 scope 정보 등의 parameter가 포함됩니다.  
또한 "약관동의" 절차를 통해 Melon Resource 사용 권한을 파트너사에 제공할 것인지 사용자로부터 승인 받습니다.

(Melon OAuth2 Authorization Endpoint *URL*)  
Authorization Endpoint는 HTTPS 호출만 가능하며, HTTP *GET* Method로 호출되어야 합니다.
* `https://auth.melon.com/oauth/authorize`

(Melon OAuth2 Authorization Endpoint *Parameters*)  
Authorization Endpoint URL과 같이 전달되어야 하는 파라미터는 *query string*으로 전달되어야 하며,
그 항목은 다음과 같습니다.

Parameter           | Required      | Description
:-------------------| :-------------| :--------------------------------------------------------
response_type       | **Yes**       | "code" 값을 설정
client_id           | **Yes**       | 파트너 앱의 client_id(발급 받은 authorization credential)
redirect_uri        | **Yes**       | 사용자 인증(로그인, 약관동의) 후, 리다이렉트 될 location 정보. authorization credential 발급 과정 > Redirect Uris 정보와 일치해야 함.
scope               | **Yes**       | 파트너 앱이 사용자의 동의하에 사용할 수 있는 Melon resource의 범위 정보(*scope 정의 참고*). 복수 시, space 구분자를 이용.
state               | Recommended   | 랜덤 문자열 정보로 파트너에서 호출하는 request와 그후 callback의 신뢰성을 보장하기 위해 사용됨(CSRF 방어 목적).

(Example)
```
--HTTP GET
https://auth.melon.com/oauth/authorize?
 response_type=code&
 client_id=partner_client_id&
 redirect_uri=https%3A%2F%2Fpartner.site.com%2Fcallback&
 scope=user-private-read%2Fstreaming%2Fuser-playlist-read&
 state=xyz
```


**Step 2: Authorization Endpoint 응답 처리**  
사용자의 로그인 및 약관동의 프로세스가 정상적으로 완료되면 OAuth 서버는 authorization_code 값을 포함한 정보를 설정한 redirect_uri로 응답합니다.    
만약 사용자가 약관동의를 거부하였다면 authorization_code 대신 error 메시지를 응답합니다.  
authoriaztion_code 와 error 메시지는 아래와 같이 *query string*에 포함됩니다.  

성공 응답:  
`https://partner.site.com/callback?code=4BaiLw&state=xyz`

에러 응답:  
`https://partner.site.com/callback?error=access_denied&error_description=User%20denied%20access&state=xyz`  


**Step 3-1: authorization_code를 이용한 refresh_token & access_token 획득(by Token Endpoint)**  
파트너 웹 서버에서 authorization_code를 정상적으로 발급 받았다면,
Token Endpoint request를 통해 refresh_token 및 access_token을 발급받을 수 있습니다.  

![#f03c15](https://placehold.it/15/f03c15/000000?text=+)
`Token Endpoint request는 반드시 파트너의 서버단에서 이루어 저야 합니다.
(파트너의 authorization credential 보호 목적)`  

(Melon OAuth2 Token Endpoint URL)  
Token Endpoint는 HTTPS 호출만 가능하며, HTTP *POST* Method로 호출되어야 합니다.
* `https://auth.melon.com/oauth/token`   

(Melon OAuth2 Token Endpoint Parameters)  
Token Endpoint와 같이 전달되어야 하는 파라미터는 *request entity-body*에 포함되어야 하며,
*UTF-8*로 인코딩 된 *application/x-www-form-urlencoded* 포맷을 사용해야 합니다.
파라미터의 항목은 다음과 같습니다.

Parameter           | Required      | Description
:-------------------| :-------------| :--------------------------------------------------------
grant_type          | **Yes**       | "authorization_code" 값을 설정
code                | **Yes**       | Authorization Endpoint 응답으로 전달받은 code
redirect_uri        | **Yes**       | redirect_uri 정보(Authorization Endpoint request시 전달한 값과 반드시 동일해야 함)
client_id           | **Yes**       | 파트너 앱의 client_id(발급받은 authorization credential)
client_secret       | **Yes**       | 파트너 앱의 client_secret(발급받은 authorization credential)

![#f03c15](https://placehold.it/15/f03c15/000000?text=+)
`파트너 앱의 authorization credintial 정보는 Authorization: Basic (Http Basic)으로도 전달이 가능합니다.`

(Example)
```

POST /oauth/token HTTP/2
Host: auth.melon.com
Content-type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=4BaiLw&
redirect_uri=https://partner.site.com/callback&
client_id=partner_client_id&
client_secret=partner_client_secret
```

**Step 3-2: refresh_token을 이용한 access_token 갱신(by Token Endpoint)**  
이미 발급받은 access_token이 만료될 경우, refresh_token을 통해 access_token을 갱신할 수 있습니다.

(Melon OAuth2 Token Endpoint URL)  
Token Endpoint는 HTTPS 호출만 가능하며, HTTP *POST* Method로 호출되어야 합니다.
* `https://auth.melon.com/oauth/token`   

(Melon OAuth2 Token Endpoint Parameters)  
Token Endpoint와 같이 전달되어야 하는 파라미터는 *request entity-body*에 포함되어야 하며,
*UTF-8*로 인코딩 된 *application/x-www-form-urlencoded* 포맷을 이용해야 합니다.
파라미터의 항목은 다음과 같습니다.

Parameter           | Required      | Description
:-------------------| :-------------| :--------------------------------------------------------
grant_type          | **Yes**       | "refresh_token" 값을 설정
refresh_token       | **Yes**       | 이미 발급받은 사용자의 refresh_token
client_id           | **Yes**       | 파트너 앱의 client_id(발급받은 authorization credential)
client_secret       | **Yes**       | 파트너 앱의 client_secret(발급받은 authorization credential)

(Example)
```

POST /oauth/token HTTP/2
Host: auth.melon.com
Content-type: application/x-www-form-urlencoded

grant_type=refresh_token&
refresh_token=<refresh_token>&
client_id=partner_client_id&
client_secret=partner_client_secret
```

**Step 4: Token Endpoint 응답 처리**  
Token Endpoint의 request가 정상적이면, short-lived access_token과 refresh_token(TTL 없음)이 JSON 포맷으로 응답됩니다.  
항목은 다음과 같습니다.

Fields              | Description
:-------------------| :--------------------------------------------------------
token_type          |  항상 "bearer"값으로 설정됨
access_token        |  Melon Resource API request의 권한을 부여받은 토큰
expires_in          |  access_token의 lifetime(초)
refresh_token       |  access_token을 갱신할 수 있는 토큰. refresh_token은 사용자의 파기 요청(비밀번호 변경, 휴면전환, 탈퇴 등)이 있기전까지 유효함.

![#f03c15](https://placehold.it/15/f03c15/000000?text=+)
`파트너는 발급된 access_token 및 refresh_token을 보안적으로 안전한 곳에 보관해야 합니다.(DB 보관 권장)`  
![#f03c15](https://placehold.it/15/f03c15/000000?text=+)
`사용자의 access_token이 만료된다면, 해당 사용자의 refresh_token을 통해 access_token을 갱신합니다. 만약 refresh_token이 유효하지 않은 상태라면, Step 1부터 반복하여 새로운 refresh_token을 발급받습니다.`

(Example)  
```json
{
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJZalk1TmpKak1qWTJaVEl3TVRaaE56RHpPWTZNaEtzYVdvNEhRcDhlNHpCU29PTlkza1BZIiwic2NvcGUiOlsic3RyZWFtaW5nIiwidXNlci1wbGF5bGlzdC1yZWFkIl0sIm5hbWUiOiLshqHqt7zsmrEiLCJwcmVmZXJyZWRfdXNlcm5hbWUiOiLrp4jsi5zrp4jroIEiLCJleHAiOjE1MjMzMjg0NTIsImF1dGhvcml0aWVzIjpbIlJPTEVfVVNFUiJdLCJqdGkiOiJjYzEwNjA5ZS1mOTNiLTRjMDktYjllZC01MTYzZTJlMmVmZWUiLCJjbGllbnRfaWQiOiIxMjM0In0.5Kw6AhnhHdl_4eGmi-DoiA59jYibYap63bSogUmZODg",
    "token_type": "bearer",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJZalk1TmpKak1qWTJaVEl3TVRaaE56RHpPWTZNaEtzYVdvNEhRcDhlNHpCU29PTlkza1BZIiwic2NvcGUiOlsic3RyZWFtaW5nIiwidXNlci1wbGF5bGlzdC1yZWFkIl0sImF0aSI6ImNjMTA2MDllLWY5M2ItNGMwOS1iOWVkLTUxNjNlMmUyZWZlZSIsIm5hbWUiOiLshqHqt7zsmrEiLCJwcmVmZXJyZWRfdXNlcm5hbWUiOiLrp4jsi5zrp4jroIEiLCJhdXRob3JpdGllcyI6WyJST0xFX1VTRVIiXSwianRpIjoiZmYyMjBlOTgtN2M0Ni00NmUyLWFiZmYtMmY5NTIwOTViYjNiIiwiY2xpZW50X2lkIjoiMTIzNCJ9.GMyrKlqPoS9Z6wV3wzuFmR4wJYgKReeb3VtwcnW3Yw0",
    "expires_in": 299,
}
```  

![#f03c15](https://placehold.it/15/f03c15/000000?text=+)
`파트너는 식별되지 않은 필드에 대해서는 무시하셔도 됩니다.`  


### Melon Resource Apis 이용  
파트너앱에서 사용자의 access_token을 정상적으로 발급 받았다면, 해당 사용자의 정보를 기반으로 Melon Resource Apis를 사용할 수 있습니다.  
이를 위해 Resource Apis 호출 시 access_token 정보를 HTTP 헤더 > *Authorization: Bearer*(권장) 혹은 *query string*에 포함하여 전달해야 합니다.  

(Example)  
Melon Resource Apis 중, "streaming endpoint"를 "Authorization: Bearer" Http 헤더를 이용하여 호출하는 방법은 다음과 같습니다.  
```
GET /oauth/delivery/streaming_path.json
Host: alliance.melon.com
Authorization: Bearer <access_token>
```

동일한 Api에 대해 access_token을 query string 파라미터로 전달하는 방법입니다.  
```
GET /oauth/delivery/streaming_path.json?access_token=<access_token>
Host: alliance.melon.com
```

(Melon Resource Apis 에러 응답)  
Melon Resource Api request 시 access_token이 비정상적이거나 만료된 경우, Http status code `401` 혹은 `403`이 반한됩니다.  
이 경우, 파트너는 해당 사용자에 맞는 refresh_token을 이용해 access_token을 갱신하거나,
refresh_token이 유효하지 않다면 OAuth2 access tokens 발급 flow > Step1 부터 반복하여 access_token을 발급 받은 후, Melon Resource Apis를 호출 해야 합니다.
