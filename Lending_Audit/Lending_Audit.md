# Upside Lending Audit - kyrie

---

## 감사 범위

- **Smart Contract:** [감사 대상 스마트 계약 파일 및 위치]
- **커밋 ID:** [감사 시점의 코드 커밋 ID]

## 감사 기준

[감사에서 발견된 주요 취약점 요약. 심각도와 상태별로 분류.]

| 심각도        | 설명                                                                                                                                                                      |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Critical      | 공격 비용이 낮고(공격 성공에 많은 시간이나 노력이 필요하지 않음), 취약점이 서비스 가용성에 영향을 미치거나 재정적 이득을 얻는 등 높은 영향의 문제를 초래합니다.           |
| High          | 공격자가 서비스 운영에 명확한 문제를 일으킬 수 있는 공격을 성공시킬 수 있습니다. 공격 비용이 높더라도 공격의 영향이 매우 큰 경우 “높음”으로 간주됩니다.                   |
| Medium        | 공격자가 서비스에서 의도하지 않은 작업을 수행할 수 있으며, 이 작업은 서비스 운영에 영향을 미칠 수 있습니다. 그러나 실제 공격이 성공하기 위해서는 몇 가지 제한이 있습니다. |
| Low           | 공격자가 서비스에서 의도하지 않은 작업을 수행할 수 있지만, 그 작업이 큰 영향을 미치지 않거나 공격 성공률이 매우 낮습니다.                                                 |
| Informational | 사용자나 프로토콜에 직접적인 영향을 미치지 않는 정보성 발견 사항입니다.                                                                                                   |

## Repo 1.

##### Repo: https://github.com/55hnnn/Lending_solidity

##### Commit: 8c1d7cd397b72ae2f659543353b8db9aa18c5a62

### 1-1. Check token address in deposit()

##### root cause: DreamAcademyLending.sol:50

##### Severity: informational

##### Description

> 토큰 주소가 0이 아닌것을 받았을 때 모두 msg.sender[token]에 저장하고 있습니다. 토큰 주소가 usdc나 해당 프로토콜에서 취급하는 것이 아닌 토큰까지 이렇게 모두 저장했을 경우 스토리지 사용량이 과도하게 늘어날 수 있습니다.
> ex) 공격자가 임의로 발급한 erc20 Token

##### Recommendation

> 해당 토큰 address가 해당 프로토콜에서 취급하는 토큰 주소인지 확인해야 합니다.

### 1-2. Calculate collateral value in borrow()

##### root cause: DreamAcademyLending.sol:(\_calculateCollateralValue())

##### Severity: Critical

##### Description

> Usdc를 사용자에게 빌려줄 때 현재 사용자가 예치한 금액의 가치를 계산한 뒤 빌려줘야 합니다. 해당 프로토콜에서는 사용자가 예치한 담보의 가치를 계산할 때 eth value + usdc value를 통해 담보의 가치를 계산하고 있습니다.
> USDC를 빌려주는 렌딩 프로토콜에서는 일반적으로 담보를 ETH와 같은 자산으로 잡습니다. 사용자가 빌린 자산(이 경우 USDC)을 다시 담보로 잡는 것은 일반적이지 않습니다. 이는 프로토콜의 기본적인 원칙과도 맞지 않으며, 담보로 잡는 자산은 변동성이 있거나 채무불이행 위험에 대비하기 위해 사용됩니다

##### Recommendation

> (사용자의 담보(eth) \* ethPrice) / usdcPrice를 통해 현재 빌릴 자산 대비 담보 자산의 가치를 구하고 이 값에 LTV를 곱해 현재 사용자가 빌릴 수 있는 최대 자산을 구한 뒤 빌리길 희망하는 값과 비교해주면 좋습니다.

### 1-3. Unnecessary loop in \_updateLoanValue()

##### root cause: DreamAcademyLending.sol:(\_updateLoanValue())

##### Severity: low

##### Description

> 대출, 청산, 상환이 실행되기 전 사용자의 이자를 업데이트 하기 위해 실행되는 함수입니다. 여기서 이자 계산을 할 때 day, block 만큼 반복물을 돌며 한번씩 연산을 수행하고 최종 값을 계산하고 있습니다. 이렇게 되면 너무 많은 반복문으로 인해 사용자들은 트랜잭션 호출 시 과도한 가스를 부담해야 합니다.

##### Recommendation

> 모두 한번에 수행할 수 있는 연산들 입니다. 한번에 수행하는 것이 훨씬 효율적입니다.

---

## Repo 2.

##### Repo: https://github.com/ExploitSori/Lending_solidity

##### Commit: 75e65dc5d5670cdb8894f75da9d6b38defb5c893

### 2-1. Amount reset in deposit()

##### root cause: DreamAcademyLending.sol:(deposit())

##### Severity: Critical

##### Description

> 사용자가 이더를 예치할 경우 금액이 해당 트랜잭션으로 보낸 msg.value로 누적되는 것이 아닌 초기화가 되고 있습니다. 이렇게 될 경우 사용자는 자신이 실제로 예치한 금액 만큼 담보를 설정할 수 없게 됩니다.

##### Recommendation

> 현재는 msg.value로 다시 재할당하고 있는 부분을 누적하는 방식으로 바꿔야 합니다.

### 2-2. TransferFrom in borrow()

##### root cause: DreamAcademyLending.sol:(borrow())

##### Severity: Critical

##### Description

> 현재 로직은 담보가 충분히 있는지 검사하고 담보가 충분히 있다면 빌릴 수 있는 usdc 개수 만큼 상대방에게서 usdc를 전송 받고 사용자가 빌리길 원하는 만큼의 이더를 전송해 주고 있습니다.
> 우선 담보 확인을 하는 이유는 이미 사용자가 빌릴 금액보다 더 높은 가치의 담보를 예치한 경우인지 확인하기 위한 것이고 만약 해당 조건이 통과가 된다면 굳이 사용자에게서 무언가를 안받아와도 됩니다.

##### Recommendation

> transferFrom으로 상대방의 자산을 받아오면 안됩니다.
> 충분한 담보가 설정되어 있는지 확인하고 그렇다면 빌려주고 아니라면 거부하면 됩니다.

### 2-3. check msg.value in repay()

##### root cause: DreamAcademyLending.sol:(repay())

##### Severity: Critical

##### Description

> ETH를 빌린뒤에 다시 repay하려는 상황이라면 사용자가 빌린 만큼(이자 포함)의 금액을 다시 전송했는지 확인해야 합니다. 그럴려면 msg.value를 확인해야 하는데 하지 않고 있습니다.

##### Recommendation

> msg.value가 amount 만큼 전송되었는지 확인하는 로직을 추가해야 합니다.
> 예치된 이더의 금액은 확인할 필요 없습니다.

## Repo 3.

##### Repo: https://github.com/je1att0/Lending_solidity

##### Commit: 7bb6357b7e5742133581be4acafe811bbbe3ee77

### 3-1. Check msg.sender in array

##### root cause: DreamAcademyLending.sol:54

##### Severity: Informational

##### Description

> 사용자의 잔고가 0이라면 예치한 사람들을 관리하는 배열에 추가하고 있습니다. 만약 사용자가 예치한 뒤 청산을 당했거나 출금을 했을 경우에도 0일 수 있고 해당 경우에는 중복되어 배열에 들어가기 때문에 문제가 발생할 수도 있습니다.

##### Recommendation

> 잔고를 통해 확인하기 보다는 구조체 안에 해당 상태를 표시하는 변수를 사용하거나 직접 배열을 돌며 확인합니다.

### 3-2. check msg.sender

##### root cause: DreamAcademyLending.sol:153

##### Severity: Critical

##### Description

> 사용자가 usdc 출금 요청을 했을 때 아무런 검사를 하지 않고 msg.sender에게 전송하고 있습니다. 프로토콜에 amount 만큼의 usdc가 있다면 누구나 토큰을 빼갈 수 있게 됩니다. 그리고 보낸 이후에도 유저의 정보를 업데이트 해야 합니다.

##### Recommendation

> msg.sender가 deposit한 금액을 확인해야 합니다.
> 토큰 전송 뒤에 전송한 금액을 차감해야 합니다.

## Repo 4.

##### Repo: https://github.com/rivercastleone/Lending_solidity

##### commit: ed0745f733124246d892e5433e5500e520936647

### 4-1. check block number before calculate interest

##### root cause: DreamAcademyLending.sol:(repay())

##### Severity: low

##### Description

> 대출, 출금, 상환 모든 동작에서 사용자들의 이자와 보상을 계산하기 위해 interest()가 호출되고 전체 이자에서 예치자들의 보상을 계산을 하기 위해 distributeInterest()가 호출되고 있습니다. 사용자가 많아질수록 해당 동작에 대한 가스 부담이 심해질 것 입니다.

##### Recommendation

> 마지막 업데이트 블록을 통해 불필요한 동작을 막거나 굳이 모든 사용자의 이자 보상을 계산할 필요 없을 경우를 대비해 distributeInterest()를 interest()와 분리하는 것이 좋아보입니다.

## Repo 5.

##### Repo: https://github.com/oliverslife/Lending_solidity

##### commit: a88436873ff204bcc0cc337eddefe94782384ef3

### 5-1. TranferFrom usdc in repay()

##### root cause: DreamAcademyLending.sol:(repay())

##### Severity: Critical

##### Description

> repay() 부분에서 이자를 계산하고 사용자의 대출 금액을 차감하고 있지만 실제 usdc는 받아오고 있지 않습니다. 이렇게 되면 유저들은 빌린 뒤 해당 함수를 호출해 자신의 대출금액을 차감하고 토큰은 보내지 않는식으로 계속해서 공격할 수 있습니다.

##### Recommendation

> 사용자의 대출 금액을 업데이트 한 뒤에 transferFrom을 통해 실제 자산을 받아와야 합니다. 그리고 차액에 관해서는 그냥 먼저 계산한 뒤 안받는 것이 더 좋은 방법일 거 같습니다.

### 5-2. Separate check in withdraw()

##### root cause: DreamAcademyLending.sol:(withdraw())

##### Severity: Critical

##### Description

> withdraw 부분에서 이더, 토큰 출금에 관한 조건 검사를 동시에 진행하고 있습니다. 둘은 다른 자산이고 프로토콜 내부에서 사용하는 방식도 다르기 때문에 처음부터 분리해서 검사를 진행해야 합니다. 저렇게 된다면 eth에 관련된 검사가 제대로 안될 수 있습니다.

##### Recommendation

> 처음부터 두 자산을 분리하고 이더는 빌린 usdc와 이자와 출금하려는 양을 비교해야 합니다.

## Repo 6.

##### Repo: https://github.com/WOOSIK-jeremy/Lending_solidity

##### commit: 544c3aff1cac5dd1f996e451c0fcfcd93a274843

### 6-1. First Account

##### root cause: DreamAcademyLending.sol

##### Severity: High

##### Description

> 이더를 예치할 때 따로 firstAccount를 사용하고 있는데 만약 유저가 이후에 여기서 withdraw를 하게되면 firstAccount에 있는 값 보다 accounts에 있는 값이 더 작이질 수도 있고 여러 조건 검사에서 불필요한 에러가 발생할 수 있습니다.
> firstAccount의 값은 청산을 제외하면 줄이들지 않고 있기 때문에 그렇습니다.

##### Recommendation

> firstAccount를 사용할 것이라면 이것을 조금 더 제어할 수 있는 함수나 withdraw 부분에서 firstAccount 금액을 같이 조절해야 합니다.
> 굳이 사용하기 보다는 그냥 accounts에서 blocknumber로 유저가 예치한 이더 금액을 관리하면 더 좋아보입니다.

### 6-2. withdraw usdc and reentrancy attack

##### root cause: DreamAcademyLending.sol:(withdraw())

##### Severity: Critical

##### Description

> 현재 withdraw는 호출되면 무조건 eth를 출금하는 방식으로 되어 있습니다. usdc 출금 기능 또한 반드시 필요한 부분이기에 추가해야 합니다.
> 이더 잔액에 대한 업데이트가 되지 않습니다. 출금을 해도 해당 컨트랙트에서는 유저가 아직 이더를 예치한 상황처럼 보이게 됩니다.

##### Recommendation

> usdc 출금 기능은 반드시 구현되어야 합니다. 그리고 call을 통해 유저에게 value를 전송하기 전에 accounts의 amount를 반드시 value 만큼 줄여야 합니다.
