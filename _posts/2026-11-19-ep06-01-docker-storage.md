---
title: "EP.06.1 - Where Did My Data Go?"
date: 2026-11-25 00:00:00 +0900
categories: [Container]
tags: [docker, container, storage, writable-layer, bind-mount, selinux]
description: "Docker Storage: Writable Layer & Bind Mount"
---

## Prologue: 컨테이너의 데이터 위치

이번에는 Docker의 **저장 공간** 을 알아보려고 함.

컨테이너 안에서 파일을 만들고 컨테이너를 종료시키면? 파일은 어디로갈까요

그리고 컨테이너를 **삭제했다가 똑같은 이미지로 다시 만들면** 이전 파일들은 다시 보일까요

---

## 1. Docker Image와 Writable Layer

Docker 이미지는 여러 개의 읽기 전용(Read-only) 레이어로 구성된다.

이미지로 컨테이너를 생성하면 그 위에 **쓰기 가능한 레이어(Writable Layer)** 가 추가됨.

```text
+------------------------------------+
| Container: ep06-temp               |
+------------------------------------+
| Writable Layer                     |  ← 컨테이너에서 만든/수정한 파일
+------------------------------------+
| Read-only Image Layers             |  ← nginx:stable 이미지
+------------------------------------+
| Host Filesystem                    |
+------------------------------------+
```

여기서 중요한 점은 **Writable Layer의 수명이 해당 컨테이너에 묶여 있다**는 것.

| 동작 | Writable Layer의 파일 |
|---|---|
| `docker stop` | 유지 |
| `docker start` | 유지 |
| `docker restart` | 유지 |
| `docker rm` | 컨테이너와 함께 삭제 |


컨테이너가 정지했다고 데이터가 바로 사라지는 건 아니고, **컨테이너 자체를 삭제할 때** 데이터도 함께 삭제됨.

---

## 2. 실습 ① Writable Layer에 파일 생성

### STEP 1. 컨테이너 생성

```bash
docker run -d --name ep06-temp nginx:stable

docker ps --filter name=ep06-temp
```

이번에는 Nginx 웹페이지에 접속하는 게 목적이 아니라서 포트 매핑은 생략하고

### STEP 2. 컨테이너 내부에 파일 생성

```bash
docker exec ep06-temp sh -c 'echo "Hello Docker Volume" > /tmp/ep06.txt'

docker exec ep06-temp cat /tmp/ep06.txt
```

예상 출력:

```text
Hello Docker Volume
```

`/tmp/ep06.txt`는 별도 Mount를 연결하지 않았으므로 컨테이너 파일시스템의 Writable Layer에 저장된다.

### STEP 3. 변경 사항 확인: `docker diff`

```bash
docker diff ep06-temp
```

출력에는 Nginx가 만든 다른 변경 파일도 보일 수 있지만, 아래 항목을 찾아보면 됨.

```text
[root@hanju-playground samsung]# docker diff ep06-temp
C /etc
C /etc/nginx

... ... ...

A /run/nginx.pid
C /tmp
A /tmp/ep06.txt
```

- `A` : Added, 새로 추가된 파일/디렉터리
- `C` : Changed, 변경된 파일/디렉터리
- `D` : Deleted, 삭제된 파일/디렉터리

즉, `docker diff`로 이미지에 비해 컨테이너 파일시스템에 어떤 작업이 있는지 볼 수 있음.

---

## 3. 실습 ② Stop / Start를 하면?

```bash
docker stop ep06-temp
docker start ep06-temp

docker exec ep06-temp cat /tmp/ep06.txt
```

```text
Hello Docker Volume
```

파일이 그대로 남는다.

**Stop은 컨테이너를 중지할 뿐, 컨테이너 파일시스템을 삭제하는 게 아니기 때문.**

컨테이너 삭제 ㄱㄱ

---

## 4. 실습 ③ Remove / Run을 하면?

```bash
docker rm -f ep06-temp

docker run -d --name ep06-temp nginx:stable

docker exec ep06-temp ls -l /tmp/ep06.txt
```

출력:

```text
ls: cannot access '/tmp/ep06.txt': No such file or directory
```

이름도 `ep06-temp`로 똑같고 이미지도 `nginx:stable`로 똑같은데 파일이 없는건

지금 생성한 컨테이너는 아예 새로운 컨테이너임(아까랑 다른 객체(?))

기존 Writable Layer는 이전 컨테이너를 삭제할 때 함께 삭제됨.

그럼 컨테이너를 삭제해도 데이터는 유지하고 싶다면?

---

## 5. Bind Mount: 호스트 디렉터리를 연결하기

**Bind Mount**는 호스트의 파일이나 디렉터리를 컨테이너 내부에 연결(마운트)시키는 작업

ex:

```text
Rocky Linux Host                    Nginx Container
/opt/ep06/html                      /usr/share/nginx/html
└── index.html        <-------->    └── index.html
       같은 호스트 파일을 참조하는 경로
```

두 곳에 파일이 복사되는 것이 아니라, 컨테이너가 지정한 경로에서 **호스트 파일을 직접 사용**하게 된다.

그래서 컨테이너를 삭제해도 호스트에 있는 파일은 남는다.

### STEP 1. 호스트 디렉터리와 HTML 파일 생성

```bash
mkdir -p /opt/ep06/html

echo '<h1>EP.06 Bind Mount Test</h1>' > /opt/ep06/html/index.html

cat /opt/ep06/html/index.html
```

### STEP 2. Bind Mount로 Nginx 실행

```bash
docker run -d \
  --name ep06-bind \
  -p 127.0.0.1:8086:80 \
  -v /opt/ep06/html:/usr/share/nginx/html:Z \
  nginx:stable
```

`-v` 옵션은 아래처러 해석하면됨.

```text
-v /opt/ep06/html:/usr/share/nginx/html:Z
   └─ Host 경로 ─┘ └ Container 경로 ┘ └ SELinux 옵션
```

- `/opt/ep06/html` : 호스트의 원본 디렉터리
- `/usr/share/nginx/html` : 컨테이너에서 보이는 디렉터리
- `:Z` : SELinux 환경에서 해당 경로를 **이 컨테이너용으로 재레이블링**하는 옵션

`:Z`옵션은 호스트 디렉토리의 SELINUX 레이블을 변경하기 때문에, 호스트의 시스템 or 다른 서비스가 사용하고 있는 디렉토리에는 사용하면 안됨.

컨테이너별로 따로 구성해서 사용해야함.

### STEP 3. Nginx 접속 확인

```bash
[root@hanju-playground samsung]# curl -i http://127.0.0.1:8086
HTTP/1.1 200 OK
Server: nginx/1.30.6
Date: Thu, 26 Nov 2026 07:22:25 GMT
Content-Type: text/html
Content-Length: 31
Last-Modified: Thu, 26 Nov 2026 07:03:18 GMT
Connection: keep-alive
ETag: "6b07d9b6-1f"
Accept-Ranges: bytes

<h1>EP.06 Bind Mount Test</h1>

```

### STEP 4. 호스트 파일을 수정하면?

컨테이너를 재시작하지 않고 호스트의 파일을 수정해 보자.

```bash
echo '<h1>Updated from Host!</h1>' > /opt/ep06/html/index.html

curl http://127.0.0.1:8086
```

출력:

```html
<h1>Updated from Host!</h1>
```

컨테이너 안으로 파일을 다시 복사하지도 않았고 Nginx를 재시작하지도 않았는데 변경 내용이 반영됨.

-> 컨테이너가 호스트 파일을 그대로 참조하기 때문임.

### STEP 5. 컨테이너 삭제 후 재생성

이번에도 컨테이너를 삭제하고 다시 실행해 본다.

```bash

[root@hanju-playground samsung]# docker rm -f ep06-bind
ep06-bind

[root@hanju-playground samsung]# docker run -d \
  --name ep06-bind \
  -p 127.0.0.1:8086:80 \
  -v /opt/ep06/html:/usr/share/nginx/html:Z \
  nginx:stable
181dceb2e0e0275d5036ad499aa3e11633866332315a2d94d61b6fa9c158da9c

[root@hanju-playground samsung]# curl http://127.0.0.1:8086
<h1>Updated from Host!</h1>
```

**컨테이너는 새로 만들어졌지만 `/opt/ep06/html/index.html`은 호스트에 그대로 남아 있기 때문에** 데이터가 유지된다.

### STEP 6. Mount 정보 확인: `docker inspect`

```bash
docker inspect ep06-bind --format '{{json .Mounts}}'
```
```bash
[{"Type":"bind","Source":"/opt/ep06/html","Destination":"/usr/share/nginx/html","Mode":"Z","RW":true,"Propagation":"rprivate"}]
```

출력에서 주목할 항목:

| 항목 | 의미 | 이번 실습의 값 |
|---|---|---|
| `Type` | Mount 종류 | `bind` |
| `Source` | 호스트의 원본 경로 | `/opt/ep06/html` |
| `Destination` | 컨테이너 내부 경로 | `/usr/share/nginx/html` |
| `RW` | 읽기/쓰기 여부 | `true` |

> 참고: Nginx의 `/usr/share/nginx/html`에 원래 있던 이미지 파일들은 Bind Mount가 연결된 동안 해당 경로에서 가려진다. 이미지를 영구 수정한 것은 아니다.

---

## 6. Writable Layer vs Bind Mount

| 구분 | Writable Layer | Bind Mount |
|---|---|---|
| 데이터 저장 위치 | 컨테이너 파일시스템의 쓰기 레이어 | 지정한 호스트 경로 |
| 컨테이너 재시작 | 유지 | 유지 |
| 컨테이너 삭제 | 함께 삭제 | 호스트 파일 유지 |
| 호스트에서 파일 직접 수정 | 일반적인 사용 방식 아님 | 가능 |
| 대표 용도 | 임시 변경 사항 | 개발 파일 연동, 설정·데이터 연결 |

물론 Bind Mount만이 정답은 아니다. 호스트 경로에 직접 의존하고, 파일 접근 권한이나 SELinux도 고려해야 한다.

그래서 Docker에서는 저장공간을 관리하는 **Named Volume**이라는 방식이 있음

그건 다음 챕터에 ㄱㄱ

---

## Review

1. 컨테이너의 **Writable Layer**는 컨테이너를 삭제하면 함께 사라진다.
2. `docker stop`과 `docker rm`은 전혀 다른 동작이다.
3. **Bind Mount**를 쓰면 컨테이너가 호스트 디렉터리의 파일을 직접 사용할 수 있다.
4. 따라서 컨테이너를 새로 생성해도 호스트 파일은 유지된다.



---

## 참고 문서

- [Docker Docs — Storage overview](https://docs.docker.com/engine/storage/)
- [Docker Docs — Bind mounts](https://docs.docker.com/engine/storage/bind-mounts/)
- [Docker Docs — docker container diff](https://docs.docker.com/reference/cli/docker/container/diff/)
