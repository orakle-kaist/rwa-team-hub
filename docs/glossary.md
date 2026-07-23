# 용어집

코드 식별자는 이 표의 **영문 식별자**를 사용한다. 문서에서는 한국어를 쓴다.

---

## 약어 표

| 약어 | 풀네임 | 한국어 |
|------|--------|--------|
| RWA | Real World Asset | 실물자산 토큰화 |
| KYC | Know Your Customer | 고객확인의무 (특금법) |
| DvP | Delivery versus Payment | 동시결제 |
| ERC | Ethereum Request for Comments | 이더리움 표준 제안 |
| KRX | Korea Exchange | 한국거래소 |
| KSD | Korea Securities Depository | 한국예탁결제원 |
| FSCMA | Financial Services and Capital Markets Act | 자본시장법 |
| IAM | IssuerAccountManager | 발행인계좌관리기관 (이 레포 내 약어) |
| ONCHAINID | — | ERC-734/735 기반 온체인 신원 프로토콜 |
| CCIP | Cross-Chain Interoperability Protocol | Chainlink 크로스체인 프로토콜 |
| PoR | Proof of Reserve | 담보 준비금 증명 |
| CRE | Chainlink Runtime Environment | DvP 오케스트레이션 실행 계층 |

---

## 제도 · 기관

| 한국어 | 영문 식별자 | 설명 |
|--------|-------------|------|
| 발행인계좌관리기관 | `IssuerAccountManager` | 발행회사가 직접 계좌관리기관이 되어 자기 발행 주식의 투자자별 계좌부를 관리. **이 설계의 핵심** |
| 일반계좌관리기관 | `GeneralAccountManager` | 기존 구조에서 투자자 계좌부를 관리하던 증권사 |
| 전자등록계좌부 | `ElectronicRegistryLedger` | 권리 이전의 효력이 발생하는 법적 원장. 이 설계에서는 온체인 분산원장이 이 역할 |
| 한국예탁결제원 | `KSD` | 중앙예탁결제기관 |
| 한국거래소 | `KRX` | 거래소. 시세·기업행위 공시 원천 |
| 명의개서대리인 | `TransferAgent` | 기준일 주주명부 작성·권리행사 대행. 토큰증권 구조에서는 역할 축소 |
| 수탁기관 | `Custodian` | 실물 주식 보관, 1차 시장 실물 매매 수행 |
| 상임대리인 | `StandingProxy` | 외국인 직접투자 시 국내 대리인 |

---

## 시장 · 거래

| 한국어 | 영문 식별자 | 설명 |
|--------|-------------|------|
| 1차 시장 | `primaryMarket` | 발행·환매. 수탁 물량이 바뀜. **대기 있음** |
| 2차 시장 | `secondaryMarket` | 투자자 간 매매. **즉시 체결** |
| 발행 | `issuance` (비즈니스) / `mint` (컨트랙트 함수) | 새 토큰 발행 (실물 확보 후). 코드 식별자는 `mint`, 문서·UI 표현은 `issuance` 또는 `발행` |
| 환매 | `redemption` (비즈니스) / `burn` (컨트랙트 함수) | 토큰 소각 후 대금 반환. 코드 식별자는 `burn`, 문서·UI 표현은 `redemption` 또는 `환매` |
| 미리 수탁 | `preCustody` | 대상 주식을 사전에 수탁해 둠. 2차 즉시 체결의 전제 |
| 동시결제 | `DvP` (Delivery versus Payment) | 자산 이전과 대금 지급의 동시 완결 |
| 최종성 | `finality` | 경제적 완결성 ≠ 법적 완결성. 구분해서 쓸 것 |

---

## 규제 · 검사

| 한국어 | 영문 식별자 | 설명 |
|--------|-------------|------|
| 판단 | `decision` | 규제 요건 충족 여부 결정. **오프체인** |
| 집행 | `enforcement` | 결정 결과를 거래에 적용. **온체인** |
| 고객확인 | `KYC` | 특금법상 고객확인의무 |
| 투자자 적격성 | `eligibility` | 자본시장법상 적격 판정 |
| 관할 | `jurisdiction` | 투자자 소재국 |
| 조세 거주성 | `taxResidency` | 조약 세율 적용 기준 |
| 외국환 신고 | `fxFiling` | 외국환거래법상 자금 유출입 신고 |
| 외국인 지분 한도 | `foreignOwnershipLimit` | 종목별 법정 상한. **종목 전체 집계 기준** |
| 제재 스크리닝 | `sanctionsScreening` | OFAC/UN/EU 제재 대상 확인 |
| 보호예수 · 락업 | `lockup` | 처분 제한 |
| 적격 보유 화이트리스트 | `eligibleHolderList` | 커스터디 구조상 보유 가능한 주소 목록 |
| 클레임 | `claim` | 검증 기관이 서명한 검증 결과. 원본 아님 |

---

## 기술 표준

| 용어 | 설명 |
|------|------|
| `ERC-3643` | 전송 제한 토큰 표준(T-REX). 이전마다 신원·규칙 검사. **"누가"** 담당 |
| `ERC-7540` | 비동기 볼트 표준. 요청→처리→수령 상태 분리. **"언제"** 담당 |
| `ERC-734/735` | 온체인 신원·클레임 (ONCHAINID) |
| `ModularCompliance` | 규제 로직을 독립 모듈로 분리·교체 가능하게 한 구조 |
| `moduleMintAction` | 발행 경로에도 컴플라이언스 모듈을 강제하는 훅 |
| `CCIP` | Chainlink 크로스체인 상호운용 프로토콜 |
| `CRE` | Chainlink Runtime Environment. DvP 오케스트레이션 실행 계층 |
| `PoR` | Proof of Reserve. 담보 준비금 증명 |

---

## 이 레포에서 쓰지 않는 용어 (사용 시 리뷰어가 반려)

| 쓰지 않음 | 이유 |
|-----------|------|
| 청구권 토큰, 래핑 토큰, 예탁증서(DR) 모델 | 이 설계에서 토큰은 **주식 자체**다 (`AGENTS.md` R3) |
| VASP 고객확인 | 디지털자산 규제. 한국 상장주식 매매에는 해당 없음 |
| 외국인투자자 등록(IRC) | 2023.12 폐지. 여권번호/LEI 식별로 대체 |
| 셰어 토큰 | `주식 토큰`(`StockToken`)으로 통일 |
