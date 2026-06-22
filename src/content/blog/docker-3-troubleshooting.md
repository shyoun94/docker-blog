---
title: 'FE를 도커로 돌렸더니 너무 느렸던 이유, 그리고 이미지 다이어트 (도커 시리즈 3편)'
description: '프론트엔드를 도커로 돌렸을 때 느렸던 진짜 원인과, multi-stage build를 망쳐버린 COPY 한 줄, 그리고 이미지 용량을 줄인 과정을 정리합니다.'
pubDate: 'Jun 22 2026'
heroImage: '../../assets/blog-placeholder-4.jpg'
---

> 도커 시리즈 마지막 3편입니다. [1편](/blog/docker-1-why-and-what/)·[2편](/blog/docker-2-dockerfile-and-compose/)이 개념이었다면, 이번엔 제가 직접 겪은 삽질과 그 해결 기록입니다.

## "FE를 도커로 돌리니까 왜 이렇게 느리지?"

처음 프론트엔드를 도커로 돌려봤을 때 가장 당황스러웠던 건 **속도**였습니다. 백엔드는 도커로 잘만 개발했는데, FE를 도커에 올리니 뜨는 것부터 답답할 만큼 느렸어요.

한참 헤매고 나서야 깨달은 건, **FE를 도커로 돌리는 방식이 두 가지이고 둘은 완전히 다르다**는 점이었습니다.

### ① 개발 서버를 도커 안에서 (`npm run dev`) — 느리고, 보통 안 함 ❌

컨테이너 안에서 Vite/webpack dev 서버를 HMR(핫리로드)과 함께 돌리는 방식입니다. 이게 느린 이유는:

- 컨테이너를 띄울 때마다 `node_modules`를 다시 설치하거나, 볼륨 마운트를 거치며 **파일시스템이 느려짐** (특히 macOS + 도커의 파일 동기화는 악명 높게 느립니다)
- HMR이 컨테이너 ↔ 호스트 간 파일 변경 감지를 거쳐야 해서 **반응이 굼뜸**

제가 느꼈던 답답함이 바로 이거였습니다. 그리고 결론은: **대부분의 FE 개발자는 개발 중엔 도커를 안 씁니다.** `npm run dev`는 그냥 호스트에서 네이티브로 돌리는 게 제일 빠르고 쾌적해요.

### ② 빌드 결과물을 서빙 (배포용 이미지) — 이게 진짜 ✅

`npm run build`로 만든 정적 파일이나 빌드 산출물을 nginx 또는 Node 서버로 서빙하는 방식입니다. 빌드는 이미지를 만들 때 **딱 한 번**만 일어나고, 실행 시점엔 빌드가 없어서 **런타임이 매우 빠릅니다.**

### 백엔드와 헷갈렸던 이유

| | 백엔드 | 프론트엔드 |
|---|--------|-----------|
| 개발 중 | 도커로 띄워 사용 (long-running 프로세스) | 보통 호스트에서 `npm run dev` |
| 배포 | 도커 이미지로 배포 | 도커 이미지 또는 정적 호스팅 |
| 컨테이너가 하는 일 | 서버 프로세스가 계속 살아있음 | 빌드된 파일을 **서빙**(전달)만 |

가장 중요한 차이는 이겁니다. 백엔드는 "서버"라서 도커 안에서 계속 실행돼야 합니다. 그런데 프론트는 빌드되고 나면 그냥 **html/js/css 덩어리**예요. 실제 실행은 **사용자의 브라우저**에서 일어나죠. 그래서 FE 컨테이너의 정체는 사실 **"정적 파일 배달부"**에 가깝습니다.

> 백엔드처럼 "FE 컨테이너 안에서 앱이 살아 돌아가야 한다"고 생각하면 계속 어색합니다. 이 감각을 바꾸고 나서야 도커가 편해졌어요.

## multi-stage를 망쳐버린 COPY 한 줄

[2편](/blog/docker-2-dockerfile-and-compose/)에서 잠깐 언급했던 제 Dockerfile의 실수입니다. runner 단계가 이랬어요.

```dockerfile
FROM base AS runner
WORKDIR /app
COPY --from=builder /app /app    # ← 문제의 한 줄
EXPOSE 3000
ENV PORT 3000
CMD ["yarn","start"]
```

`deps` → `builder` → `runner`로 단계를 애써 나눠놓고, 마지막에 builder의 `/app`을 **통째로** 복사했습니다. multi-stage를 쓰는 가장 큰 이유가 "최종 이미지를 작게"인데, 전부 다시 합쳐버려서 **그 효과가 0이 된** 거죠.

### "근데 /app만 복사하면 node_modules는 안 따라오는 거 아닌가?"

제가 실제로 했던 착각입니다. `node_modules`와 `.next`는 `/app`의 **상위**가 아니라 **하위**에 있습니다.

```
/app/                  ← 복사 대상
├── package.json
├── node_modules/   ← /app/node_modules (하위!)
└── .next/          ← /app/.next (하위!)
```

폴더를 복사하면 그 안의 내용물이 다 따라옵니다. `/app`이라는 **상자를 복사**하면 안에 든 무거운 짐(`node_modules`)도 같이 옮겨지는 거예요. 그래서 devDependencies까지 포함된 무거운 `node_modules`가 최종 이미지에 통째로 실렸습니다.

> 참고로 Next.js의 `src/app`(App Router 폴더)과 Dockerfile의 `WORKDIR /app`은 **이름만 같고 전혀 다른 것**입니다. 앞은 내가 지은 컨테이너 작업 폴더 이름이고, 뒤는 Next가 강제하는 라우팅 폴더예요. 전체 경로로는 `/app/src/app/`이 됩니다.

## 이미지 다이어트 — 고친 버전

문제를 정리하면 이렇습니다.

- 🔴 `COPY /app /app`로 전부 복사 → 멀티스테이지 효과 0
- 🔴 베이스가 `node:20.19`(데비안, ~1GB대) → 무거움
- 🔴 `.dockerignore`가 없어 `COPY . .`이 호스트 `node_modules`/`.git`까지 복사
- 🟡 devDependencies가 최종 이미지에 포함
- 🟡 `ENV PORT 3000` (등호 없는 옛 문법), root 실행

Next.js라면 `output: 'standalone'`을 쓰는 게 정석입니다.

```js
// next.config.js
module.exports = { output: 'standalone' }
```

```dockerfile
FROM node:20.19-alpine AS base

FROM base AS deps
WORKDIR /app
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN yarn build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV PORT=3000

RUN addgroup -g 1001 -S nodejs && adduser -S nextjs -u 1001

# standalone 결과물만 복사 (node_modules 통째로 X)
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

그리고 `.dockerignore`는 필수입니다.

```
node_modules
.next
.git
Dockerfile
.dockerignore
.env*.local
```

### 핵심 원칙

바꾼 것들을 한 줄로 줄이면 이렇습니다.

> **"runner에는 실행에 꼭 필요한 것만 골라 담는다."**

- 베이스를 `alpine`으로 → 이미지 용량 대폭 감소
- standalone으로 실행에 필요한 최소 파일만 복사 → devDependencies 제거
- `.dockerignore`로 빌드 컨텍스트 정리 → 빌드 속도·안정성 ↑
- non-root 유저로 실행 → 보안

## 배운 것 정리

FE 개발자로서 도커를 공부하며 얻은 결론입니다.

1. **개발 중엔 도커를 굳이 쓰지 않는다.** `npm run dev`가 빠르다.
2. **FE에서 도커는 "배포되는 환경"을 만들고 재현하는 도구**다. 빠른 코딩 도구가 아니다.
3. **multi-stage의 핵심은 "최종 이미지에 필요한 것만"** 이다. 다 복사하면 의미가 없다.
4. 깊게 파기보단 **"내 FE 앱을 이미지로 만들고 띄울 줄 안다"** 수준이면 이직용으로 충분하다.

도커가 막연히 "백엔드 것"처럼 느껴졌는데, 직접 Dockerfile을 고쳐보고 이미지가 가벼워지는 걸 눈으로 확인하고 나니 훨씬 친숙해졌습니다. 다음 목표는 이 블로그 자체를 도커로 컨테이너화해보는 것, 그리고 쿠버네티스 개념을 가볍게 훑어보는 것입니다.

읽어주셔서 감사합니다. 🐳
