---
title: 'Dockerfile과 docker-compose, 한 번에 정리하기 (도커 시리즈 2편)'
description: 'Dockerfile의 각 문법이 무엇을 의미하는지, multi-stage build가 왜 필요한지, 그리고 docker-compose는 왜 따로 존재하는지 FE 관점에서 정리합니다.'
pubDate: 'Jun 21 2026'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

> 도커 시리즈 2편입니다. [1편](/blog/docker-1-why-and-what/)에서 도커가 무엇인지 큰 그림을 잡았다면, 이번엔 실제로 쓰는 두 파일 **Dockerfile**과 **docker-compose**를 정리합니다.

## 전체 관계 한 장 그림

먼저 둘의 관계부터 머릿속에 그려두면 이해가 쉽습니다.

```
Dockerfile  ──build──▶  이미지  ──run──▶  컨테이너 1개
   (레시피)            (붕어빵틀)         (붕어빵)

docker-compose.yml  ──▶  컨테이너 여러 개를 한 번에 + 연결 + 관리
   (여러 컨테이너 오케스트라 악보)
```

- **Dockerfile** = "컨테이너 **1개**를 어떻게 만드나"
- **docker-compose** = "컨테이너 **여러 개**를 어떻게 같이 돌리나"

층위가 다릅니다. 그럼 하나씩 봅시다.

## 1. Dockerfile — 이미지를 만드는 레시피

아래는 제가 실제로 작성했던 (Next.js 기준) Dockerfile입니다.

```dockerfile
FROM node:20.19 AS base

FROM base AS deps
WORKDIR /app
COPY ./package.json ./package.json
COPY ./yarn.lock ./yarn.lock
RUN yarn

FROM base AS builder
WORKDIR /app
COPY . .
COPY --from=deps /app/node_modules ./node_modules
RUN yarn build

FROM base AS runner
WORKDIR /app
COPY --from=builder /app /app
EXPOSE 3000
ENV PORT 3000
CMD ["yarn","start"]
```

### 각 문법이 의미하는 것

| 명령어 | 의미 |
|--------|------|
| `FROM node:20.19 AS base` | 베이스 이미지 지정. `AS base`로 이 단계에 **이름표**를 붙임 |
| `WORKDIR /app` | 컨테이너 안 작업 디렉토리를 `/app`으로 |
| `COPY 원본 대상` | 호스트 → 컨테이너로 파일 복사 (왼쪽=내 PC, 오른쪽=컨테이너) |
| `RUN yarn` | 이미지 **빌드 중** 명령 실행 (의존성 설치) |
| `COPY --from=deps ...` | **다른 스테이지**의 결과물을 가져옴 ← multi-stage의 핵심 |
| `EXPOSE 3000` | "이 컨테이너는 3000 포트를 쓴다"는 **표시**(실제 개방 아님) |
| `ENV PORT 3000` | 환경변수 설정 |
| `CMD ["yarn","start"]` | 컨테이너가 **실행될 때** 돌릴 기본 명령 |

### 꼭 기억할 것: RUN vs CMD

이거 면접 단골입니다.

- `RUN`: 이미지를 **만들 때** 실행 (설치, 빌드) — 1번
- `CMD`: 컨테이너를 **시작할 때** 실행 (서버 켜기) — 매 실행마다
- `EXPOSE`: 그냥 "이 포트 쓴다"는 메모. 실제 포트 연결은 `docker run -p`가 함

### multi-stage build — 왜 단계를 나누나

위 파일은 `deps` → `builder` → `runner` 세 단계로 나뉘어 있습니다. 이렇게 나누는 이유는 두 가지입니다.

1. **캐싱**: `deps`에서 `package.json`과 `yarn.lock`만 먼저 복사해 설치하면, 소스만 바뀌고 lock 파일이 그대로일 때 도커가 설치 단계를 **캐시에서 재사용**합니다. 빌드가 빨라져요.
2. **가벼운 최종 이미지**: 빌드할 때만 필요한 무거운 도구는 버리고, **실행에 필요한 것만** 최종(runner) 이미지에 담을 수 있습니다.

> ⚠️ 그런데 위 코드엔 큰 실수가 하나 숨어 있습니다. `runner`에서 `COPY --from=builder /app /app`로 **전부 다시 복사**해버려서, 애써 나눈 multi-stage의 "가벼운 이미지" 효과가 통째로 날아갔어요. 이 삽질 이야기는 [3편](/blog/docker-3-troubleshooting/)에서 자세히 다룹니다.

## 2. docker-compose — 여러 컨테이너를 한 번에

컨테이너 하나는 `docker run`으로 띄우면 됩니다. 그런데 실제 앱은 **FE + 백엔드 API + DB**처럼 컨테이너가 여러 개죠. 이걸 명령어로만 띄우면:

```bash
docker network create myapp
docker run -d --name db --network myapp -e MYSQL_ROOT_PASSWORD=1234 mysql
docker run -d --name api --network myapp -p 4000:4000 my-backend
docker run -d --name web --network myapp -p 3000:3000 my-frontend
```

외울 것도 많고, 순서·네트워크·환경변수를 전부 수동으로 챙겨야 합니다. 팀원에게 공유하기도 어렵죠. **docker-compose**는 이걸 파일 하나로 대체합니다.

```yaml
services:
  web:
    build: ./frontend          # frontend 폴더의 Dockerfile로 빌드
    ports:
      - "3000:3000"
    depends_on:
      - api                    # api가 먼저 뜨고 나서 web 시작
  api:
    image: node:20-alpine
    ports:
      - "4000:4000"
  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: 1234
```

그리고 끝입니다.

```bash
docker compose up      # 셋 다 한 번에 뜸
docker compose down    # 셋 다 한 번에 정리
```

### compose가 주는 가장 큰 마법: 서비스 이름 = 주소

| 직접 docker run | docker compose |
|----------------|----------------|
| 명령어 여러 줄, 매번 손으로 | 파일 1개 + `up` 한 줄 |
| 네트워크 수동 생성 | 자동으로 같은 네트워크에 묶임 |
| 컨테이너끼리 IP로 찾아야 | **서비스 이름으로 통신** (`http://api:4000`) |
| 팀 공유 어려움 | 파일을 git에 커밋 → 누구나 `up` 한 줄 |

FE 컨테이너에서 백엔드를 부를 때 IP를 몰라도 `http://api:4000`처럼 **컨테이너 이름으로** 접근됩니다. compose가 내부 네트워크를 자동으로 깔아주거든요.

> 사실 예전에 백엔드 팀이 "이거 띄우면 API랑 DB 다 떠요" 하고 줬던 그 개발 환경이, 십중팔구 이 `docker-compose.yml`이었습니다. 저는 그걸 **소비**만 했던 거고요.

### ⚠️ 함정: `deploy:`는 Swarm 전용

운영용 compose를 보면 이런 옵션이 있습니다.

```yaml
deploy:
  replicas: 2
  update_config:
    parallelism: 1
    delay: 5s
```

`replicas`, `update_config` 같은 옵션은 **그냥 `docker compose up`으로는 대부분 무시됩니다.** 이건 **Docker Swarm**(`docker stack deploy`) 같은 오케스트레이터에서만 진짜로 동작해요. 그래서 이런 옵션이 보이면 "아, 이 프로젝트는 Swarm으로 운영하는구나"라고 읽으면 됩니다.

## 둘의 관계 — 같이 쓰인다

compose는 Dockerfile을 **포함**하는 상위 개념입니다.

```yaml
services:
  web:
    build: ./frontend     # 여기서 frontend/Dockerfile을 빌드
  api:
    image: node:20        # 이미 있는 이미지 그대로
```

compose의 `build:`가 Dockerfile을 호출해 이미지를 만들고, 그걸 컨테이너로 띄웁니다. 즉 **Dockerfile(1개 만들기) → compose(여러 개 엮기)** 순으로 쌓이는 관계입니다.

## 면접용 한 줄 답변

- **Dockerfile이 뭐예요?** → 이미지를 만드는 레시피. 앱과 실행 환경을 묶어 어디서든 동일하게 실행되게 합니다.
- **docker-compose는요?** → 여러 컨테이너를 YAML 하나로 정의해 한 번에 띄우고 연결하는 도구. FE+API+DB 같은 멀티 컨테이너 환경에 씁니다.
- **multi-stage build 왜 써요?** → 빌드에만 필요한 무거운 도구를 최종 이미지에서 빼고, 실행에 필요한 것만 담아 이미지를 가볍게 만들려고요.

다음 편에서는 제가 직접 겪은 **"FE를 도커로 돌렸더니 너무 느렸던"** 경험과, 위에서 언급한 `COPY /app /app` 실수를 어떻게 발견하고 고쳤는지를 정리합니다.
