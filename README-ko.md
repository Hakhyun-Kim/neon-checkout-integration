# Neon 체크아웃 통합 — 한국어 요약

> 2026-09-05 검토: [최신 범위·수정·미완 항목](docs/12-review.md). 아래 설명은 이전 구현 기록도 포함합니다. 영문 README가 최신 실행 안내입니다.

영문 문서의 한국어 요약본입니다. 자세한 내용은 [README.md](README.md)와
`docs/` 아래 문서를 보세요.

## 무엇을 만들었나

3D 매치-3 디펜스 게임 [Constellation Defense](https://github.com/Hakhyun-Kim/constellation-defense)에
Neon **Hosted Checkout**을 붙였습니다. 판매 품목은 전투에 영향을 주지 않는
영구 치장품 세 개(`CELESTIAL_BANNER`·`AURORA_SPIRES`·`GOLDEN_SENTINELS`)입니다.

원래 이 게임은 서버가 전혀 없는 정적 빌드였습니다. 결제를 붙이려면 API 키를
브라우저에 둘 수 없고, devtools로 고칠 수 있는 소유 기록은 소유 기록이 아니므로
게임 최초의 서버 권한 영역이 생겼습니다. 서버가 없는 GitHub Pages 배포에서는
상점 버튼이 조용히 사라집니다.

## 흐름

1. 클라이언트가 `POST /api/store/checkout`에 **SKU와 표시 언어만** 보냅니다.
   가격도, 국가도, 통화도 보내지 않습니다.
2. 서버가 허용 목록에서 가격·통화를 붙여 Neon `POST /checkout`을 호출합니다.
3. 우리가 만든 `externalReferenceId`로 **대기 중 결제 의도**를 기록합니다.
4. 플레이어는 `redirectUrl`의 Neon 호스팅 페이지에서 결제합니다.
5. 두 갈래가 동시에 돌아옵니다.
   - **리다이렉트** — 빠르지만 아무 권한이 없습니다. 모달을 열고 폴링만 시작합니다.
   - **웹훅** — 늦을 수 있지만 지급할 수 있는 유일한 경로입니다.
6. 클라이언트가 소유 여부를 폴링하고, 늦어지면 "다시 확인" 버튼을 남깁니다.

Neon 문서가 `successUrl`이 웹훅보다 먼저 도착할 수 있다고 명시합니다. 그래서
주소창에 `?purchase=return`을 직접 입력해도 아무것도 지급되지 않습니다.

## 설계에서 중요한 네 가지

**신뢰 경계** — 브라우저는 SKU와 언어만 보냅니다. 테스트가 일부러
`price: 1, country: 'US', currency: 'USD'`를 함께 보내고 서버가 전부 무시하는지
확인합니다.

**멱등성과 재시도 코드** — Neon은 2xx가 아닌 응답을 최대 36시간 재시도합니다.
그래서 재시도로 절대 풀리지 않는 경우(모르는 참조, 처리하지 않는 이벤트 종류,
환경 불일치, 금액·계정 불일치)는 사유를 로그에 남기고 **200으로 받습니다**.
서명 실패만 403으로 남깁니다 — 설정 오류는 조용하면 안 되기 때문입니다.

**국가는 언어가 아니다** — Neon은 통화를 `playerCountry`에 강제로 맞춥니다.
청구 국가는 ①명시적 선택 ②플랫폼 지리 헤더 ③브라우저 `Accept-Language` 지역
④기본값 KR 순서로 **서버가** 정합니다. 게임의 ko/en 토글은 이 목록에 없습니다.

**원화 100배 표기** — Neon은 가격을 기본 단위의 100배 정수로 받습니다. 원화는
보조 단위가 없어서 ₩4,900이 `490000`으로 나가는데, 이는 100배 오류가 나기 쉬운
지점입니다. 배수는 상수 표 한 곳에만 두고 표시 문자열은 `Intl.NumberFormat`으로
파생시킵니다.

## 실행

자격 증명 없이 전체 구매 흐름을 재현할 수 있습니다.

```bash
git clone https://github.com/Hakhyun-Kim/constellation-defense
cd constellation-defense
npm install
npm run build
cp .env.example .env   # NEON_MOCK_CHECKOUT=1 이 이미 설정돼 있습니다
npm run serve
```

`http://127.0.0.1:8642`를 열고 **별빛 상점**에서 구매하면 됩니다. 모의 모드는
같은 원장과 같은 멱등 지급 경로를 쓰고, Neon만 호출하지 않습니다.

**체크아웃 인스펙터** — 예전 13단계 자동 투어를 대체했습니다(2026-09-05).
구매를 자동으로 만들지 않고 실제 상점을 관찰합니다(`demo=고수`면 게임은 봇이
플레이하고, 상점이 열린 동안 봇은 멈춥니다): 직접 구매하면 5단계 안내가 실제 상점
이벤트로 진행되고, 현재 빌드에서 추출한 소스 발췌와 민감정보를 가린 실시간
HTTP 기록, 3D 성에 표시되는 장식 3종과 개별 환불 버튼을 보여줍니다. 구매는
새로고침해도 유지되며 **Test refund**(또는 `.data/` 삭제)로 초기화합니다:

```
http://127.0.0.1:8642/?demo=%EA%B3%A0%EC%88%98&tour=neon&mute
```

이전 투어의 단계별 스크린샷은 역사 기록으로 남아 있습니다:
[docs/evidence/tour/](docs/evidence/tour/)

**향후 작업의 토대 — 서버 측 상점 게이트웨이 (관전 데모)** — 서버 프로세스 하나가
공유 봇 정책으로 **혼자** 플레이하고, 브라우저는 그 스냅샷을 그리는 관전자가
되는 모드도 있습니다(`start-dedicated.bat` / `./start-dedicated.command`).
이때 서버는 상점의 **유일한 클라이언트 접점**입니다: 상점 호출이 같은 소켓을
타고 계정 신원과 함께 결제 서비스로 중계되고, 지급된 장식은 모든 관전자의
공유 성에 나타납니다. 멀티플레이어 의미의 dedicated 서버는 아직 아닙니다:
프로토콜에 플레이어 행동 메시지가 없고, Unity/Unreal 파일은 장면을 그리는
뷰어가 아니라 프로토콜 스모크 테스트입니다. 진짜 dedicated 서버(서버 경유
플레이어 입력)와 그 위의 Unity/Unreal 클라이언트가 다음 단계입니다.
실행법·설계·두 모드의 결제 토폴로지·지금 되는 것과 다음 단계·검증 기록은
[13 — Store gateway / server mode](docs/13-dedicated-server.md) 참조.

샌드박스 연결 절차는 [docs/07-sandbox-checklist.md](docs/07-sandbox-checklist.md)에
단계별로 있습니다.

## 문서

**다른 게임에 Neon을 붙이려는 개발자라면** [00 — Integration guide](docs/00-integration-guide.md)부터
보세요. 흐름 전체를 한 번 훑고, 그대로 가져다 쓸 코드와 **자주 틀어지는 여섯 가지**를
정리했습니다. Unity·Unreal·모바일 클라이언트에서 무엇이 달라지는지도 포함돼 있습니다.

| 문서 | 내용 |
|---|---|
| [00 — Integration guide](docs/00-integration-guide.md) | **직접 붙일 때 여기부터.** 흐름·복사할 코드·실패 6종·이식 체크리스트 |
| [01 — Architecture](docs/01-architecture.md) | 구성 요소 · 신뢰 경계 · 데이터 모델 |
| [02 — Checkout flow](docs/02-checkout-flow.md) | 시퀀스 · 리다이렉트/웹훅 레이스 · 오리진 함정 |
| [03 — Decisions](docs/03-decisions-and-assumptions.md) | 판단 근거와 남은 과제 |
| [04 — Korean market notes](docs/04-korea-market-notes.md) | 원화 · 국내 결제수단 · 미성년자 · 환불 |
| [05 — Open questions](docs/05-open-questions-for-neon.md) | Neon에 확인이 필요한 항목 |
| [06 — AI usage](docs/06-ai-usage.md) | 도구 · 역할 분담 · 검증 방법 |
| [07 — Sandbox checklist](docs/07-sandbox-checklist.md) | 샌드박스 실행 절차 |
| [08 — Storage & identity](docs/08-storage-and-identity.md) | 저장소·신원 결정과 버린 대안들 |
| [09 — Sandbox run](docs/09-sandbox-run.md) | 샌드박스 실거래 기록 · 확인된 것 · 환불 장애 리포트 |
| [10 — What this proves](docs/10-what-this-proves.md) | 단계별로 무엇이 증명됐고 무엇이 아닌지 |
| [11 — Accounts & saves](docs/11-accounts-and-saves.md) | 인계 코드 · 서버 저장본 · 충돌 처리 |
| [12 — Current review](docs/12-review.md) | 최신 검토 · 이식 경계 · 남은 한계 |
| [13 — Store gateway / server mode](docs/13-dedicated-server.md) | 관전 데모 실행법 · 설계 · 상점 게이트웨이 · 지금 되는 것 vs 다음 단계 |
| [14 — AI command journal](docs/14-ai-command-journal.md) | 무엇을 지시했고 무엇이 바뀌었는지 — 지시 단위 기록 |
