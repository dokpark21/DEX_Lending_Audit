# Upside DEX Audit - kyrie

---

## 목차

- https://github.com/ooMia/Upside_DEX_solidity
- https://github.com/rivercastleone/DEX_solidity
- https://github.com/je1att0/DEX_solidity

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

---

##### Repo: https://github.com/ooMia/Upside_DEX_solidity

##### commit: 3dc4e1869aa0e942963de705450de1ee9d25f70a(Remove Remappings)

### 1-1. Type conversion in refresh

- Root cause: Dex.sol:53,54
- Severity: Medium

##### Description

> int256(bx - balanceX)에서 bx와 balanceX는 type uint256입니다. 만약 공격자가 많은 양의 토큰을 해당 컨트랙트로 직접 전송해 되어 결과 값이 type(int256).max < bx - balanceX < type(uint256).max 가 된다면 의도치 않은 overflow가 발생할 수 있습니다. 그렇게 되면 직접 직접 balanceX or balanceY를 수정할 때 까지 addLiquidity 및 removeLiquidity 호출 불가.

##### Recommendation

> dy, dy 증감량을 파악해 직접 토큰 transfer 통해 유동성 풀의 토큰 비율을 유지할 수 있으나 예기치 못한 overflow가 발생 가능하기 때문에 전체적인 로직 변화보단 유효성 검사 및 해당 상황에 호출할 함수 정의가 필요합니다.

1. dx, dy를 구하기 전에 결과 값 검사
2. 공격자의 토큰 전송으로 인한 overflow 상황에서 bx,by를 조정할 수 있는 함수 선언(다른 곳으로 transfer or burn)

### 1-2. LpBalances in removeLiquidity()

- Root cause: Dex.sol:99,100
- Severity: Critical

##### Description

> 1. test 통과만을 위해 전체 lpBalance로 부터 유동성 양에 대한 토큰 비율을 산출하는 것이 아닌 msg.sender 단일 타켓의 lpBalance로 부터 반환할 토큰의 양을 계산하고 있습니다. 사용자 수가 2명 이상이 된다면 사용자가 풀에 기여한 유동성 정도를 파악할 수 없습니다.
> 2. 전체 유동성 공급량을 추적하고 있지 않습니다.

##### Recommendation

> 1. lpBalances[msg.sender] -> total lp balance
> 2. ERC20 totalSupply() or 전역변수를 통해 추적

### 1-3. Update Balances After Swap()

- Root cause: Dex.sol:109-136
- Severity: Critical

##### Description

> swap() 호출 뒤에 바로 swap()이 호출된다면 balanceX, balanceY가 업데이트 되지 않은 채로 다시 호출되게 됩니다. swap() 비율이 풀에 존재하는 토큰 비율을 따라가지 못하게 됩니다. 사용자들이 토큰 반환량을 예측하지 못하게 됩니다.

##### Recommendation

> 1. balance update modifier 사용 -> refresh()
> 2. ERC20 balanceOf() 사용
> 3. swapX() or swapY() 실행 후 balance update

### 1-4. Zero division error

- Root cause: Dex.sol:67
- Severity: Informational

##### Description

> AddLiquidity amountX, amountY가 모두 0이 전달 된다면 조건 검사 시 zeroDivisionError 발생

##### Recommendation

> refresh() or AddLiquidity()에서 amountX,Y가 0인지 검사

## Repo 2.

---

##### Repo: https://github.com/rivercastleone/DEX_solidity

##### commit: edf6e78c4b63258f408a29ef74f29b5ae6c88501(complete dex test)

### 2-1. Unbalance Token Pair After addLiquidity()

- Root cause: Dex.sol:45,46
- Severity: Critical

##### Description

> 이미 유동성 공급이 된 상태에서 누군가 addLiquidity() 함수 호출 시에 반환할 lp token의 양은 tokenX와 tokenY로 계산한 값 중 더 작은 값 입니다. 만약 두 값이 다르단 말은 사용자가 보낸 토큰의 양이 기존의 토큰 페어의 비율과는 맞지 않다는 말 입니다.
> 반환활 lp token의 양을 더 작은 값으로 조정했다면 해당 값을 계산한 페어 토큰의 받는 양도 같이 조절해야 이후에도 토큰 페어의 비율이 일정하게 됩니다. 하지만 addLiquidity에서는 amountX, amountY를 그대로 transferFrom에 사용하고 있습니다. 이렇게 되면 유동성 공급때는 비율이 틀어지게 됩니다. 사용자는 수령한 LP Token 양에 비해 더 많은 유동성을 공급한 것이 됩니다.

##### Recommendation

> tokenX,Y를 받아오기 전 받아올 토큰의 양을 조정합니다.

- X가 작다면 Y값 조정 <-> Y가 작다면 X값 조정

---

### 2-2. Handling Liquidity

- Root cause: Dex.sol
- Severity: informaitonal

##### Descripion

> 현재 사용자들의 유동성과 총 유동성 수치를 변수와 맵으로 관리하고 있습니다. 이런식으로 관리해도 관리할 수는 있지만 ERC20 표준을 사용하여 유동성 토큰으로 관리하는 것이 더 많이 이점이 존재합니다.

##### Recommdation

> 1. **표준화 및 상호 운용성**
> 2. **보안 및 투명성**: ERC20 토큰은 표준 함수(balanceOf, transfer, approve, transferFrom 등)를 통해 보안 및 투명성을 제공합니다. 이는 코드 검증과 사용자가 소유한 유동성 지분을 쉽게 추적할 수 있게 합니다.
> 3. **액세스 제어 및 보안 기능**: ERC20 표준을 사용하면 토큰 전송 및 소각 시의 액세스 제어를 쉽게 구현할 수 있습니다. 예를 들어, Ownable 또는 AccessControl과 같은 모듈을 사용하여 특정 기능에 대한 액세스를 제한할 수 있습니다.

- LP Token을 발행 후 사용

## Repo 3.

##### Repo: https://github.com/je1att0/DEX_solidity

##### commit: a5981c8297cdee9ab8aae232a32743319445efe5(remove forge fmt)

### 3-1. Miss Authorize check in removeLiquidity()

- Root cause: Dex.sol:49
- Severity: Critical

##### Description

> removeLiquidity()에서 msg.sender가 유동성 공급자인지 확인하지 않고 있습니다.
> 유동성 공급을 하지 않은 사용자이더라도 풀에 충분한 유동성이 공급되어 있다면 원하는 만큼 가져갈 수 있습니다.
> 또한 유동성 공급 시 LP token을 전송해주지만 유동성을 제거했을 때는 해당 수량만큼 다시 가져오거나 삭제시키지 않고 있습니다.

##### Recommdation

> 1. msg.sender가 공급한 유동성 양의 가치가 출금하려는 토큰의 가치보다 큰지 검사해야 합니다.
> 2. 토큰 전송 후 해당 사용자의 유동성 token을 burn 시켜야 합니다.

---

### 3-2. Initial Supply

- Root cause: Dex.sol:8
- Severity: Critical

##### Description

> LP Token을 초기에 해당 DEX 컨트랙트에 최대 물량을 민팅하고 사용자들에게 transfer하는 방식을 사용하고 있습니다. 이렇게 되면 발행된 LP Token 추적 및 관리가 어려워집니다.
> 그리고 해당 컨트랙트에 전 수량을 approve하고 있는데 필요 없어 보이는 로직 입니다.

##### Recommdation

> 컨트랙트 생성 시 토큰을 발행하지 않고 사용자들이 유동성 공급 시에는 mint, 유동성 제거 시에는 burn을 시켜 주도록 로직을 변경하는 것이 좋습니다.

---

### 3-3. Get Token Balance

- Root cause: Dex.sol:26,27,50,51
- Severity: High

##### Description

> balanceOf를 통해 Dex가 가지고 있는 토큰 X,Y의 balance를 가져오고 있습니다. 이렇게 된다면 악의적인 공격자가 컨트랙트에 다이렉트로 토큰을 보내게 된다면 풀에 존재하는 토큰 페어 비율이 변형될 수 있습니다.

##### Recommdation

> 1. 내부 전역 변수를 통해 token balance를 같이 추적합니다.
> 2. 토큰 페어의 비율을 검사하고 만약 맞지 않다면 수정합니다.

---

### 3-4. Zero token exchanges in Swap

- Root cause: Dex.sol:65
- Severity: Informational

##### Description

> 만약 swap()에 amountX,Y가 모두 0이 들어온다면 실제로 swap되는 수량은 존재하지 않지만 트랜잭션은 실행 될 것 입니다. 이렇게 불필요한 동작이 실행될 수 있습니다.

##### Recommdation

> 토큰 페어 중 한쪽만 0인 경우만 조건을 통과하도록 수정해야 합니다.

## Repo 4.

##### Repo: https://github.com/kaymin128/Dex_solidity

##### commit: 1b37751893d23ec3119147f4e8e6d804fb40d1d1

### 4-1. Token ratio imbalance in liquidity pool

- Root cause: Dex.sol:27
- Severity: Critical

##### Description

> 1. reserveX만 balanceOf를 통해 다시 가져오고 있습니다.
> 2. tokenX와 tokenY로 계산한 lpAmount가 다를 경우 더 작은 값으로 적용 시킨 뒤에 토큰 페어 중 나머지 한 토큰을 받아올 수량을 비율에 맞게 수정해야 합니다.
>    그렇게 하지 않는다면 토큰 페어 비율이 변경되어 뒤에 유동성 공급을 하는 사용자들은 손해를 보게 됩니다.

##### Recommdation

> 1. reserveX, reserveY를 모두 업데이트 하되 누군가 다이렉트로 보낸 토큰 수량에 대해 고려해야 합니다.
> 2. transferFrom을 통해 토큰을 받아오기 전에 받아올 토큰 수량을 조정합니다.

## Repo 5.

##### Repo: https://github.com/55hnnn/DEX_solidity

##### commit: 67b860e5c79668265596fe5997510bd0dcb4970e

### 5-1. Check Direct Transfer

- Root cause: Dex.sol:18
- Severity: High

##### Description

> 만약 악의적인 누군가가 한쪽 토큰만 풀에 전송하게 된다면 유동성 풀의 균형이 깨질수도 있게 됩니다. 또한 reserveX만 balanOf로 새로운 값을 가져오고 reserveY는 이전의 값을 그대로 사용하고 min으로 더 작은 값을 적용하고 있습니다. 실제 유동성 공급과는 다른 값 때문에 잘못된 양의 유동성이 반환될 수 있습니다.
> 또한 작은 값을 그냥 유동성 토큰 보상량으로 결정하고 넘어가는데 이렇게 풀에 존재하는 토큰의 비율을 고려하지 않고 유동성을 공급 받아버리면 유동성 풀의 비율이 일정하지 않아지기 때문에 뒤에 유동성을 공급하는 사람들은 예기치 않은 손실을 보게 됩니다.

##### Recommdation

> 내부 변수와 업데이트를 통해 풀의 잔액을 추적하고 계산하면 해당 문제를 방지할 수 있습니다.
> 아니면 직접적인 transfer로 인해 예기치 못하게 늘어난 금액에 대해서 따로 관리하는 modifier나 함수를 구현할 수 있습니다.

---

### 5-2. Initial LP Token Supply

- Root cause: Dex.sol:24
- Severity: Critical

##### Description

> Total Liquidity가 0인 상태에서 유동성 공급을 하면 유동성 공급자에게 민팅할 토큰의 양을 아래와 같은 수식으로 계산하고 있습니다.
> amountX \* amountY
> 이렇게 계산한다면 초기 공급량이 너무 많아지는 문제가 생깁니다. 초기 발행량이 많다면 이후에도 계속해서 많은 양의 토큰이 발급되게 되면 초기 유동성 공급자에게만 유리한 비율이 산출될 수 있습니다.

##### Recommdation

> 일반적으로 CPMM을 사용하는 Dex에선 sqrt를 통해 초기 발행량을 설정합니다.

---

### 5-3. Token amount rebalancing

- Root cause: Dex.sol:30
- Severity: Critical

##### Description

> tokenX와 tokenY로 계산한 lpAmount가 다를 경우 더 작은 값으로 적용 시킨 뒤에 토큰 페어 중 나머지 한 토큰을 받아올 수량을 비율에 맞게 수정해야 합니다.
> 그렇게 하지 않는다면 토큰 페어 비율이 변경되어 뒤에 유동성 공급을 하는 사용자들은 손해를 보게 됩니다.

##### Recommdation

> transferFrom을 통해 토큰을 받아오기 전에 받아올 토큰 수량을 조정합니다.

## Repo 6.

##### Repo: https://github.com/GODMuang/DEX_solidity

##### commit: d99cfb5e97a8e3b70c1a984fb875d681677e8a51

### 6-1. Integer Type

- Root cause: Dex.sol:86
- Severity: Informational

##### Description

> 예치되어 있는 X,Y를 가지고 올 때 uint112로 캐스팅을 해 받아옵니다. 이렇게된다면 만약 예치된 금액이 굉장히 커질 경우 overflow가 발생할 수 있습니다.

##### Recommdation

> 굳이 형 변환을 할 필요가 없어보입니다.

---

### 6-2. Check Direct Transfer

- Root cause: Dex.sol:26
- Severity: High

##### Description

> 다른 사용자가 직접 Dex에 보낸 토큰에 대한 수량 검사를 하고 있지 않습니다. 받아올 토큰을 우선 받은 상태에서 계산하고 있기 때문에 해당 수량에 대한 추적이 불가능하고 풀의 토큰 페어 비율이 잘못 산출되게 됩니다.
> 또한 계산 후 조건 검사까지 통과하면 그 때 토큰을 받아오는 것이 바람직한 코드 레이아웃입니다. 실패하는 트랜잭션에 대해서 필요 없는 동작을 수행하게 되고 마지막 조건 검사에서 걸리게 되어 있기 때문에 가스 최적화 부분에서도 좋지 않습니다.

##### Recommdation

> 토큰을 받기 전 반환할 lp token 계산을 진행하고 유효성 검사를 진행한 뒤에 토큰을 받아 오게 해야 합니다. 이 때 다른 유저가 직접 보낸 토큰의 양을 계산하고 풀에 존재하는 토큰 비율과 맞는지 검사를 진행해야 합니다.

### 6-3. Token amount rebalancing

- Root cause: Dex.sol:97
- Severity: Critical

##### Description

> tokenX와 tokenY로 계산한 lpAmount가 다를 경우 더 작은 값으로 적용 시킨 뒤에 토큰 페어 중 나머지 한 토큰을 받아올 수량을 비율에 맞게 수정해야 합니다.
> 그렇게 하지 않는다면 토큰 페어 비율이 변경되어 뒤에 유동성 공급을 하는 사용자들은 손해를 보게 됩니다.

##### Recommdation

> 1. transferFrom은 addLiquidity의 맨 마지막 부분에서 실행되어야 합니다.
> 2. 토큰을 가져오기 전 토큰 페어 비율을 고려해 받아올 토큰의 양을 조정해야 합니다.

## Repo 7.

##### Repo: https://github.com/choihs0457/DEX_solidity

##### commit: 973f72bc81e641d131f37cb5569628ed0f439661

### 7-1. Direct Transfer Handling

- Root cause: Dex.sol:20 (UpdateLiquidity)
- Severity: High

##### Description:

> UpdateLiquidity modifier는 유저의 잔액을 업데이트하는 데 사용되고 있지만, 외부에서 직접 전송된 토큰에 대한 처리를 따로 하지 않고 있습니다. 외부 사용자가 유동성 풀로 직접 토큰을 전송하면 이로 인해 풀의 토큰 비율이 불균형하게 될 수 있습니다. 이런 상태에서 유동성 공급이나 스왑이 발생할 경우, 풀의 상태가 왜곡되어 사용자가 예상하지 못한 결과를 초래할 수 있습니다.

##### Recommendation:

> 풀에 직접 전송된 토큰을 감지하고 처리하는 별도의 로직이 필요합니다. 이를 위해 풀의 상태가 업데이트될 때마다 실제 풀에 존재하는 잔액과 내부적으로 추적되는 잔액 간의 일치 여부를 확인하고, 불일치가 발생할 경우 풀의 비율을 조정하는 로직을 추가하는 것이 좋습니다.

### 7-2. Zero Amount Handling in addLiquidity

- Root cause: Dex.sol
- Severity: Informational

##### Description:

> addLiquidity 함수에서 amountX와 amountY의 값이 둘 다 0일 경우에 대한 검사를 하지 않고 있습니다. 현재는 lpAmount 계산에서 걸리긴 하지만, 입력 값이 잘못되었을 경우 함수의 초기에 검사를 하는 것이 더 효율적이고 명확한 로직입니다.

###### Recommendation:

> addLiquidity 함수의 시작 부분에서 amountX와 amountY가 0인지 검사하고, 0일 경우 트랜잭션을 거부하는 로직을 추가하는 것이 좋습니다. 이를 통해 불필요한 계산을 줄이고 코드의 명확성을 높일 수 있습니다.

### 7-3. Fixed Fee

- Root cause: Dex.sol (고정된 수수료 구조)
- Severity: Medium

###### Description:

> DEX의 스왑에서 고정된 수수료를 사용하고 있습니다. 고정 수수료는 풀의 상태에 따라 유동적으로 설정되지 않기 때문에, 특정 상황에서 풀의 비율이나 거래량에 맞지 않는 수수료가 부과될 수 있습니다. 이러한 경우 사용자는 예상보다 더 많은 수수료를 지불하거나, 반대로 수수료가 너무 낮아 DEX의 수익성을 해칠 수 있습니다.

##### Recommendation:

> 고정 수수료 대신, 풀의 상태나 거래량에 따라 동적으로 수수료를 조정하는 메커니즘을 도입할 수 있습니다. 예를 들어, 유동성이 낮을 때 수수료를 더 높게 설정하고, 유동성이 많을 때는 수수료를 낮추는 방식으로 풀의 안정성과 DEX의 수익성을 보장할 수 있습니다.

### 7-4. Swap Check

- Root cause: Dex.sol (swap 함수에서의 유효성 검사 부족)
- Severity: High

##### Description:

> swap 함수에서는 amountX 또는 amountY 중 하나만 0이어야 한다고 가정하고 있지만, 한 부분만 0인지 검사하는 로직이 명확하지 않습니다. 또한, 유동성 풀에 그만큼의 토큰이 실제로 존재하는지에 대한 확인이 부족합니다. 만약 유동성 풀에 충분한 토큰이 존재하지 않는데도 스왑이 진행되면, 풀의 상태가 왜곡되거나 거래가 실패할 수 있습니다.

##### Recommendation:

> swap 함수에서 amountX 또는 amountY 중 하나가 반드시 0이어야 한다는 조건을 명확하게 검사하고, 유동성 풀에 충분한 토큰이 존재하는지 확인하는 로직을 추가해야 합니다. 이를 통해 스왑 과정에서 발생할 수 있는 오류를 방지할 수 있습니다.
