# Integrating with Melon using Client Credentials Grant

### Prerequisites
**authorization credentials 생성**  
클라이언트 혹은 제휴사가(이하 '파트너') OAuth2를 이용해 Melon Resource API를 사용하려면,  
먼저 파트너를 식별할 수 있는 authorization credentials(client_id/client_secret 기반)를 발급 받아야 합니다.  
현재, 수동적인 프로세스를 통해 authorization credentials을 발급하고 있습니다.  
아래 내용을 포함한 정보를 담당자(파트너사와 협업하는 Melon의 담당자)에게 전달해주시면 됩니다.  

1. 애플리케이션 유형 : (웹애플리케이션/Android/Chrome앱/iOS/기타)
2. 파트너 담당자 이메일 주소 :
3. 사용자에게 표시 되는 애플리케이션 명(서비스 명) :
4. 파트너 서비스 대표 URL : 복수설정 가능합니다. ex) https://your.site.com)


![#f03c15](https://placehold.it/15/f03c15/000000?text=+) `발급된 authorization credential은 외부 유출이 되지 않도록 별도 보관하셔야 합니다.  
(만약, credential이 유출되었다고 판단되는 경우 담당자에게 연락주시면 client_secret을 변경해드립니다.)`


**scope 정의**  
Melon Resource Apis는 scope을 통해 세부적으로 제어 됩니다.  
파트너는 scope에 대한 사용 여부를 멜론 담당자로부터 승인 받아야 하며, 승인된 scope의 resource API만 접근 허용됩니다.  

OAuth 2.0 integration을 진행하기 전에, 멜론의 제휴담당자 혹은 기획담당자와 협의하여 사용될 서비스에 대한 scope을 식별하시길 권장합니다.  

현재 정의된 Melon OAuth 2.0 API scope은 다음과 같습니다.

Scope               | Description                             | Required
:-----------------------| :---------------------------------------| :-----------
streaming               | 음원재생 권한                              | No
user-pay-amount-read    | 사용자 "결제 요청 금액" read 권한(원스토어)     | No  

### OAuth2 access tokens 발급 flow
Melon의 OAuth2 access token을 발급받는 과정은 크게 *2단계*로 나뉩니다.

**Step 1: client_credentials를 이용한 access_token 획득(by Token Endpoint)**  
파트너는 Token Endpoint request를 통해 access_token을 발급받을 수 있습니다.  

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
grant_type          | **Yes**       | "client_credentials" 값을 설정
client_id           | **Yes**       | 파트너 앱의 client_id(발급받은 authorization credential)
client_secret       | **Yes**       | 파트너 앱의 client_secret(발급받은 authorization credential)
scope               | **Yes**       | 파트너 앱이 승인 받은 Melon resource의 범위 정보(*scope 정의 참고*). 복수 시 space 구분자를 이용.

![#f03c15](https://placehold.it/15/f03c15/000000?text=+)
`파트너 앱의 authorization credintial 정보는 Authorization: Basic (Http Basic)으로도 전달이 가능합니다.`

(Example)
```
POST /oauth/token HTTP/2
Host: auth.melon.com
Content-type: application/x-www-form-urlencoded

grant_type=client_credentials&
client_id=partner_client_id&
client_secret=partner_client_secret&
scope=user-pay-amount-read
```

**Step 2: Token Endpoint 응답 처리**  
Token Endpoint의 request가 정상적이면, short-lived access_token과 TTL이 JSON 포맷으로 응답됩니다.  
항목은 다음과 같습니다.

Fields              | Description
:-------------------| :--------------------------------------------------------
token_type          |  항상 "bearer"값으로 설정됨
access_token        |  Melon Resource API request의 권한을 부여받은 토큰
expires_in          |  access_token의 lifetime(초)

![#f03c15](https://placehold.it/15/f03c15/000000?text=+)
`파트너는 발급된 access_token을 보안적으로 안전한 곳에 보관해야 합니다.(DB 보관 권장)`  
![#f03c15](https://placehold.it/15/f03c15/000000?text=+)
`access_token이 만료되거나 유효하지 않은 상태라면, Step 1부터 반복하여 새로운 access_token을 발급받습니다.`

(Example)  
```json
{
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzY29wZSI6WyJ1c2VyLXBheS1hbW91bnQtcmVhZCJdLCJleHAiOjE1MjY2MzExODEsImF1dGhvcml0aWVzIjpbIlJPTEVfVFJVU1RFRF9DTElFTlQiLCJST0xFX0NMSUVOVCJdLCJqdGkiOiIzMzZiNmUyMy1mY2FhLTRiZjUtYmJiYy02MGExOWU0YmJiODciLCJjbGllbnRfaWQiOiJjM2QwMmI3OGYzYWU2ZDk0YTI3MTYzNTcwMjNmYjkyMyJ9.cztNNtGyRRskEhRUWGu93Y8O-W4UNCnunaJFKDNsHkQ",
    "token_type": "bearer",
    "expires_in": 289,
    "scope": "user-pay-amount-read"
}
```  

![#f03c15](https://placehold.it/15/f03c15/000000?text=+)
`파트너는 식별되지 않은 필드에 대해서는 무시하셔도 됩니다.`  


### Melon Resource Apis 이용  
파트너앱에서 access_token을 정상적으로 발급 받았다면, Scope 정보를 기반으로 Melon Resource Apis를 사용할 수 있습니다.  
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
Melon Resource Api request 시 access_token이 비정상적이거나 만료된 경우, Http status code `401` 혹은 `403`이 반환됩니다.  
이 경우, OAuth2 access tokens 발급 flow > Step1 부터 반복하여 access_token을 발급 받은 후, Melon Resource Apis를 재호출 해야 합니다.
