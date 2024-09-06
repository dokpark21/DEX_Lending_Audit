# Upside DEX Audit - kyrie

---

## 목차

- https://github.com/ooMia/Upside_DEX_solidity
-

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

---

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
