# Certbot 자동갱신 및 설정

## 3줄 요약

- `certbot run`으로 인증서를 처음 발급할 때 성공했다면 발급 시에 사용한 명령어가 시스템에 설정파일로 저장된다.
- 추후에 `certbot renew`으로 인증서를 갱신할 때 해당 설정 파일을 사용한다.
- 그래서 서버 호스트 환경이 변하면 인증서 자동 갱신이 불가능할 수 있다.

## 문제 상황

- 3개월마다 https 보안 인증서를 발급받아야한다.
- 그런데 단순하게 `certbot renew`명령어를 실행하면 아래와 같은 에러가 발생하고 인증서 갱신이 이루어지지 않는다. 
```
Error while running apache2ctl graceful.
httpd not running, trying to start
Action 'graceful' failed.
The Apache error log may have more information.

(98)Address already in use: AH00072: make_sock: could not bind to address [::]:443
(98)Address already in use: AH00072: make_sock: could not bind to address 0.0.0.0:443
no listening sockets available, shutting down
AH00015: Unable to open logs

Unable to restart apache using ['apache2ctl', 'graceful']
```

- 3개월마다 콘솔에 접속하여 실행하는 것은 비효율적이고 사용자 경험에도 좋지 않으므로 문제 해결 방안을 찾아야한다.

## 문제 분석

### 로그 확인

- 이전에 구축된 시스템이라서 초기 인증서 갱신 설정이 어떻게 되어 있는지는 알기 어렵다.
- 그렇지만 에러 로그를 통해서 `certbot`이 `apache2` 웹서버 실행 명령을 수행하고 있는 것을 확인할 수 있다. 
- `certbot`은 웹서버와 독립적으로 동작할텐데 왜 웹서버 실행 명령을 의존성으로 가지고 있을까

### `certbot --help` 확인

- `certbot --help`를 통해 확인할 수 있는 옵션 중에 플러그인 관련 항목이 있다.
```
  --apache          Use the Apache plugin for authentication & installation
  --standalone      Run a standalone webserver for authentication
  --nginx           Use the Nginx plugin for authentication & installation
  --webroot         Place files in a server's webroot folder for authentication
  --manual          Obtain certificates interactively, or using shell script hooks
```
- `apache`를 확인할 수 있는 부분인 이곳이므로 certbot 플러그인과 관련하여 더 알아본다.

### `certbot` 공식 문서 확인

- [docs](https://eff-certbot.readthedocs.io/en/stable/using.html#getting-certificates-and-choosing-plugins)
- `certbot`은 인증서 발급시에 아래 두 단계를 거친다.
  1. 사용자 인증을 거친 후 인증서를 발급받는다. 인증서를 로컬에 저장하고 **자동으로 갱신**하도록 시스템에 등록한다.
  2. **추가적으로** 여러 웹서버를 지원하도록 인증서를 설치한다.

- 위 단계 중에서 2번째에 해당하는 부분에서 문제가 발생하는 것으로 보인다.

## 문제 해결

### 플러그인 변경

- 플러그인이 문제인 것으로 보이니 테스트로 실행하여 확인한다.
- `certbot renew --debug --standalone --dry-run`
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/my.domain.com.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Simulating renewal of an existing certificate for my.domain.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Congratulations, all simulated renewals succeeded: 
  /etc/letsencrypt/live/my.domain.com/fullchain.pem (success)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```
- 에러 없이 종료되었다.

### 웹서버 지원
- 인증서가 변경되면 웹서버가 재시작하여 새로운 인증서를 호스트하도록 변경해야한다.
- 해당 사항은 `pre-hook`, `post-hook`, `deploy-hook`으로 설정이 가능하다.
- 위 값은 `certbot renew` 실행시에 인자로 넘기면 해당 도메인 갱신시에만 실행되고 `/etc/letsencrypt/renewal-hooks/` 폴더에 저장하면 모든 도메인 갱신마다 실행한다.
- 현재 시스템은 `apache`혹은 `nginx`와 같은 웹서버 대신 파이썬을 통해서 호스트하고 있으므로 기존 `certbot`로직의 2번째를 지원하도록 수정해야한다.
- 그러므로 실행중인 웹서버 중지 및 실행을 수행하는 스크립트 코드를 작성하여 직접 등록한다.


## 인증서 자동 갱신

### 시스템 타이머 확인
- 위 공식문서에 따르면 `certbot`으로 인증서를 발급받으면 자동으로 갱신하도록 설정한다고 한다.
- 해당 파일은 시스템마다 다르지만 우분투에서는 이와 같이 확인할 수 있다. `sudo systemctl list-timers | grep certbot`
```
Mon 2023-10-16 15:24:00 KST  4h 23min left Mon 2023-10-16 05:36:14 KST  5h 24min ago snap.certbot.renew.timer     snap.certbot.renew.service
```
- 그리고 해당 설정 파일은 아래에서 확인 가능하다. `ls /etc/systemd/system/snap.certbot.renew.*`
```
/etc/systemd/system/snap.certbot.renew.service  /etc/systemd/system/snap.certbot.renew.timer`
```
- 서비스 파일을 확인하면 아래와 같은 명령어로 인증서 갱신을 수행하는 것을 확인할 수 있다.
```
ExecStart=/usr/bin/snap run --timer="00:00~24:00/2" certbot.renew
```


### 설정 파일
- `certbot new` 명령어로 인증서를 발급할 때 사용한 플러그인 정보가 시스템에 설정 파일로 저장되어 있다.
- 그리고 `certbot renew`수행시에 해당 설정 파일을 참조한다.
- 그래서 문제 해결 방안을 알아도 `certbot renew --dry-run`을 수행할 때 `apache2`에러가 발생한다.
- 따라서 설정 파일을 변경할 필요가 있다.

### 설정 파일 수정
- 도메인 별 설정 파일은 이 폴더에 위치한다. `ls /etc/letsencrypt/renewal/`
```
my.domain.com.conf
```
- 해당 파일 내용은 아래와 같다.
```
# renew_before_expiry = 30 days
version = 2.7.1
archive_dir = /etc/letsencrypt/archive/my.domain.com
cert = /etc/letsencrypt/live/my.domain.com/cert.pem
privkey = /etc/letsencrypt/live/my.domain.com/privkey.pem
chain = /etc/letsencrypt/live/my.domain.com/chain.pem
fullchain = /etc/letsencrypt/live/my.domain.com/fullchain.pem

# Options used in the renewal process
[renewalparams]
account = secret_hash
authenticator = apache
server = https://acme-v02.api.letsencrypt.org/directory
key_type = secret_key_type
installer = apache
```

- 인증기와 인스톨러가 `apache`로 설정되어 있다. 위 파일을 아래와 같이 변경한다.
```
# renew_before_expiry = 30 days
version = 2.7.1
archive_dir = /etc/letsencrypt/archive/my.domain.com
cert = /etc/letsencrypt/live/my.domain.com/cert.pem
privkey = /etc/letsencrypt/live/my.domain.com/privkey.pem
chain = /etc/letsencrypt/live/my.domain.com/chain.pem
fullchain = /etc/letsencrypt/live/my.domain.com/fullchain.pem

# Options used in the renewal process
[renewalparams]
account = secret_hash
authenticator = standalone
server = https://acme-v02.api.letsencrypt.org/directory
key_type = secret_key_type
installer = standalone
pre_hook = stop.sh
post_hook = start.sh
```

- 그리고 확인하면 동작한다. `sudo certbot renew --debug --dry-run`
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/my.domain.com.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Simulating renewal of an existing certificate for my.domain.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Congratulations, all simulated renewals succeeded: 
  /etc/letsencrypt/live/my.domain.com/fullchain.pem (success)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```