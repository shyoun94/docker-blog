---
title: "FE 개발자가 도커를 공부하는 이유, 그리고 도커란? (도커 시리즈 1편)"
description: "백엔드만 도커를 쓰는 줄 알았던 프론트엔드 개발자가 도커를 공부하기로 한 이유와, 이미지·컨테이너 같은 기본 개념을 정리합니다."
pubDate: "Jun 20 2026"
heroImage: "../../assets/docker.jpg"
---

> 이 글은 FE 개발자인 제가 도커를 공부하면서 정리한 3부작 시리즈의 1편입니다.
> 1편(개념·동기) → 2편(Dockerfile & docker-compose) → 3편(트러블슈팅) 순으로 이어집니다.

## 왜 갑자기 도커인가

안녕하세요. 프론트엔드 2년차 개발자입니다.

도커는 실무를 하면서 처음 접했습니다. DevOps 담당자나 백엔드 쪽에서 만들어둔 도커 환경을 **가져다 쓰기만** 했을 뿐, 제가 직접 환경을 세팅해 본 적은 없었습니다. 솔직히 말하면 그 환경이 **무엇을 위해 존재하는지조차 제대로 알지 못했어요.**

처음엔 아무 생각 없이, 세팅이 다 된 상태로 그냥 쓰기만 했습니다. 그런데 쓰다 보니 점점 이런 생각이 들더군요. _"이게 정말 최적의 환경일까?", "직접 세팅하면 조금 더 나한테 맞는 상태로 쓸 수 있지 않을까?"_ 그렇게 한 번 제대로 공부해보기로 했습니다.

## 도커가 도대체 뭐길래

한 문장으로 정리하면 이렇습니다.

> **앱을 "실행에 필요한 모든 것"과 함께 통째로 포장해서, 어디서든 똑같이 실행되게 해주는 도구.**

### 도커가 풀어주는 진짜 문제

개발하다 보면 이런 경험 한 번쯤 있으시죠?

- 😫 "**제 컴퓨터에선 되는데요?**" — 동료는 Node 18, 나는 Node 22라서 빌드가 깨짐
- 😫 새 팀원이 들어오면 환경 세팅하는 데 반나절
- 😫 로컬에선 잘 되던 게 배포 서버에선 안 됨

전부 **환경이 제각각**이라서 생기는 문제입니다. 도커는 앱과 그 실행 환경(Node 버전, 라이브러리, 설정)을 **하나의 박스(이미지)에 봉인**해서, 내 맥에서도 동료 윈도우에서도 배포 서버에서도 똑같이 실행되게 만듭니다.

### 비유: 컨테이너 = 화물 컨테이너 🚢

도커(Docker)라는 이름은 부두(dock)에서 왔습니다. 옛날엔 배에 짐을 실을 때 상자 모양이 제각각이라 무척 힘들었는데, **규격이 통일된 철제 컨테이너**가 등장하면서 안에 뭐가 들었든 어떤 배·트럭·항구에서도 똑같이 다룰 수 있게 됐죠.

도커도 같습니다. 컨테이너 **안**에 뭐가 들었든(React 앱이든 Python 서버든), **밖**에서 다루는 방식은 늘 동일합니다(`docker run`, `docker stop`...).

여기까지 읽다보면 이런 의문이 생깁니다. **"앱을 통째로 격리해서 어디서든 똑같이 실행한다"**는 설명, 가상머신(VM)에서 들어본 얘기 같거든요. 저도 도커를 처음 접했을 때 "결국 VM이랑 같은 거 아냐?" 싶었습니다.

실제로 **"환경을 격리해서 똑같이 재현한다"는 목표는 둘이 닮았습니다.** 하지만 그걸 **어떤 방식으로**가 완전히 달라요. 핵심 차이만 그림으로 보면:

<div style="display:flex; flex-wrap:wrap; gap:1.25rem; margin:1.5rem 0;">
  <!-- 가상머신(VM) -->
  <div style="flex:1 1 260px; border:1px solid rgb(var(--gray-light)); border-radius:12px; padding:1rem;">
    <p style="margin:0 0 .15rem; font-weight:700; text-align:center;">🖥️ 가상머신 (VM)</p>
    <p style="margin:0 0 .75rem; text-align:center; font-size:.8rem; color:#c0392b;">수 GB · 부팅 수십 초~분</p>
    <div style="display:flex; gap:.5rem;">
      <div style="flex:1;">
        <div style="padding:.45rem; border-radius:8px; text-align:center; font-size:.85rem; font-weight:600; background:rgb(var(--gray-light));">앱 A</div>
        <div style="margin-top:.35rem; padding:.45rem; border-radius:8px; text-align:center; font-size:.8rem; font-weight:600; background:#fdecec; color:#c0392b; border:1px dashed #e5484d;">게스트 OS</div>
      </div>
      <div style="flex:1;">
        <div style="padding:.45rem; border-radius:8px; text-align:center; font-size:.85rem; font-weight:600; background:rgb(var(--gray-light));">앱 B</div>
        <div style="margin-top:.35rem; padding:.45rem; border-radius:8px; text-align:center; font-size:.8rem; font-weight:600; background:#fdecec; color:#c0392b; border:1px dashed #e5484d;">게스트 OS</div>
      </div>
    </div>
    <div style="margin-top:.35rem; padding:.45rem; border-radius:8px; text-align:center; font-size:.8rem; background:rgba(var(--gray),.18);">하이퍼바이저</div>
    <div style="margin-top:.35rem; padding:.45rem; border-radius:8px; text-align:center; font-size:.8rem; color:rgb(var(--gray-dark)); background:rgba(var(--gray),.28);">내 컴퓨터 OS (호스트)</div>
    <p style="margin:.6rem 0 0; text-align:center; font-size:.78rem; color:rgb(var(--gray-dark));">앱마다 <strong style="color:#c0392b;">게스트 OS</strong>를 한 겹 더 ↑</p>
  </div>

  <!-- 도커 컨테이너 -->
  <div style="flex:1 1 260px; border:1px solid rgb(var(--gray-light)); border-radius:12px; padding:1rem;">
    <p style="margin:0 0 .15rem; font-weight:700; text-align:center;">🐳 도커 컨테이너</p>
    <p style="margin:0 0 .75rem; text-align:center; font-size:.8rem; color:var(--accent);">수십 MB · 1초 이내</p>
    <div style="display:flex; gap:.5rem;">
      <div style="flex:1; padding:.45rem; border-radius:8px; text-align:center; font-size:.85rem; font-weight:600; background:rgb(var(--gray-light));">앱 A</div>
      <div style="flex:1; padding:.45rem; border-radius:8px; text-align:center; font-size:.85rem; font-weight:600; background:rgb(var(--gray-light));">앱 B</div>
    </div>
    <div style="margin-top:.35rem; padding:.45rem; border-radius:8px; text-align:center; font-size:.8rem; font-weight:700; color:var(--accent-dark); background:rgba(35,55,255,.12); border:1px solid rgba(35,55,255,.25);">도커 엔진</div>
    <div style="margin-top:.35rem; padding:.45rem; border-radius:8px; text-align:center; font-size:.8rem; color:rgb(var(--gray-dark)); background:rgba(var(--gray),.28);">내 컴퓨터 OS (호스트)</div>
    <p style="margin:.6rem 0 0; text-align:center; font-size:.78rem; color:rgb(var(--gray-dark));">호스트 OS를 <strong style="color:var(--accent);">공유</strong> → 게스트 OS 없음 ↓</p>
  </div>
</div>

- **VM**: 앱마다 운영체제를 통째로 하나씩 더 올림 → 무겁고(수 GB) 느림(켜는 데 수십 초~분)
- **도커**: 운영체제는 내 컴퓨터 것을 공유하고 앱과 필요한 것만 격리 → 가볍고(수십 MB) 빠름(1초 이내)

> 한 줄 요약: **VM은 컴퓨터를 통째로 흉내, 도커는 앱만 격리.**

## 핵심 용어 3개

사실 이 셋만 알면 시작은 충분합니다.

| 용어                    | 뜻                        | 비유                      |
| ----------------------- | ------------------------- | ------------------------- |
| **이미지(Image)**       | 실행 준비된 박스(템플릿)  | 붕어빵 **틀**             |
| **컨테이너(Container)** | 이미지를 실제로 실행한 것 | 구워진 **붕어빵**         |
| **Dockerfile**          | 이미지를 만드는 레시피    | 붕어빵 **반죽 만드는 법** |

관계는 이렇습니다: **`Dockerfile`(레시피) → 빌드 → `이미지`(틀) → 실행 → `컨테이너`(붕어빵)**

이미지 하나로 컨테이너를 여러 개 찍어낼 수 있다는 것, 컨테이너를 지워도 이미지(틀)는 남아있다는 것만 기억하면 됩니다.

## 그래서 FE 개발자에게 도커는?

다음 편에서 더 자세히 다루겠지만, 미리 결론만 말하면 이렇습니다.

- **개발(코딩) 중에는 도커를 거의 안 씁니다.** `npm run dev`가 훨씬 빠르고 쾌적해요.
- **도커는 주로 "배포되는 환경을 만들고 / 고정하고 / 재현하는" 용도**로 씁니다.

즉 FE에게 도커는 "빠른 코딩 도구"가 아니라 "배포 도구"에 가깝습니다. 이 감각이 왜 중요한지는 3편(트러블슈팅)에서 제가 직접 겪은 삽질과 함께 풀어볼게요.

다음 편에서는 실제로 **Dockerfile을 한 줄씩 뜯어보고, docker-compose가 왜 필요한지**를 정리합니다.
