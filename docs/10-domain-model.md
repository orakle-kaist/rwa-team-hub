# 10 — 도메인 모델

---

## 1. 불변식 (Invariants) — 협상 대상 아님

구현·리팩터링·최적화 어느 단계에서도 아래는 깨지면 안 된다.
가능하면 테스트로 강제한다.

| ID | 불변식 |
|----|--------|
| **INV-1** | 온체인 토큰 잔고의 합 = 발행인계좌관리기관 계좌부상 총 전자등록 수량 |
| **INV-2** | 토큰을 보유한 모든 주소는 필수 클레임을 유효하게 보유한다 |
| **INV-3** | 어떤 이전도 5개 컴플라이언스 모듈을 모두 통과해야 성립한다 (발행 포함) |
| **INV-4** | 개인정보 원본 및 그 해시는 온체인에 존재하지 않는다 |
| **INV-5** | 외국인 총보유 비율은 오라클 공급값으로만 판정한다 (온체인 산출 금지) |
| **INV-6** | 2차 매매는 단일 트랜잭션에서 완결된다 (중간 대기 상태 없음) |
| **INV-7** | 1차 발행에서 실물 확보 확인(attestation) 전에 토큰이 발행되지 않는다 |
| **INV-8** | 모든 실패는 명시적 상태로 관측 가능하다 (조용한 실패 없음) |
| **INV-9** | 스테일 오라클 데이터로는 한도 검사를 통과시키지 않는다 |

---

## 2. 엔티티

### Investor
```
investorId          내부 식별자 (개인정보 아님)
onchainIdAddress    ONCHAINID 컨트랙트 주소
walletAddress       보유 지갑 (적격 보유 화이트리스트 등재 대상)
claims              KYC / 적격성 / 관할 / 조세거주성 / 외국환신고
```
> `claims`는 온체인에 **존재 여부와 issuer 서명**만 있다. 원본 값은 오프체인.

### StockToken (ERC-3643)
```
symbol              종목 (예: 000660)
isin                국제 증권 식별번호
totalSupply         발행량 = 수탁 실물 수량 (INV-1)
identityRegistry    ONCHAINID 검증
modularCompliance   5개 모듈
```

### ComplianceModule (5개)
```
1 IdentityEligibility    필수 클레임 보유 여부
2 Lockup                 보호예수·동결 수량
3 EligibleHolderList     적격 보유 주소
4 SanctionsScreening     제재 대상 여부
5 ForeignOwnershipLimit  외국인 한도 (오라클 의존)
```

### PrimaryRequest (ERC-7540)
```
requestId
kind                deposit | redeem
investor
amount
status              아래 상태기계 참조
createdAt / settledAt
failureReason?      실패 시 필수
```

### OracleFeed
```
feedType            price | foreignOwnership | corporateAction | reserveProof
symbol
value
updatedAt           스테일 판정에 사용 (INV-9)
source              권위 기관 (KRX / KSD / 수탁기관)
```

### Mandate (에이전트 위임)
```
maxNotionalPerOrder / maxNotionalPerDay
allowedSymbols / allowedActions
expiresAt
requireHumanApprovalAbove
```

---

## 3. 상태기계

### 3.1 2차 매매 — 상태 없음 (동기)

```
[요청] ──canTransfer() 5개 모듈──> [체결 완료]
                    │
                    └──실패──> revert (거래 자체가 성립하지 않음)
```

> 중간 상태가 없다. 성공하면 확정, 실패하면 아무 일도 일어나지 않는다. (INV-6)

### 3.2 1차 발행 (Creation)

```
                 requestDeposit()
                        │
                        ▼
                    ┌────────┐
                    │Pending │◄──── 오프체인: 환전 · 실물 매수 · 계좌대체 (T+2)
                    └────┬───┘
          custody attestation │        실패
                        ▼            └──────► [Failed] ──► void / refund / manualReview
                   ┌──────────┐
                   │Claimable │
                   └────┬─────┘
                    claim() │ (컴플라이언스 모듈 전체 검사)
                        ▼
                   ┌────────┐
                   │ Minted │
                   └────────┘
```

### 3.3 1차 환매 (Redemption)

```
   requestRedeem()        토큰 잠금
        │
        ▼
   ┌────────┐
   │Pending │◄──── 오프체인: 실물 매도 · 재환전 (T+2)
   └────┬───┘
        │ fulfillRedeem()  (소각 — 적격성 검사 미적용)
        ▼                        실패 ──► [Failed] ──► 잠금 해제 / manualReview
   ┌──────────┐
   │  Burned  │
   └────┬─────┘
        │ withdraw()
        ▼
   ┌──────────┐
   │ Settled  │
   └──────────┘
```

### 3.4 크로스체인 DvP (1차 시장 대금 결제)

```
LOCK ──► VERIFY ──► FUND ──► COMMIT ──► [완결]
                       │
                       └── 시간 초과 / 조건 미충족 ──► ABORT ──► [양쪽 에스크로 해제]
```

---

## 4. 권한 모델

| 액션 | 권한 주체 | 비고 |
|------|-----------|------|
| 클레임 발급·철회 | ClaimIssuer | 오프체인 심사 결과 반영 |
| 적격 보유 화이트리스트 갱신 | CustodyAdapter | 실물 확보 확인 후 |
| 발행(mint) | IssuerAccountManager | 컴플라이언스 검사 필수 (R6) |
| 소각(burn) | IssuerAccountManager | 환매 경로 |
| 동결·강제이전 | IssuerAccountManager | 법원 명령·오류 정정 등 예외 처리 통로 |
| 오라클 값 갱신 | Oracle | 권위는 KRX/KSD, 오라클은 전달자 |
| DvP commit/abort | Orchestrator | Finality 정책 적용 |

> **관계:** `IssuerAccountManager`는 `IStockToken.mint()`와 `burn()`을 호출하는 **유일한 주소**다. 직접 토큰 잔고를 조작하는 다른 경로는 존재하지 않는다.

> 모든 권한 행사는 **감사 가능한 이벤트**를 남긴다. 누가·언제·어떤 근거로 행사했는지 추적 가능해야 한다.
> 편의를 위해 컴플라이언스를 우회하는 관리자 함수를 만들지 않는다. → `AGENTS.md` §6

---

## 5. 데이터 배치 요약

| 데이터 | 위치 | 이유 |
|--------|------|------|
| 클레임 존재 여부 + issuer 서명 | 온체인 | 검사에 필요 |
| 개인정보 원본·해시 | **오프체인** | 파기의무 (INV-4) |
| 토큰 잔고 = 계좌부 | 온체인 | 이게 법적 원장 |
| 외국인 총보유 비율 | 오라클 → 온체인 | 온체인 산출 불가 (INV-5) |
| 실물 수탁 상태 | 오프체인 → attestation | 어댑터가 온체인에 반영 |
| 심사 서류 | 오프체인 | 개인정보 |
