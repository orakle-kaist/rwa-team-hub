# 11 — 인터페이스 명세

> 아래는 **계약(contract)의 형태**를 정의한다. 시그니처 세부는 구현 중 조정될 수 있으나,
> 파라미터가 담는 **의미와 검사 책임**은 바꾸지 않는다. 바꿔야 하면 이 문서를 먼저 고친다.

---

## 1. 컴플라이언스 모듈 공통

```solidity
interface IComplianceModule {
    /// @notice 이전 가능 여부 검사. 실패 시 구분 가능한 이유와 함께 revert
    /// @dev view여야 한다 — 검사가 상태를 바꾸면 안 됨
    function checkTransfer(address from, address to, uint256 amount)
        external view returns (bool);

    /// @notice 이전 성립 후 모듈 내부 상태 갱신 (필요한 모듈만)
    /// @dev transfer() 성공 직후 호출된다. checkTransfer()와 달리 상태를 바꿀 수 있다
    function transferred(address from, address to, uint256 amount) external;

    /// @notice 발행 시에도 호출되어야 한다 (mint 갭 대응 — AGENTS.md R6)
    function mintAction(address to, uint256 amount) external;

    function moduleName() external pure returns (string memory);
}
```

### revert reason 규약

```solidity
error ComplianceRejected(string module, string reason);

// 외국인 한도 모듈은 사후 검증 가능하도록 추가 정보를 남긴다
error ForeignLimitExceeded(
    string  symbol,
    uint256 statutoryLimitBps,   // 법정 한도 (basis point)
    uint256 projectedRatioBps,   // 이전 후 예상 비율
    uint256 oracleUpdatedAt      // 오라클 기준 시각
);

error OracleDataStale(string feed, uint256 updatedAt, uint256 maxAge);
```

---

## 2. 주식 토큰 (ERC-3643 기반)

```solidity
interface IStockToken {
    // --- 2차 시장: 동기 · 즉시 (대기 상태 없음, INV-6) ---
    function transfer(address to, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);

    /// @notice 사전 검사. 에이전트/UI가 제출 전 확인용으로 호출
    function canTransfer(address from, address to, uint256 amount)
        external view returns (bool);

    // --- 1차 시장: IssuerAccountManager만 호출 ---
    /// @dev 컴플라이언스 모듈 검사를 반드시 거친다 (R6)
    function mint(address to, uint256 amount) external;
    function burn(address from, uint256 amount) external;

    // --- 예외 처리 통로 (감사 이벤트 필수) ---
    function freezePartialTokens(address account, uint256 amount) external;
    function forcedTransfer(address from, address to, uint256 amount) external;

    event ComplianceChecked(address indexed from, address indexed to, uint256 amount);
    event ForcedAction(string action, address indexed actor, string justification);
}
```

---

## 3. 신원 레지스트리

```solidity
interface IIdentityRegistry {
    /// @notice 필요한 topic의 클레임이 신뢰 가능한 issuer 서명으로 존재하는지만 확인
    /// @dev "이 투자자가 누구인가"를 반환하지 않는다 (INV-4)
    function isVerified(address account) external view returns (bool);

    function hasClaim(address account, uint256 topic) external view returns (bool);

    /// @notice 조세 거주성 등 배당 계산에 필요한 클레임 조회
    function claimValue(address account, uint256 topic) external view returns (bytes memory);
}
```

### 클레임 topic (프로토콜 상수 — 번호는 미결정, [13](./13-open-decisions.md) 참조)

| topic | 내용 | 발급 주체 |
|-------|------|-----------|
| KYC | 고객확인 완료 | 증권사·KYC 인증기관 |
| ELIGIBILITY | 투자자 적격성 | 증권사 |
| JURISDICTION | 거주 관할 | 발행 주체·판매사 |
| TAX_RESIDENCY | 조세 거주성 | 세무 확인 기관 |
| FX_FILING | 외국환 신고 완료 | 외국환은행 |

---

## 4. 외국인 한도 오라클 컨슈머

```solidity
interface IForeignOwnershipOracle {
    struct Reading {
        uint256 statutoryLimitBps;  // 법령 개정 시에만 변동
        uint256 currentRatioBps;    // 장중 변동, 시장 전체 집계값
        uint256 updatedAt;
    }

    /// @notice updatedAt이 maxAge()를 초과하면 revert한다 (INV-9 — 스테일 데이터로 통과시키지 않음)
    function read(string calldata symbol) external view returns (Reading memory);

    /// @notice 허용 지연(초). 초과 시 read()가 OracleDataStale로 revert
    function maxAge() external view returns (uint256);
}
```

---

## 5. 1차 시장 볼트 (ERC-7540 계열)

```solidity
interface IPrimaryMarketVault {
    // 발행
    function requestDeposit(uint256 assets, address controller) external returns (uint256 requestId);
    /// @dev CustodyAdapter의 attestation 이후에만 호출 가능 (INV-7)
    function fulfillDeposit(uint256 requestId, uint256 shares) external;
    function claim(uint256 requestId) external;

    // 환매
    function requestRedeem(uint256 shares, address controller) external returns (uint256 requestId);
    function fulfillRedeem(uint256 requestId, uint256 assets) external;
    function withdraw(uint256 requestId) external;

    // 실패 경로 — 반드시 구현 (INV-8)
    function markFailed(uint256 requestId, string calldata reason) external;
    function refund(uint256 requestId) external;

    function statusOf(uint256 requestId) external view returns (uint8);

    event RequestStatusChanged(uint256 indexed requestId, uint8 from, uint8 to, string reason);
}
```

---

## 6. 커스터디 어댑터 (오프체인 서비스 ↔ 온체인)

```typescript
interface CustodyAdapter {
  /** 실물 매수 지시 (모의). 지연·실패 주입 가능해야 함 */
  buyPhysical(symbol: string, notionalKRW: string): Promise<CustodyOrderResult>;
  sellPhysical(symbol: string, quantity: string): Promise<CustodyOrderResult>;

  /** 실물 확보 확인 → 온체인 반영. 이것 없이는 발행 불가 (INV-7) */
  attestCustody(requestId: string, quantity: string): Promise<TxHash>;

  /** 적격 보유 화이트리스트 갱신 */
  updateEligibleHolder(wallet: string, eligible: boolean): Promise<TxHash>;
}

type CustodyOrderResult =
  | { status: "settled"; quantity: string; settledAt: string }
  | { status: "pending"; expectedSettlement: string }
  | { status: "failed"; reason: string };
```

> **MOCK 규칙:** 실제 KSD 대외 연계는 계약 기관만 접근 가능하므로 타이머 기반 모의로 대체한다.
> 파일 상단에 `// MOCK:`으로 가정을 명시하고, 지연·실패 시나리오를 주입할 수 있게 만든다.

---

## 7. DvP 오케스트레이터 (오프체인 서비스)

```typescript
interface DvpOrchestrator {
  /** 1차 시장 결제에만 사용. 2차 매매는 이 경로를 타지 않는다 (R4) */
  execute(params: {
    // chain: 체인 식별자 문자열. 예: "optimism-sepolia", "besu-local"
    assetLeg: { chain: string; token: string; amount: string };
    cashLeg:  { chain: string; token: string; amount: string };
    deadline: string;
  }): Promise<DvpResult>;
}

type DvpResult =
  | { status: "committed"; assetTx: string; cashTx: string }
  | { status: "aborted"; reason: "timeout" | "compliance" | "finality" | "cash_unavailable" };
```

**단계:** `LOCK → VERIFY → FUND → COMMIT | ABORT`

**현 PoC의 Finality 정책:** 경제적 완결(트랜잭션 포함 확인)을 commit 시점으로 삼는다. 법적 완결(7일)은 오프체인 의무로 처리한다. 구체적 정책은 D-C5 결정에 따른다. → [13-open-decisions.md](./13-open-decisions.md)

---

## 8. 에이전트 서비스

```typescript
interface TradingAgent {
  handle(utterance: string, ctx: { investorId: string; mandate: Mandate }): Promise<AgentReply>;
}

type AgentReply =
  | { kind: "executed"; intent: OrderIntent; txHash: string; filled: { quantity: string; notional: string } }
  | { kind: "rejected"; intent: OrderIntent; reason: ComplianceRejection; explanation: string }
  | { kind: "needs_confirmation"; intent: OrderIntent; why: "mandate_exceeded" | "ambiguous" }
  | { kind: "needs_clarification"; missing: string[] };  // 추측 금지, 되묻기
```

> `rejected`를 `executed`로 보고하는 경로가 있으면 안 된다 (INV-8).
> 거절 우회를 위해 주문을 쪼개는 로직을 만들지 않는다.

---

## 9. 배당 분배자 (스텁 — D-B8 결정 후 확정)

> AC-8이 이 인터페이스를 요구한다. 조세 거주성 클레임 표현(D-B8) 결정 전까지는 스텁으로 구현한다.

```solidity
interface IDividendDistributor {
    /// @notice 배당기준일 스냅샷. 이 블록 기준 보유자·수량 고정
    function snapshot(string calldata symbol) external returns (uint256 snapshotId);

    /// @notice 스냅샷 기준 배당금을 투자자별 조세 거주성으로 세율 적용 후 분배
    /// @dev 조세 거주성은 IIdentityRegistry.claimValue(account, TAX_RESIDENCY_TOPIC)에서 읽음
    /// @dev TODO(decision): D-B8 — 세율 조회 인터페이스 미확정
    function distribute(uint256 snapshotId, uint256 grossAmountKRW) external;

    /// @notice 투자자 미수령 배당금 조회
    function pendingDividend(address account) external view returns (uint256);

    /// @notice 배당금 수령
    function claim() external;

    event DividendDistributed(uint256 indexed snapshotId, string symbol, uint256 totalGross, uint256 recipientCount);
    event DividendClaimed(address indexed account, uint256 netAmount, uint256 taxWithheld);
}
```
