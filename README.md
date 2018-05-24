# Integrating with Melon using Standard OAuth2

이 문서는 OAuth2를 통해 Melon의 resource를 사용하기 위한 integration 가이드이며, 표준 Full Spec은 [이곳](https://oauth.net/2/)에서 확인 가능합니다.  
Melon에서는 일반적으로 [Authorization Code Grant](https://tools.ietf.org/html/rfc6749#section-4.1) 혹은 [Client Credentials Grant](https://tools.ietf.org/html/rfc6749#section-4.4) Grant Type의 인증체계를 사용합니다.

## Authorization Code Grant
* 제휴사가 사용자 기반의 Melon resource 접근이 필요한 경우.
* 상세 링크 : - [Authorization Code Grant](authorization_code/README.md)

## Client Credentials Grant
* 사용자 기반의 resource가 아닌, 제휴사가 클라이언트라는 개념으로 Melon의 resource 접근이 필요한 경우.
* 상세 링크 : - [Client Credentials Grant](client_credentials/README.md)
