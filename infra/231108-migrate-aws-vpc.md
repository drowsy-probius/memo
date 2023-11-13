# AWS VPC 이전

기존 인프라 상황에서의 EIP 고갈과 안전하지 않은 보안 환경 문제로 새로운 VPC를 생성하여 기존 인스턴스를 이전해야 하는 상황이 발생하였다.

그래서 기존 동작중인 인스턴스를 모두 private 서브넷에 위치하고 nginx서버가 구동될 public 서브넷에 위치한 인스턴스를 생성하여 reverse proxy로 웹 서비스를 호스트하고자 한다.

## 새 VPC 생성

기존에 사용하던 `default` VPC 대신에 `root VPC`를 서브넷을 `10.0.0.0/16`으로 설정하여 생성하였다.

두개의 AZ로 구성하였고 각각을 `a`, `b` 리전에 두었다. 각 리전마다 private, public 서브넷을 구성하였다. private 서브넷은 `10.0.0.0/20` ~ `10.0.127.255/20` 대역으로, public 서브넷은 `10.0.128.0/20` ~ `10.0.255.255/20` 대역으로 설정하였다.

라우팅 테이블은 리전이 아닌 private, public으로 구분하여 두개의 라우팅 테이블을 생성하였다. private 라우팅 테이블에는 퍼블릭 주소를 할당(연결 유형 public + EIP할당)한 NAT를, public 라우팅 테이블에서는 IGW를 연결하였다. private rtb에 할당되는 NAT에 퍼블릭 주소가 할당되어야 하는 이유는 외부로 나가는 트래픽이 필요하기 때문이다.

각각의 라우팅 테이블에서는 `10.0.0.0/16` 주소는 `local`으로 나머지 `0.0.0.0/0` 주소는 각각 `NAT`, `IGW`으로 나가도록 설정하였다.

인스턴스 내에서 방화벽을 설정하지 않고 사용하기 위하여 보안 그룹을 설정하였다. 보안 그룹은 private 인스턴스용 한개와 public 인스턴스용 여러개를 생성하였다. private 보안 그룹은 inbound 허용은 `10.0.0.0/16`만, outbound는 `0.0.0.0/0`으로 설정하였다. public 보안 그룹은 inbound 허용은 각 인스턴스의 앱에 따라서 설정하였고 outbound는 `0.0.0.0/0`으로 설정하였다. 


## 인스턴스 마이그레이션

기존 인스턴스가 다른 vpc에 생성되어있기 때문에 새로 생성한 vpc로 이전한다면 동일한 인스턴스를 새로 생성할 필요가 있다. 

내부 파일을 백업 후 복원하는 것보다는 동작 중인 파일 시스템 전체를 이미지로 저장한 뒤 이미지로 인스턴스를 생성하는 것이 더 빠르고 안정적이라 판단하였다.

기존 인스턴스로부터 AMI이미지를 생성하려면 `EC2 > 인스턴스 > 인스턴스 체크박스 한개 선택 > 작업 > 이미지 및 템플릿 > 이미지 생성`을 선택한다. 인스턴스의 디스크 볼륨 크기에 따라서 시간이 소요될 수 있다. 복제된 이미지와 이전 인스턴스의 내용이 동일함을 보장하기 위해서 인스턴스를 **중지**하고 수행해야한다. 

생성한 AMI이미지로부터 새로운 인스턴스를 생성하려면 `EC2 > 인스턴스 > 인스턴스 시작 > Application and OS Images (Amazon Machine Image) > 내 AMI` 에서 생성한 이미지를 선택하면 된다. 이때 아직 생성이 완료되지 않은 이미지를 선택한다면 인스턴스 생성시에 에러가 발생한다.

인스턴스 생성 시에 네트워크 설정에서 방금 생성한 `root` VPC를 선택하고 기존 서비스 인스턴스이므로 private 서브넷을 선택한다. 보안 그룹도 private에 생성한 것으로 선택한다.

## nginx 인스턴스 및 bastion 인스턴스 생성

public 서브넷에 위치한 인스턴스는 두개 생성할 것이다. nginx는 모든 웹 트래픽을 통과하도록 할 것이고 bastion 인스턴스는 private 인스턴스의 ssh 접속을 위한 것이다. 두개의 인스턴스를 분리한 이유는 만일 nginx 인스턴스에 두개의 기능을 구현한다면 nginx 인스턴스가 다운될 때 내부 인스턴스로 ssh 접속이 불가능해질 것이다.

nginx가 위치할 인스턴스는 고정 IP가 할당되어야 하므로 EIP를 할당한다. 하지만 bastion 인스턴스는 웹 콘솔에 접속하여 ip를 확인할 수 있으므로 EIP를 할당하지 않아도 된다.

기존의 웹 서비스들로 향하던 모든 트래픽이 nginx을 통과할 것이므로 nginx 인스턴스의 유형은 기존 인스턴스 중에서 가장 큰 유형인 `t2.large`로 설정하였다. 

bastion 인스턴스는 현재로서는 ssh 접속을 위한 인스턴스이므로 비용 절감을 위하여 작은 유형인 `t2.micro`로 설정하였다.

## 가비아 DNS 설정

얘네는 변경사항 배포가 느리다. DNS 엔트리 각 항목의 우선순위가 어떻게 되는지 모르겠다. 엔트리 순서 변경이 불가능하다.

`*.our_domain.com`의 A레코드로 nginx 인스턴스에 연결된 EIP 값을 설정하야여한다.

## nginx 설정

reverse proxy의 대상으로 `10.0.0.0/16`대역의 주소를 할당하여야 한다.

nginx 설정으로 buffer 크기 및 연결 시간 등을 넉넉하게 잡아주어야 한다. 

```nginx
client_header_timeout 5m;   #default 15
client_body_timeout 5m;     #default
client_max_body_size 1024M;

keepalive_timeout 20m;

proxy_connect_timeout 5m; #default 60초
proxy_send_timeout 5m;    #default 60초
proxy_read_timeout 5m;    #default 60초
send_timeout 5m;          #default 60초

proxy_buffering   on;
proxy_buffer_size    1024k;
proxy_buffers        1024   1024k;
client_body_buffer_size 1024k;
proxy_busy_buffers_size 1024k;


fastcgi_buffers 16 16k;
fastcgi_buffer_size 32k;
fastcgi_read_timeout 5m;  #default 60초
```

## 아쉬운점

terraform 등의 IaaC 툴을 사용하여 추후 유지보수를 원할하게 할 수 있으면 좋겠다.

사용자별 IAM설정을 철저하게 하면 좋겠다.
