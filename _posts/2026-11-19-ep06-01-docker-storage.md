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

이미지로 컨테이너를 만들면 그 위에 **쓰기 가능한 레이어(Writable Layer)** 가 추가됨.

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

> 이 표는 **Bind Mount나 Volume에 저장하지 않은 파일**을 기준으로 한다.

컨테이너가 정지했다고 데이터가 바로 사라지는 건 아니고, **컨테이너 자체를 삭제할 때** 문제가 생기는 거였음.

---

## 2. 실습 ① Writable Layer에 파일 생성

기존에 사용하던 `nginx-test`, `nginx-test2`는 건드리지 않고 별도 컨테이너로 테스트한다.

### STEP 1. 컨테이너 생성

```bash
docker run -d --name ep06-temp nginx:stable

docker ps --filter name=ep06-temp
```

이번에는 Nginx 웹페이지에 접속하는 게 목적이 아니라서 포트 매핑은 생략했다.

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
A /tmp/ep06.txt
```

- `A` : Added, 새로 추가된 파일/디렉터리
- `C` : Changed, 변경된 파일/디렉터리
- `D` : Deleted, 삭제된 파일/디렉터리

즉, `docker diff`로 이미지에 비해 컨테이너 파일시스템에 어떤 변화가 생겼는지 볼 수 있다.

---

## 3. 실습 ② Stop / Start를 하면?

```bash
docker stop ep06-temp
docker start ep06-temp

docker exec ep06-temp cat /tmp/ep06.txt
```

예상 출력:

```text
Hello Docker Volume
```

파일이 그대로 남는다.

**Stop은 컨테이너를 중지할 뿐, 컨테이너 파일시스템을 삭제하는 게 아니기 때문.**

그럼 컨테이너 자체를 지워보자.

---

## 4. 실습 ③ Remove / Run을 하면?

> 아래 명령어는 이번 실습용 `ep06-temp` 컨테이너를 삭제한다. 앞에서 만든 `/tmp/ep06.txt`는 복구되지 않는다.

```bash
docker rm -f ep06-temp

docker run -d --name ep06-temp nginx:stable

docker exec ep06-temp ls -l /tmp/ep06.txt
```

예상 출력:

```text
ls: cannot access '/tmp/ep06.txt': No such file or directory
```

이름도 `ep06-temp`로 똑같고 이미지도 `nginx:stable`로 똑같은데 파일이 없다.

**같은 컨테이너를 다시 시작한 게 아니라, 새로운 컨테이너를 생성했기 때문**이다.

기존 Writable Layer는 이전 컨테이너를 삭제할 때 함께 사라졌다.

실습용 컨테이너는 정리해 둔다.

```bash
docker rm -f ep06-temp
```

그럼 컨테이너를 삭제해도 데이터를 유지하려면 어떻게 해야 할까?

---

## 5. Bind Mount: 호스트 디렉터리를 연결하기

**Bind Mount**는 호스트의 파일이나 디렉터리를 컨테이너 내부의 특정 경로에 연결하는 방식이다.

예를 들어:

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

`-v` 옵션은 아래처럼 해석하면 된다.

```text
-v /opt/ep06/html:/usr/share/nginx/html:Z
   └─ Host 경로 ─┘ └ Container 경로 ┘ └ SELinux 옵션
```

- `/opt/ep06/html` : 호스트의 원본 디렉터리
- `/usr/share/nginx/html` : 컨테이너에서 보이는 디렉터리
- `:Z` : SELinux 환경에서 해당 경로를 **이 컨테이너용으로 재레이블링**하는 옵션

Rocky Linux 9에서는 SELinux가 활성화되어 있을 수 있어서 `:Z` 옵션도 알아두면 좋다. 다만 이 옵션은 **호스트 디렉터리의 SELinux 레이블을 변경**하므로, 시스템 디렉터리나 다른 서비스가 사용하는 공유 디렉터리에 함부로 사용하면 안 된다. 여러 컨테이너가 동일 경로를 공유해야 한다면 `:z` 등 별도 구성을 검토해야 한다.

또한 `-p 127.0.0.1:8086:80`으로 설정했기 때문에 **호스트의 루프백 주소에서만** 접속하도록 제한했다.

### STEP 3. Nginx 접속 확인

```bash
curl http://127.0.0.1:8086
```

예상 출력:

```html
<h1>EP.06 Bind Mount Test</h1>
```

### STEP 4. 호스트 파일을 수정하면?

컨테이너를 재시작하지 않고 호스트의 파일을 수정해 보자.

```bash
echo '<h1>Updated from Host!</h1>' > /opt/ep06/html/index.html

curl http://127.0.0.1:8086
```

예상 출력:

```html
<h1>Updated from Host!</h1>
```

컨테이너 안으로 파일을 다시 복사하지도 않았고 Nginx를 재시작하지도 않았는데 변경 내용이 반영된다.

컨테이너가 호스트 파일을 그대로 참조하기 때문임.

### STEP 5. 컨테이너 삭제 후 재생성

이번에도 컨테이너를 삭제하고 다시 실행해 본다.

```bash
docker rm -f ep06-bind

docker run -d \
  --name ep06-bind \
  -p 127.0.0.1:8086:80 \
  -v /opt/ep06/html:/usr/share/nginx/html:Z \
  nginx:stable

curl http://127.0.0.1:8086
```

예상 출력:

```html
<h1>Updated from Host!</h1>
```

**컨테이너는 새로 만들어졌지만 `/opt/ep06/html/index.html`은 호스트에 그대로 남아 있기 때문에** 데이터가 유지된다.

### STEP 6. Mount 정보 확인: `docker inspect`

```bash
docker inspect ep06-bind --format '{{json .Mounts}}'
```

출력에서 주목할 항목:

| 항목 | 의미 | 이번 실습의 값 |
|---|---|---|
| `Type` | Mount 종류 | `bind` |
| `Source` | 호스트의 원본 경로 | `/opt/ep06/html` |
| `Destination` | 컨테이너 내부 경로 | `/usr/share/nginx/html` |
| `RW` | 읽기/쓰기 여부 | `true` |

여기까지 확인했다면 Bind Mount의 개념은 어느 정도 감이 온다.

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

Docker가 저장 공간을 관리하는 **Named Volume**이라는 방식도 있다고 한다.

그건 다음에 직접 확인해 보자.

---

## Review

오늘의 결론은 간단하다.

1. 컨테이너의 **Writable Layer**는 컨테이너를 삭제하면 함께 사라진다.
2. `docker stop`과 `docker rm`은 전혀 다른 동작이다.
3. **Bind Mount**를 쓰면 컨테이너가 호스트 디렉터리의 파일을 직접 사용할 수 있다.
4. 따라서 컨테이너를 새로 생성해도 호스트 파일은 유지된다.

VM을 다룰 때는 OS 디스크와 데이터 디스크를 분리해서 생각했는데, 컨테이너에서는 **컨테이너의 수명과 데이터의 수명을 따로 생각해야 한다**는 점이 특히 중요해 보인다.

일단 직접 해보니 이해가 됨. 다음은 Docker가 알아서 관리한다는 Volume을 공부해 보겠음.

**Next Quest: EP.06.2 - The Data Must Survive (Docker Named Volume)**

---

## 참고 문서

- [Docker Docs — Storage overview](https://docs.docker.com/engine/storage/)
- [Docker Docs — Bind mounts](https://docs.docker.com/engine/storage/bind-mounts/)
- [Docker Docs — docker container diff](https://docs.docker.com/reference/cli/docker/container/diff/)
