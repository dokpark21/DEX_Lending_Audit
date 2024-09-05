# Upside DEX Audit - kyrie

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

### 1-1. Type conversion in refresh

- Root cause: Dex.sol:53,54
- Severity: Medium

##### Description

> int256(bx - balanceX)에서 bx와 balanceX는 type uint256입니다.
> 만약 결과 값이 type(int256).max < bx - balanceX < type(uint256).max 가 된다면 의도치 않은 overflow가 발생할 수 있습니다. 그렇게 되면 직접 직접 balanceX or balanceY를 수정할 때 까지 addLiquidity 및 removeLiquidity 호출 불가.

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
