# Inception 개념 및 과제 구조 정리

> 이 문서는 지금까지의 대화 내용과 Inception 과제 명세를 바탕으로, 과제의 목적과 Docker 관련 핵심 개념을 처음부터 연결해서 이해할 수 있도록 정리한 학습 노트이다.

---

# 1. Inception 과제의 목적

Inception은 단순히 WordPress 사이트 하나를 띄우는 과제가 아니다.

과제의 핵심 목적은 **Docker를 사용해 작은 서버 인프라를 직접 구성하면서 시스템 관리(System Administration), 컨테이너, 네트워크, 저장소, 보안 설정의 기본 구조를 이해하는 것**이다.

최종적으로는 가상 머신(VM) 내부에서 Docker를 사용하여 다음 서비스를 각각 별도의 컨테이너로 구성한다.

```text
Physical PC
└─ Virtual Machine
   └─ Docker
      ├─ NGINX Container
      ├─ WordPress + PHP-FPM Container
      └─ MariaDB Container
```

이 세 컨테이너는 서로 독립적으로 실행되지만, Docker Network를 통해 통신하고 Docker Volume을 통해 데이터를 영속적으로 보존한다.

전체 요청 흐름은 다음과 같다.

```text
사용자 브라우저
      │
      │ HTTPS / TLS / Port 443
      ▼
   NGINX
      │
      │ FastCGI
      ▼
WordPress + PHP-FPM
      │
      │ SQL / DB Connection
      ▼
   MariaDB
```

---

# 2. NGINX란?

NGINX는 웹 서버(Web Server)이다.

쉽게 말하면 사용자가 웹사이트에 접속했을 때 **가장 먼저 요청을 받아주는 입구** 역할을 한다.

예를 들어 사용자가 브라우저에서 다음 주소로 접속한다고 하자.

```text
https://login.42.fr
```

브라우저의 요청은 가장 먼저 NGINX에 도착한다.

```text
Browser
   │
   ▼
 NGINX
```

NGINX의 주요 역할은 다음과 같다.

- HTTP/HTTPS 요청 수신
- 정적 파일 제공
- 다른 서버나 프로세스로 요청 전달
- TLS/SSL 처리
- Reverse Proxy 역할
- PHP 처리가 필요할 경우 PHP-FPM으로 요청 전달

Inception에서는 NGINX가 외부에서 접근 가능한 유일한 진입점이며, 포트 443만 사용해야 한다.

---

# 3. WordPress란?

WordPress는 PHP로 작성된 웹 애플리케이션이다.

직접 웹사이트를 만든다면 다음 기능을 직접 구현해야 할 수 있다.

- 로그인
- 게시글
- 댓글
- 사용자 관리
- 관리자 페이지
- 테마
- 플러그인
- 페이지 관리

WordPress는 이런 기능을 이미 제공한다.

즉,

```text
NGINX = 요청을 받는 웹 서버
WordPress = 실제 웹사이트 기능을 담당하는 애플리케이션
```

이라고 이해하면 된다.

WordPress는 PHP 코드로 작성되어 있기 때문에 PHP 코드를 실행해줄 프로그램이 필요하다.

그 역할을 하는 것이 PHP-FPM이다.

---

# 4. PHP-FPM이란?

PHP-FPM은 PHP FastCGI Process Manager의 약자이다.

쉽게 말하면 **PHP 코드를 실제로 실행해주는 프로그램**이다.

NGINX는 PHP 코드를 직접 실행하지 못한다.

예를 들어 다음과 같은 PHP 코드가 있다고 하자.

```php
<?php
echo "<h1>Hello</h1>";
?>
```

NGINX는 이 PHP 코드를 직접 해석하지 않고 PHP-FPM에게 전달한다.

```text
NGINX
   │
   │ FastCGI
   ▼
PHP-FPM
   │
   │ PHP 실행
   ▼
HTML 결과 생성
```

PHP-FPM이 PHP 코드를 실행한 뒤 생성한 결과를 다시 NGINX에 전달한다.

전체 흐름은 다음과 같다.

```text
Browser
   ↓
NGINX
   ↓
PHP-FPM
   ↓
WordPress PHP 코드 실행
   ↓
HTML 생성
   ↓
NGINX
   ↓
Browser
```

Inception에서는 WordPress와 PHP-FPM이 같은 컨테이너에 들어간다.

---

# 5. MariaDB란?

MariaDB는 관계형 데이터베이스 관리 시스템(RDBMS)이다.

WordPress는 다음과 같은 데이터를 저장해야 한다.

- 사용자 정보
- 게시글
- 댓글
- WordPress 설정
- 카테고리
- 권한
- 관리자 계정 정보

이 데이터는 MariaDB에 저장된다.

구조는 다음과 같다.

```text
WordPress
    │
    │ SQL
    ▼
MariaDB
```

예를 들어 WordPress 내부에서는 개념적으로 다음과 같은 SQL이 실행될 수 있다.

```sql
SELECT * FROM posts;
INSERT INTO users (...);
UPDATE options SET ...;
```

MariaDB는 MySQL과 매우 유사한 관계형 데이터베이스이며 WordPress에서 사용할 수 있다.

---

# 6. Docker란?

Docker는 프로그램과 그 프로그램이 실행되기 위해 필요한 환경을 격리하여 실행하고 관리할 수 있게 해주는 플랫폼이다.

예를 들어 어떤 프로그램이 다음 환경을 필요로 한다고 하자.

```text
애플리케이션
├─ Debian 기반 환경
├─ PHP
├─ 특정 라이브러리
├─ 설정 파일
└─ 프로그램 파일
```

이것을 Docker Image 형태로 만들어두면 Docker가 설치된 다른 환경에서도 거의 동일한 실행 환경을 재현할 수 있다.

Docker의 큰 목적 중 하나는 다음 문제를 줄이는 것이다.

```text
"내 컴퓨터에서는 되는데 다른 컴퓨터에서는 안 됩니다."
```

Docker는 애플리케이션 실행 환경을 함께 구성할 수 있기 때문에 개발 환경과 실행 환경의 차이를 줄여준다.

다만 Docker는 완전한 운영체제를 새로 하나 띄우는 VM과는 다르다.

---

# 7. Docker Container란?

Container는 Docker Image를 기반으로 실제 실행되고 있는 격리된 환경이다.

예를 들어 Inception에서는 다음과 같이 각각의 서비스가 별도 컨테이너에서 실행된다.

```text
Docker
├─ NGINX Container
├─ WordPress Container
└─ MariaDB Container
```

Container는 하나의 독립된 컴퓨터처럼 보이지만 실제로는 호스트 Linux Kernel을 공유한다.

즉 다음과 같이 이해할 수 있다.

```text
Docker = 컨테이너를 만들고 관리하는 플랫폼
Container = Docker가 실제로 실행한 격리된 환경
```

---

# 8. Dockerfile이란?

Dockerfile은 Docker Image를 만들기 위한 설계도이다.

예를 들어 NGINX용 Dockerfile은 개념적으로 다음과 같은 내용을 가진다.

```dockerfile
FROM debian:...

RUN apt update
RUN apt install -y nginx

COPY nginx.conf /etc/nginx/nginx.conf

CMD ["nginx", "-g", "daemon off;"]
```

의미는 다음과 같다.

```text
1. Debian 기반 환경에서 시작
2. nginx 설치
3. 설정 파일 복사
4. 컨테이너 시작 시 nginx 실행
```

핵심 관계는 다음과 같다.

```text
Dockerfile
    │
    │ docker build
    ▼
Docker Image
    │
    │ docker run
    ▼
Container
```

즉,

```text
Dockerfile = 설계도
Image      = 설계도로 만들어진 실행 템플릿
Container  = Image가 실제 실행된 상태
```

라고 이해하면 된다.

---

# 9. Docker Image란?

Docker Image는 Container를 만들기 위한 실행 템플릿이다.

하나의 Image에서 여러 Container를 생성할 수도 있다.

```text
NGINX Image
├─ NGINX Container 1
├─ NGINX Container 2
└─ NGINX Container 3
```

프로그래밍 관점에서 아주 느슨하게 비유하면 다음 관계와 비슷하다.

```text
Class → Object
Image → Container
```

완전히 같은 개념은 아니지만 처음 이해할 때 도움이 되는 비유이다.

---

# 10. venv와 Docker의 차이

Python의 venv도 환경을 분리하지만 Docker와 격리 범위가 다르다.

## venv

```text
Python 프로젝트
└─ venv
   ├─ Python package A
   ├─ Python package B
   └─ Python package C
```

venv는 주로 Python 패키지와 의존성을 분리한다.

## Docker

```text
Container
├─ 프로그램
├─ 시스템 라이브러리
├─ 설정 파일
├─ 네트워크 환경
└─ 파일시스템 환경
```

Docker는 훨씬 넓은 실행 환경을 격리한다.

단순화하면 다음과 같이 볼 수 있다.

```text
venv
= Python 패키지 수준 격리

Docker Container
= 애플리케이션 실행 환경 수준 격리

VM
= 운영체제 전체 수준 격리
```

---

# 11. Docker와 VM의 차이

VM은 운영체제 전체를 가상화한다.

```text
Physical PC
└─ Host OS
   └─ VMware / VirtualBox
      └─ Guest OS
         ├─ Guest Kernel
         └─ Application
```

각 VM은 독립적인 운영체제와 Kernel을 가진다.

반면 Docker Container는 Host Kernel을 공유한다.

```text
Host Linux
├─ Linux Kernel
│
├─ NGINX Container
├─ WordPress Container
└─ MariaDB Container
```

따라서 Docker Container는 일반적으로 VM보다 가볍고 시작 속도가 빠르다.

## 비교

| 항목 | VM | Docker Container |
|---|---|---|
| 가상화 범위 | 운영체제 전체 | 애플리케이션 실행 환경 |
| Kernel | VM마다 별도 | Host Kernel 공유 |
| 크기 | 비교적 큼 | 비교적 작음 |
| 시작 속도 | 비교적 느림 | 빠름 |
| 리소스 사용 | 큼 | 작음 |
| 격리 방식 | 하드웨어/OS 가상화 | Namespace, cgroup 등 |

---

# 12. Docker가 Host Kernel을 공유한다는 의미

Container마다 별도의 Kernel이 있는 것이 아니다.

```text
실제
= Kernel 1개

Container가 보는 모습
= 자기만의 시스템 환경이 있는 것처럼 보임
```

이것을 가능하게 하는 핵심 Linux 기능 중 하나가 Namespace이다.

---

# 13. Linux Namespace란?

Linux Namespace는 같은 Kernel을 사용하는 여러 프로세스 그룹에게 서로 다른 시스템 환경을 보는 것처럼 만들어주는 기능이다.

핵심 개념은 다음과 같다.

> Namespace는 Kernel 자체를 여러 개 만드는 것이 아니라, 프로세스가 Kernel 자원을 바라보는 관점을 분리한다.

예를 들어 같은 Host Kernel 위에서:

```text
Linux Kernel
│
├─ Container A
│  ├─ 자기 PID 공간
│  ├─ 자기 Network
│  └─ 자기 Mount 환경
│
└─ Container B
   ├─ 자기 PID 공간
   ├─ 자기 Network
   └─ 자기 Mount 환경
```

처럼 동작할 수 있다.

## 주요 Namespace

| Namespace | 역할 |
|---|---|
| PID | 프로세스 ID와 프로세스 트리 격리 |
| NET | 네트워크 인터페이스, IP, 포트, 라우팅 격리 |
| MNT | Mount와 파일시스템 관점 격리 |
| UTS | hostname, domain name 격리 |
| IPC | 일부 IPC 자원 격리 |
| USER | UID/GID 매핑 및 격리 |
| CGROUP | cgroup 관점 격리 |

---

# 14. PID Namespace

PID Namespace는 프로세스 목록과 PID 공간을 분리한다.

예를 들어 Host에서는:

```text
PID 1200 nginx
PID 1300 php-fpm
PID 1400 mariadb
```

로 보이지만 각각의 컨테이너 안에서는:

```text
NGINX Container
PID 1 nginx

WordPress Container
PID 1 php-fpm

MariaDB Container
PID 1 mariadbd
```

처럼 보일 수 있다.

Container 내부에서 PID 1은 매우 중요하다.

보통 Container의 주요 프로세스가 PID 1이며 이 프로세스가 종료되면 Container도 종료된다.

Inception에서 `tail -f`, `sleep infinity`, `while true` 같은 방식으로 컨테이너를 억지로 살아 있게 만드는 방법이 금지되는 이유와도 연결된다.

---

# 15. Network Namespace

Network Namespace는 다음 네트워크 자원을 분리한다.

- IP 주소
- Network interface
- Routing table
- Port
- Firewall rule

그래서 여러 Container가 각자 독립적인 IP를 가진 것처럼 동작할 수 있다.

예:

```text
NGINX       172.18.0.2
WordPress   172.18.0.3
MariaDB     172.18.0.4
```

또한 서로 다른 Container가 같은 내부 Port 번호를 사용할 수도 있다.

```text
Container A → Port 80
Container B → Port 80
```

각자 다른 Network Namespace이기 때문에 충돌하지 않는다.

---

# 16. Mount Namespace

Mount Namespace는 각 프로세스가 보는 파일시스템 구조를 분리한다.

Container 안에서:

```bash
ls /
```

를 실행하면 일반 Linux 시스템처럼:

```text
/bin
/etc
/usr
/var
/tmp
```

등이 보인다.

하지만 이것은 VM처럼 별도의 Kernel이 있는 것이 아니라, 해당 Container에 특정 파일시스템 tree를 보이도록 구성한 것이다.

Docker Volume도 특정 경로에 저장공간을 mount한다는 점에서 이 개념과 연결된다.

---

# 17. Namespace와 cgroup의 차이

두 개념은 자주 같이 등장한다.

```text
Namespace
= 무엇을 볼 수 있는가

cgroup
= 얼마나 사용할 수 있는가
```

예를 들어:

```text
PID Namespace
→ 다른 Container의 프로세스를 보이지 않게 함

Network Namespace
→ 별도의 IP/Port 환경처럼 보이게 함

cgroup
→ CPU, RAM 등의 사용량을 제한하고 관리
```

따라서 Docker Container는 대략 다음 기술 조합으로 이해할 수 있다.

```text
Linux Process
+
Namespaces
+
cgroups
+
Filesystem
+
Networking
+
Container Runtime
```

---

# 18. File Descriptor와 Pipe는 공유되는가?

Container들이 같은 Kernel을 사용한다고 해서 File Descriptor나 Pipe가 자동으로 서로 공유되는 것은 아니다.

File Descriptor 번호는 프로세스별 File Descriptor Table에서 관리된다.

```text
Process A의 fd 3
≠
Process B의 fd 3
```

같은 번호라 하더라도 서로 다른 Kernel object를 가리킬 수 있다.

Pipe도 마찬가지다.

```c
pipe(fd);
fork();
```

와 같이 부모-자식 관계에서 동일한 Pipe object에 대한 File Descriptor를 상속하는 경우에는 같은 Pipe를 사용할 수 있다.

하지만 Container A에서 Pipe를 만들었다고 해서 Container B에서 자동으로 그 Pipe를 사용할 수 있는 것은 아니다.

핵심은:

> 같은 Kernel이 관리하지만, Container 간 자원은 기본적으로 격리되어 있다.

---

# 19. C++ Namespace와 Linux Namespace의 차이

이름은 같지만 완전히 다른 개념이다.

## C++ Namespace

C++ Namespace는 이름 충돌을 방지하기 위한 언어 기능이다.

```cpp
namespace A
{
    int value = 10;
}

namespace B
{
    int value = 20;
}
```

사용:

```cpp
A::value
B::value
```

C++ Namespace의 목적은 다음과 같다.

- 이름 충돌 방지
- 코드 구조화
- 라이브러리 구분

예를 들어 `std::cout`에서 `std`가 Namespace이다.

## Linux Namespace

Linux Namespace는 Kernel이 제공하는 시스템 자원 격리 기능이다.

```text
C++ Namespace
= 이름을 구분

Linux Namespace
= 프로세스가 보는 시스템 자원을 구분
```

둘은 개념적으로 전혀 다른 기능이다.

---

# 20. 왜 NGINX, WordPress, MariaDB를 3개 Container로 나누는가?

기술적으로는 하나의 Container에 모두 넣는 것도 가능하다.

```text
Container
├─ nginx
├─ php-fpm
├─ wordpress
└─ mariadb
```

하지만 Docker에서는 일반적으로 서비스를 분리하는 구조를 사용하며 Inception 과제도 이를 요구한다.

분리 이유는 다음과 같다.

## 20.1 장애 격리

MariaDB만 문제가 생겼다면 MariaDB Container만 재시작할 수 있다.

```text
NGINX       정상
WordPress   정상
MariaDB     장애
```

다른 서비스 전체를 같이 재시작할 필요가 없다.

## 20.2 독립적인 업데이트

NGINX 설정만 바뀌었다면 NGINX Image만 다시 build하면 된다.

## 20.3 확장성

WordPress 처리량만 부족하면 WordPress Container만 늘리는 설계가 가능하다.

## 20.4 프로세스 관리

각 Container는 하나의 주요 서비스 프로세스를 중심으로 구성하는 것이 관리하기 쉽다.

```text
NGINX Container
PID 1 = nginx

WordPress Container
PID 1 = php-fpm

MariaDB Container
PID 1 = mariadbd
```

## 20.5 데이터 관리

MariaDB처럼 상태가 중요한 서비스와 NGINX처럼 비교적 stateless한 서비스는 관리 방식이 다르다.

서비스를 분리하면 각각에 맞는 Volume, Network, Restart Policy를 적용하기 쉽다.

---

# 21. Docker Compose란?

Container가 여러 개라면 각각 직접 `docker run` 하는 것은 복잡하다.

Docker Compose는 여러 Container를 하나의 서비스 스택으로 정의하고 관리한다.

예를 들어:

```yaml
services:
  nginx:
    ...

  wordpress:
    ...

  mariadb:
    ...

networks:
  inception:

volumes:
  wordpress_data:
  mariadb_data:
```

다음 명령으로 전체 서비스를 실행할 수 있다.

```bash
docker compose up
```

중요한 점은 Container 자체는 분리되어 있지만 Compose를 통해 하나의 애플리케이션처럼 관리할 수 있다는 것이다.

단, `docker compose up`이 DB transaction과 같은 의미의 atomic operation은 아니다.

예를 들어:

```text
MariaDB 시작 성공
WordPress 시작 성공
NGINX 시작 실패
```

같은 상황은 가능하다.

---

# 22. Docker Volume이 필요한 이유

Container의 writable filesystem은 Container 생명주기에 종속된다.

Container 안에만 데이터를 저장하면 Container 삭제 시 데이터도 사라질 수 있다.

예:

```text
MariaDB Container
└─ /var/lib/mysql
```

Container를 삭제하면 해당 데이터가 사라질 수 있다.

그래서 중요한 데이터는 Volume으로 분리한다.

```text
MariaDB Container
       │
       ▼
   DB Volume
```

Container를 삭제하고 다시 만들어도 Volume을 유지하면 기존 데이터를 사용할 수 있다.

이것을 Persistence, 즉 데이터 영속성이라고 한다.

---

# 23. Inception에서 필요한 Volume

Inception에서는 크게 두 개의 persistent storage가 필요하다.

```text
1. MariaDB Database Volume
2. WordPress Website Files Volume
```

구조:

```text
WordPress Container
       │
       ▼
WordPress Volume

MariaDB Container
       │
       ▼
Database Volume
```

MariaDB Volume에는 DB 데이터가 저장되고 WordPress Volume에는 WordPress 파일, 업로드 파일, 테마, 플러그인 등이 저장될 수 있다.

---

# 24. Named Volume과 Bind Mount

## Named Volume

Docker가 이름으로 관리하는 Volume이다.

```yaml
volumes:
  mariadb_data:
  wordpress_data:
```

서비스에서:

```yaml
services:
  mariadb:
    volumes:
      - mariadb_data:/var/lib/mysql
```

처럼 사용한다.

## Bind Mount

Host의 실제 경로를 직접 Container에 연결한다.

```yaml
volumes:
  - /home/user/mysql:/var/lib/mysql
```

Inception에서는 WordPress 데이터와 MariaDB 데이터의 persistent storage에 Bind Mount를 직접 사용하는 것이 아니라 Docker Named Volume을 사용해야 한다.

또한 실제 데이터는 Host의 다음 경로 아래에 위치해야 한다.

```text
/home/<login>/data
```

---

# 25. Docker Network란?

각 Container는 서로 다른 Network Namespace를 가지기 때문에 서로 통신하려면 Docker Network를 통해 연결해야 한다.

예:

```text
              Docker Network
        ┌────────┼─────────┐
        │        │         │
        ▼        ▼         ▼
      nginx   wordpress   mariadb
```

각 Container는 내부 IP를 가질 수 있다.

```text
nginx       172.18.0.2
wordpress   172.18.0.3
mariadb     172.18.0.4
```

하지만 IP 주소는 Container 재생성 시 달라질 수 있기 때문에 보통 Service Name을 사용한다.

예:

```text
WordPress → mariadb:3306
NGINX → wordpress:<php-fpm-port>
```

Docker 내부 DNS가 Service Name을 해당 Container IP로 해석해준다.

---

# 26. Inception의 통신 흐름

전체 통신 구조는 다음과 같다.

```text
Internet
   │
   │ HTTPS :443
   ▼
NGINX Container
   │
   │ FastCGI
   ▼
WordPress + PHP-FPM Container
   │
   │ MariaDB Protocol / SQL
   ▼
MariaDB Container
```

통신 종류를 정리하면:

```text
Browser → NGINX
= HTTPS

NGINX → PHP-FPM
= FastCGI

WordPress → MariaDB
= Database Connection / SQL
```

---

# 27. HTTP와 HTTPS

HTTP는 웹 브라우저와 웹 서버가 통신하기 위한 프로토콜이다.

예:

```http
GET /
```

서버는 다음처럼 응답할 수 있다.

```http
HTTP/1.1 200 OK
```

일반 HTTP는 기본적으로 암호화되지 않는다.

HTTPS는:

```text
HTTPS
= HTTP + TLS
```

라고 이해하면 된다.

TLS가 통신을 보호한다.

---

# 28. TLS의 역할

TLS의 핵심 역할은 다음과 같다.

## 28.1 암호화

통신 내용을 제3자가 읽기 어렵게 만든다.

## 28.2 무결성

전송 중 데이터가 변경되거나 변조되지 않았는지 확인한다.

## 28.3 서버 인증

브라우저가 접속한 서버가 해당 도메인의 서버가 맞는지 인증서를 통해 확인한다.

---

# 29. Port 443

대표적으로:

```text
HTTP  → Port 80
HTTPS → Port 443
```

을 사용한다.

Inception에서는 외부 사용자가 접근할 수 있는 유일한 진입점은 NGINX이며 Port 443만 사용해야 한다.

```text
Internet
   │
   │ 443
   ▼
NGINX
```

MariaDB Port 3306 등을 외부에 직접 공개하는 구조가 아니다.

---

# 30. `.env`란?

`.env`는 환경변수를 파일로 관리하는 방식이다.

예:

```env
DOMAIN_NAME=login.42.fr
MYSQL_DATABASE=wordpress
MYSQL_USER=wordpress_user
```

Docker Compose에서 다음처럼 사용할 수 있다.

```yaml
environment:
  MYSQL_DATABASE: ${MYSQL_DATABASE}
  MYSQL_USER: ${MYSQL_USER}
```

장점은 프로그램 코드와 설정을 분리할 수 있다는 것이다.

예:

```text
개발 환경
DB_HOST=localhost

Docker 환경
DB_HOST=mariadb
```

코드를 수정하지 않고 설정만 변경할 수 있다.

---

# 31. Environment Variable이란?

Environment Variable은 프로세스가 실행될 때 제공되는 외부 설정값이다.

Shell에서는:

```bash
echo $MYSQL_USER
```

C에서는:

```c
getenv("MYSQL_USER");
```

처럼 접근할 수 있다.

환경변수를 사용하면 설정값을 코드 내부에 직접 하드코딩하지 않아도 된다.

---

# 32. Secret이란?

비밀번호와 같은 민감한 정보를 일반 설정과 분리해 관리하기 위한 방식이다.

개념적으로:

```text
일반 설정
→ .env

민감정보
→ secrets
```

예:

```text
DOMAIN_NAME
MYSQL_DATABASE
MYSQL_USER
→ .env

DB_PASSWORD
DB_ROOT_PASSWORD
→ secret
```

예시 디렉터리:

```text
secrets/
├─ credentials.txt
├─ db_password.txt
└─ db_root_password.txt
```

비밀번호, 자격증명, API Key 등을 Git 저장소에 공개하면 안 된다.

---

# 33. Inception 디렉터리 구조

과제에서 요구되는 전체 구조를 단순화하면 다음과 같다.

```text
inception/
│
├─ Makefile
│
├─ README.md
├─ USER_DOC.md
├─ DEV_DOC.md
│
├─ secrets/
│  ├─ credentials.txt
│  ├─ db_password.txt
│  └─ db_root_password.txt
│
└─ srcs/
   │
   ├─ docker-compose.yml
   ├─ .env
   │
   └─ requirements/
      │
      ├─ nginx/
      │  ├─ Dockerfile
      │  ├─ .dockerignore
      │  ├─ conf/
      │  └─ tools/
      │
      ├─ wordpress/
      │  ├─ Dockerfile
      │  ├─ conf/
      │  └─ tools/
      │
      ├─ mariadb/
      │  ├─ Dockerfile
      │  ├─ conf/
      │  └─ tools/
      │
      └─ bonus/
```

---

# 34. Makefile 역할

Makefile은 프로젝트 전체 실행을 간단하게 만든다.

예를 들어:

```bash
make
```

명령 하나로 내부적으로:

```bash
docker compose up --build
```

같은 명령을 실행하게 구성할 수 있다.

과제에서 Makefile은 Repository Root에 있어야 한다.

---

# 35. docker-compose.yml 역할

`docker-compose.yml`은 프로젝트 전체 인프라 구성 파일이다.

여기에서 다음 내용을 정의한다.

- Service
- Build 경로
- Port
- Environment Variable
- Secret
- Volume
- Network
- Restart Policy
- Service 간 관계

개념적으로:

```yaml
services:
  nginx:
    ...

  wordpress:
    ...

  mariadb:
    ...

networks:
  inception:

volumes:
  wordpress_data:
  mariadb_data:
```

와 같은 구조를 가진다.

---

# 36. 각 서비스 디렉터리 역할

## NGINX

```text
requirements/nginx/
├─ Dockerfile
├─ conf/
└─ tools/
```

주요 내용:

- NGINX 설치
- TLS 설정
- Port 443 설정
- PHP-FPM 연결
- NGINX configuration

## WordPress

```text
requirements/wordpress/
├─ Dockerfile
├─ conf/
└─ tools/
```

주요 내용:

- PHP 설치
- PHP-FPM 설치 및 설정
- WordPress 설치
- WordPress 초기 설정

## MariaDB

```text
requirements/mariadb/
├─ Dockerfile
├─ conf/
└─ tools/
```

주요 내용:

- MariaDB 설치
- Database 초기화
- User 생성
- WordPress DB 생성
- Password/Secret 처리

---

# 37. 프로젝트 전체 실행 과정

전체 과정을 순서대로 보면 다음과 같다.

```text
Makefile
   ↓
docker-compose.yml
   ↓
각 Service Dockerfile Build
   ↓
Docker Images 생성
   ↓
Containers 생성 및 실행
   ↓
Docker Network 연결
   ↓
Volumes Mount
   ↓
Environment / Secrets 전달
   ↓
서비스 실행
```

실행 결과:

```text
                    Internet
                       │
                       │ HTTPS / TLS :443
                       ▼
                ┌─────────────┐
                │    NGINX    │
                │  Container  │
                └──────┬──────┘
                       │ FastCGI
                       ▼
                ┌─────────────┐
                │ WordPress   │
                │ + PHP-FPM   │
                │  Container  │
                └──────┬──────┘
                       │ SQL
                       ▼
                ┌─────────────┐
                │   MariaDB   │
                │  Container  │
                └─────────────┘

                  Docker Network

               │                 │
               ▼                 ▼
       WordPress Volume      DB Volume
```

---

# 38. Inception에서 반드시 이해해야 할 핵심 관계

## Docker 관련

```text
Dockerfile
    ↓
Image
    ↓
Container
```

## 실행 관리

```text
docker-compose.yml
    ↓
여러 Container를 하나의 Stack으로 관리
```

## 데이터

```text
Container
    ↓
Volume
    ↓
데이터 영속성
```

## 네트워크

```text
Container
    ↓
Docker Network
    ↓
다른 Container와 통신
```

## 보안 설정

```text
.env
→ 일반 설정

Secrets
→ 민감정보
```

---

# 39. Inception 과제의 핵심 학습 포인트

이 프로젝트를 마치면 다음 질문에 답할 수 있어야 한다.

- Docker란 무엇인가?
- Container란 무엇인가?
- Dockerfile이란 무엇인가?
- Image와 Container의 차이는 무엇인가?
- Docker와 VM의 차이는 무엇인가?
- 왜 Container는 Host Kernel을 공유하는가?
- Linux Namespace란 무엇인가?
- cgroup은 어떤 역할을 하는가?
- Docker Volume은 왜 필요한가?
- Named Volume과 Bind Mount의 차이는 무엇인가?
- Docker Network는 왜 필요한가?
- Container가 Service Name으로 통신할 수 있는 이유는 무엇인가?
- NGINX는 어떤 역할을 하는가?
- WordPress는 어떤 역할을 하는가?
- PHP-FPM은 왜 필요한가?
- MariaDB는 왜 필요한가?
- HTTP와 HTTPS의 차이는 무엇인가?
- TLS는 무엇을 보호하는가?
- 왜 Port 443을 사용하는가?
- `.env`는 왜 사용하는가?
- Secret은 왜 필요한가?
- PID 1이 왜 중요한가?
- 왜 NGINX, WordPress, MariaDB를 각각 다른 Container로 분리하는가?

---

# 40. 최종 핵심 요약

Inception은 다음 구조를 직접 만들어보는 과제이다.

```text
Physical PC
   ↓
Virtual Machine
   ↓
Docker
   ↓
Docker Compose
   ↓
┌─────────────────────────────────┐
│                                 │
│ NGINX Container                 │
│        │                        │
│        ▼                        │
│ WordPress + PHP-FPM Container   │
│        │                        │
│        ▼                        │
│ MariaDB Container               │
│                                 │
└─────────────────────────────────┘
           │
           ├─ Docker Network
           ├─ Docker Volumes
           ├─ .env
           └─ Secrets
```

핵심 문장으로 정리하면:

> **Inception은 VM 내부에서 Docker를 이용해 NGINX, WordPress + PHP-FPM, MariaDB를 각각 독립적인 Container로 구축하고, Docker Network와 Volume을 이용해 하나의 웹 서비스 인프라로 연결하는 프로젝트이다.**

그리고 Docker Container는:

> **별도의 운영체제를 실행하는 VM이 아니라, Host Linux Kernel을 공유하면서 Namespace와 cgroup 등의 기능을 사용해 격리된 실행 환경을 제공하는 방식이다.**

