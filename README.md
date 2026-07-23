# Korean Stock RWA — 규제부터 거래까지

외국인 투자자가 **한국 상장주식**을 온체인에서 보유·거래할 수 있게 하는 레퍼런스 구현.

이 레포는 **명세(spec)가 먼저 있고 구현이 따라오는** 구조다. `docs/` 아래 문서가 단일 진실 공급원(SSOT)이며,
AI 에이전트를 포함한 모든 구현자는 코드를 쓰기 전에 해당 문서를 읽고, 문서와 다르게 구현해야 할 이유가 생기면
**코드가 아니라 문서를 먼저 고친다.**

---

## 한 문단 요약

발행회사가 개정 전자증권법상 **발행인계좌관리기관**이 되어 자신이 발행한 주식의 투자자별 계좌부를
분산원장으로 직접 관리한다. 그 결과 온체인 토큰은 주식에 대한 *청구권*이 아니라 **전자등록주식 그 자체**이고,
온체인 이전이 곧 주식 양도다. 대상 주식은 **미리 수탁**되어 있으므로 투자자 간 매매(2차)는 즉시 체결되고,
T+2·환전 시차는 수탁 물량이 바뀌는 발행·환매(1차)에만 발생한다. 규제 준수는 ERC-3643이 매 이전마다
코드로 강제하고, 사용자는 AI 에이전트에게 자연어로 주문한다.

---

## 읽는 순서

| # | 문서 | 내용 |
|---|------|------|
| — | [AGENTS.md](./AGENTS.md) | **구현 에이전트 작업 규칙 — 코드 쓰기 전 필독** |
| 00 | [docs/00-cycle-overview.md](./docs/00-cycle-overview.md) | 전체 사이클과 4개 트랙의 연결 |
| 01 | [docs/01-regulation.md](./docs/01-regulation.md) | 규제 요건, 발행인계좌관리기관 구조 |
| 02 | [docs/02-infrastructure.md](./docs/02-infrastructure.md) | 네트워크·브릿지·오라클, DvP |
| 03 | [docs/03-tokenization.md](./docs/03-tokenization.md) | ERC-3643·ERC-7540, 1차/2차 시장 |
| 04 | [docs/04-agent-ux.md](./docs/04-agent-ux.md) | 자연어 주문 에이전트 |
| 10 | [docs/10-domain-model.md](./docs/10-domain-model.md) | 엔티티·상태기계·불변식 |
| 11 | [docs/11-interfaces.md](./docs/11-interfaces.md) | 컨트랙트/서비스 인터페이스 |
| 12 | [docs/12-acceptance.md](./docs/12-acceptance.md) | 인수 시나리오 (구현 완료 판정 기준) |
| 13 | [docs/13-open-decisions.md](./docs/13-open-decisions.md) | 미결정 사항 — 임의로 정하지 말 것 |
| 20 | [docs/20-team-implementation-guide.md](./docs/20-team-implementation-guide.md) | 비개발 팀원을 위한 트랙별 구현·통합 플레이북 |
| 21 | [docs/21-sprint-roadmap.md](./docs/21-sprint-roadmap.md) | **주차별 스프린트 로드맵 (→ 9/5 라이브 데모)** |
| — | [docs/glossary.md](./docs/glossary.md) | 용어집 (한/영 대응) |

---

## 팀 구현 시작

각 트랙이 무엇을 읽고, 무엇을 구현하고, 다른 트랙에 무엇을 넘겨야 하는지는
**[트랙별 구현·통합 플레이북](./docs/20-team-implementation-guide.md)** 에 정리돼 있다.
트랙별 코드를 마지막에 한꺼번에 합치지 말고, 공개 인터페이스와 모의 구현부터 작은 PR로 계속 통합한다.

---

## 구현 범위

**목표.** 아래 시나리오가 테스트로 통과하는 것.

> 두바이 거주 투자자 Alice가 신원 검증을 마치고, 미리 수탁된 SK하이닉스 주식 토큰 100주를
> 2차 시장에서 즉시 매수하고, 배당을 투자자별 조약 세율로 수령하고, 40주를 다른 투자자 Bob에게
> 즉시 매도하고, 수탁 물량 조정이 필요한 발행·환매만 T+2 대기를 거친다.
> 이 모든 과정을 "하이닉스 500만 원어치 사줘" 같은 자연어로 지시할 수 있다.

**비목표(non-goals).** 아래는 이번 구현에서 다루지 않는다.
- 실제 KSD 대외 연계 접속 (계약 기관만 접근 가능 → **모의 어댑터**로 대체)
- 실제 KRW 환전·외국환 신고 처리 (오프체인 절차로 가정, 상태만 표현)
- 실물 주식 매매 주문 집행 (수탁기관 API는 모의)
- 프로덕션 수준 키 관리·감사·인가 (PoC 범위의 권한 모델만)
- 법률 의견 생성. 이 레포의 법령 해석은 설계 근거이지 자문이 아니다.

---

## 기술 스택 (기본값)

| 영역 | 선택 | 비고 |
|------|------|------|
| 컨트랙트 | Solidity + **Foundry** | 테스트·배포 스크립트 포함 |
| 토큰 표준 | **ERC-3643**(전송 제한), **ERC-7540**(비동기 발행·환매) | [03](./docs/03-tokenization.md) |
| 신원 | **ONCHAINID** (ERC-734/735) | 개인정보 원본은 온체인 금지 |
| 자산 원장 | OP Stack 계열 L2 (로컬은 Anvil) | [13](./docs/13-open-decisions.md) 참조 |
| 결제 원장 | Hyperledger Besu (로컬은 별도 Anvil 인스턴스로 대체 가능) | 동상 |
| 크로스체인 | Chainlink CCIP (로컬은 모의 릴레이) | 동상 |
| 오케스트레이터 | TypeScript 서비스 | commit/abort 판단 |
| 에이전트 | TypeScript + LLM tool-calling | [04](./docs/04-agent-ux.md) |

> 스택 변경이 필요하면 [13-open-decisions.md](./docs/13-open-decisions.md)에 근거와 함께 기록한 뒤 진행한다.

---

## 디렉터리 계획

```
contracts/          Foundry 프로젝트
  src/
    token/          ERC-3643 주식 토큰 + Modular Compliance 모듈
    identity/       ONCHAINID 레지스트리 연동
    primary/        ERC-7540 발행·환매 볼트
    oracle/         한도·가격 피드 컨슈머
  test/             인수 시나리오와 1:1 대응
services/
  orchestrator/     1차 시장 DvP commit/abort
  custody-adapter/  수탁·KSD 모의 어댑터
  agent/            자연어 주문 → 온체인 실행
docs/               이 명세 (SSOT)
```

---

## 배포 대상

현재는 **로컬 개발 환경(Anvil + 모의 서비스)만** 지원한다. 테스트넷·메인넷 배포는 PoC 범위 밖이다.
실제 배포 전에는 `docs/13-open-decisions.md`의 D-B, D-C 미결 항목을 모두 해소해야 한다.

---

## 시작하기

```bash
# 컨트랙트
cd contracts && forge build && forge test

# 서비스
cd services/<name> && pnpm i && pnpm test
```

구현을 시작하기 전에 **[AGENTS.md](./AGENTS.md)를 먼저 읽어야 한다.**
