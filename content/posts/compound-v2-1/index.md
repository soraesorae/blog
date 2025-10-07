---
date: '2025-10-04T22:15:23+09:00'
title: 'Compound v2 컨트랙트 로직 - 1'
# weight: 1
# aliases: ["/first"]
tags: [""]
author: ""
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
disableShare: true
disableHLJS: false
hideSummary: false
ShowRssButtonInSectionTermList: false
UseHugoToc: true
# cover:
#     image: "<image path/url>" # image path/url
#     alt: "<alt text>" # alt text
#     caption: "<text>" # display caption under cover
#     relative: false # when using page bundles set this to true
#     hidden: true # only hide on current single page
---

Compound는 이더리움 온체인 랜딩 프로토콜이다. 2025년 가을 현재는 TVL 순위가 떨어진 상태지만, 과거 COMP 거버넌스 토큰 출시로 2020년 defi summer를 주도했던 이력이 있다. 지금 Compound는 v3이지만 v2 코드부터 읽어보면 이해에 도움이 될 것 같아 정리해 보기로 했다.

---

[컨트랙트 코드](https://github.com/compound-finance/compound-protocol/tree/a3214f67b73310d547e00fc578e8355911c9d376)는 오픈소스로 공개되어 있다. v2는 2022년 이후로 커밋이 없다. 코드는 solidity로 작성되어 있다.

Compound에는 크게 2가지 컨트랙트가 존재한다.

- `CToken` : ERC-20을 구현한 컨트랙트로 대출, 예치 등의 핵심 기능이 포함되어 있다. abstract 컨트랙트라서 CEther 또는 CErc20 컨트랙트가 CToken 컨트랙트를 상속하게 된다. CToken을 상속받은 `C[underlying token]` 컨트랙트는 해당 underlying token(e.g., DAI, ETH)의 대출, 예치 등의 기능을 제공한다. 기초 자산은 교환비에 따라 대응되는 CToken(`C[underlying token]`)으로 변환되어 Compound에서 사용된다.
- `Comptroller` : meta cToken 느낌의 역할을 하는 컨트랙트. Compound에서 지원할 market(cToken)을 결정하고, cToken의 기능을 사용할시에 각종 verification을 수행하며 Compound의 거버넌스 토큰인 COMP 토큰 분배 관련 작업을 수행한다.

이번 글에 이자 처리와 관련된 로직을 작성하고 이후에 mint, redeem, borrow, liquidation 기능 등의 로직에 대해 작성하려고 한다.

## 이자 메커니즘

Compound에는 전체 대출 양에 대해서 이자를 메기고 그 이자를 각 borrower들이 자신이 대출한 비율만큼 이자를 부담하는 구조이다. supplier 이자는 총 borrower 이자 중에 준비금을 제외한 이자를 나눠 갖는다.

먼저, 이자와 관련되어 코드에서 반복적으로 사용되는 함수는 아래 4개다.

- accrueInterest
- getBorrowRate
- utilizationRate
- exchangeRateStored

### utilizationRate

underlying asset 기준으로 (총 빌려간 양)과 (총 현금 + 총 빌려간 양 - 준비금)의 비율이다. `borrows`에는 블록마다 발생하는 이자도 포함되어 있다. 직관적으로는 총 공급 + 발생한 이자 중 얼마만큼 대출되어서 활용되는지 나타내는 지표라고 볼 수 있다.

```solidity
// JumpRateModelV2
function utilizationRate(uint cash, uint borrows, uint reserves) public pure returns (uint) {
    // Utilization rate is 0 when there are no borrows
    if (borrows == 0) {
        return 0;
    }

    return borrows * BASE / (cash + borrows - reserves);
}
```

### getBorrowRate

```solidity
// WhitePaperInterestRateModel
function getBorrowRate(uint cash, uint borrows, uint reserves) override public view returns (uint) {
    uint ur = utilizationRate(cash, borrows, reserves);
    return (ur * multiplierPerBlock / BASE) + baseRatePerBlock;
}

// JumpRateModelV2
function getBorrowRate(uint cash, uint borrows, uint reserves) override external view returns (uint) {
    return getBorrowRateInternal(cash, borrows, reserves);
}

function getBorrowRateInternal(uint cash, uint borrows, uint reserves) internal view returns (uint) {
    uint util = utilizationRate(cash, borrows, reserves);

    if (util <= kink) {
        return ((util * multiplierPerBlock) / BASE) + baseRatePerBlock;
    } else {
        uint normalRate = ((kink * multiplierPerBlock) / BASE) + baseRatePerBlock;
        uint excessUtil = util - kink;
        return ((excessUtil * jumpMultiplierPerBlock) / BASE) + normalRate;
    }
}
```

Compound 에서는 market마다 interest model을 지정해 사용한다. 현재 배포된 cToken 컨트랙트를 확인했을 때는 대부분 JumpRateModel을 사용하는 것 같다. 이 interest model은 인터페이스만 맞추면 되기 때문에 거버넌스에 의해 갈아 끼워질 수도 있다.

`WhitePaperInterestRateModel` 은 백서에 나와있는 이자 모델로 borrower 이자율이 utilization에 따라 선형으로 증가하는 구조이다. JumpRate 모델은 kink를 기준으로 multiplier가 바뀌어서 이자율 곡선의 기울기가 커진다 . 아마도 유동성 고갈이 위험한 구간이 오면 이자를 급격히 늘려서 대출을 상환하는 쪽에 인센티브를 주기 위한 아이디어로 보인다.

### accrueInterest

Compound에서 쌓인 대출 이자를 처리하기 위해 모든 상태 변경 함수 시작 부분에 이 함수를 호출한다. 만약 블록마다 발생하는 이자를 매 블록마다 개별 borrower에 대해서 처리한다고 생각해본다면, 당연하게도 $O(n)$ 시간 복잡도 코드를 매 블록마다 실행해야 한다. 이렇게 되면 가스 효율성은 형편없어지며 누군가는 그 비싼 가스비를 부담해야 한다. Compound에서는 index라는 개념과 `accrueInterest` 함수를 모든 상태 변경 함수 시작 부분에 호출하는 것으로 이 문제를 해결한다. 

index는 partial sum과 비슷한 느낌으로 처음부터 계속 누적해서 더하거나 곱하고, 나중에 일부 구간의 값이 필요하면 나누거나 빼는 식이다. 백서에는 $\text{Index}_{a,n} = \text{Index}_{a,(n-1)} *(1+r*t)$ 이런 수식으로 나온다. `accrueInterest`함수를 이용해서 계속 index를 갱신하고, 특정 유저의 이자 계산이 필요할 때 lazy하게 해당 유저에 대해서만 계산하도록 해서 최적화 했다.

```solidity
function accrueInterest() virtual override public returns (uint) {
    /* Remember the initial block number */
    uint currentBlockNumber = getBlockNumber();
    uint accrualBlockNumberPrior = accrualBlockNumber;

    /* Short-circuit accumulating 0 interest */
    if (accrualBlockNumberPrior == currentBlockNumber) {
        return NO_ERROR;
    }

    /* Read the previous values out of storage */
    uint cashPrior = getCashPrior();
    uint borrowsPrior = totalBorrows;
    uint reservesPrior = totalReserves;
    uint borrowIndexPrior = borrowIndex;
```

1. 먼저 이전에 `accrueInterest` 함수를 호출했던 블록 이후인지 체크한다. 이자는 블록 단위로 지급되기 때문에 한 블록에서 여러 번 호출할 필요는 없다.
2. cash는 해당 cToken 컨트랙트에 있는 underlying 토큰 수량, totalBorrows는 underlying 토큰 대출해준 총 수량, totalReserves는 쌓인 준비금, borrowIndex는 현재 borrower 이자의 index이다.

```solidity
    /* Calculate the current borrow interest rate */
    uint borrowRateMantissa = interestRateModel.getBorrowRate(cashPrior, borrowsPrior, reservesPrior); // 어떻게 하면 여기서 implemenet를 찾을까?
    require(borrowRateMantissa <= borrowRateMaxMantissa, "borrow rate is absurdly high");

    /* Calculate the number of blocks elapsed since the last accrual */
    uint blockDelta = currentBlockNumber - accrualBlockNumberPrior;

    /*
     * Calculate the interest accumulated into borrows and reserves and the new index:
     *  simpleInterestFactor = borrowRate * blockDelta
     *  interestAccumulated = simpleInterestFactor * totalBorrows
     *  totalBorrowsNew = interestAccumulated + totalBorrows
     *  totalReservesNew = interestAccumulated * reserveFactor + totalReserves
     *  borrowIndexNew = simpleInterestFactor * borrowIndex + borrowIndex
     */

    Exp memory simpleInterestFactor = mul_(Exp({mantissa: borrowRateMantissa}), blockDelta); // borrow rate * time delta
    uint interestAccumulated = mul_ScalarTruncate(simpleInterestFactor, borrowsPrior); // borrow rate * time delta * 총 빌린 금액 업데이트
    uint totalBorrowsNew = interestAccumulated + borrowsPrior; // 이자를 더해서 총 빌린 금액을 업데이트
    uint totalReservesNew = mul_ScalarTruncateAddUInt(Exp({mantissa: reserveFactorMantissa}), interestAccumulated, reservesPrior); // reserveFactor 고랴헤사 세러은 준비금 
    uint borrowIndexNew = mul_ScalarTruncateAddUInt(simpleInterestFactor, borrowIndexPrior, borrowIndexPrior);

    /////////////////////////
    // EFFECTS & INTERACTIONS
    // (No safe failures beyond this point)

    /* We write the previously calculated values into storage */
    accrualBlockNumber = currentBlockNumber;
    borrowIndex = borrowIndexNew;
    totalBorrows = totalBorrowsNew;
    totalReserves = totalReservesNew;

    /* We emit an AccrueInterest event */
    emit AccrueInterest(cashPrior, interestAccumulated, borrowIndexNew, totalBorrowsNew);

    return NO_ERROR;
}
```

값 계산 부분은 실수 연산 때문에 약간 읽기가 불편하다. 감사하게도 코드 주석으로 수식이 정리되어 있어서 이걸 보면 편하다. 중요한 부분은 이 함수는 매 블록 호출되지 않을 수도 있기 때문에 이전에 실행된 블록과 현재 블록의 차이를 사용한다는 점이다.

이자는 블록 단위로 징수되지만 실제로는 이 함수 호출 시점에 몰아서 정산이 된다. 이게 왜 가능하냐면, 이자율 변경은 수식에서 확인했듯이 입금, 대출, 상환, 출금 등으로 cash, borrows가 변하거나 이자 발생으로 borrows에 이자가 더해지는 경우에 발생한다. 이 함수는 상태 변화 함수 시작 부분에서 항상 호출되기 때문에 이전 `accrueInterest` 함수 호출부터 다음 호출 전까지 구간에는 같은 이자율이 적용된다(이자 발생으로 인한 이자율 변화는 그 다음 호출부터 반영). 따라서 몰아서 정산해도 문제 없는 것이다.

약간 이해가 안 가는 부분은 이자를 처리할 때 `borrowRate^blockDelta` 가 아니라 `borrowRate*blockDelta` 로 함수 호출 사이의 블록 구간에서는 단리처럼 처리가 된다는 것이다. 추측으로는 borrowRate가 워낙작다보니 수학적으로 근사를 하는 게 아닐까 싶다. 관련된 자료를 못 찾겠다.

### borrowBalanceStored

```solidity
function borrowBalanceStored(address account) override public view returns (uint) {
    return borrowBalanceStoredInternal(account);
}

function borrowBalanceStoredInternal(address account) internal view returns (uint) {
    /* Get borrowBalance and borrowIndex */
    BorrowSnapshot storage borrowSnapshot = accountBorrows[account];
		// ...
    uint principalTimesIndex = borrowSnapshot.principal * borrowIndex;
    return principalTimesIndex / borrowSnapshot.interestIndex;
}
```

`borrowSnapshot`은 마지막으로 빌렸을 때 (원금, borrow index)를 기록해둔 것이다. 이걸로 대출 이자를 고려한 총 부채를 구한다.

현재 블록 넘버가 $n$일 때 컨트랙트 전체의 index를 전개해본다면 다음과 같을 것이다.

$$
\text{Index}_{a,n} = \text{Index}_{a,(n-1)}\times  (1+r_{n}t_{n}) \\
= \text{Index}_{a,(n-2)}\times (1+r_{n-1}t_{n-1})\times(1+r_{n}t_{n}) \\
= \text{Base} \times  \dots \times  (1+r_{n-1}t_{n-1})\times(1+r_{n}t_{n})
$$

여기서 $t$시점에 대출 했다고 하면

$$
\frac{\text{Index}_{a, n}}{\text{Index}_{a, t}}= {(1+r_{t+1}t_{t+1})\times \dots \times (1+r_{n-1}t_{n-1})(1+r_{n}t_{n})}
$$

둘을 나누면 대출 시점부터 지금 까지 쌓인 이자가 나오고 여기에 원금을 곱하면 이자를 적용한 총 부채를 구할 수 있다.

### exchangeRateStored

```solidity
function exchangeRateStored() override public view returns (uint) {
    return exchangeRateStoredInternal();
}

function exchangeRateStoredInternal() virtual internal view returns (uint) {
    uint _totalSupply = totalSupply;
    if (_totalSupply == 0) {
        return initialExchangeRateMantissa;
    } else {
        /*
         * Otherwise:
         *  exchangeRate = (totalCash + totalBorrows - totalReserves) / totalSupply
         */
        uint totalCash = getCashPrior(); // underlying token의 balance
        uint cashPlusBorrowsMinusReserves = totalCash + totalBorrows - totalReserves;
        uint exchangeRate = cashPlusBorrowsMinusReserves * expScale / _totalSupply;

        return exchangeRate;
    }
}
```

underlying 토큰을 cToken으로 변환하는 비율이다. underlying 토큰 갯수를 exchange rate로 나누면 변환된 cToken 갯수를 얻는다. 다음은 borrow rate와 supply rate 그리고 exchange rate의 관계

- $U_a$: 마켓 $a$의 utilization; $U_a=\frac{\text{borrows}}{\text{cash}+\text{borrows}-\text{reserves}}$
- $\text{borrowRate}=U_a \times \text{multiplierPerBlock} + \text{baseRatePerBlock}$
- $\text{supplyRate}=\frac{\text{borrowRate} \times \text{borrows}}{\text{cash}+\text{borrows}-\text{reserves}}\times (1-\text{reserveFactor}) = \text{borrowRate} \times U_a \times (1-\text{reserveFactor})$
    - 전체 대출 이자에서 준비금 부분을 제하고 전체 underlying 토큰 공급량으로 나눠서 underlying token 1개당 이자 산출
- $\text{exchangeRate} = \frac{\text{cash} + \text{borrows} - \text{reserves}} {\text{totalSupply}}$
    - 식을 전개해보면 mint, redeem, borrow, repay 자체로는 이 값이 바뀌지 않는다는 것을 알 수 있다. 대출 이자로 인해 borrows 값이 증가하는 경우에 증가한다. 블록 1개의 대출 이자가 증가한 상황을 가정해보자.
    - $\text{exchangeRateOld} = \frac{\text{cash} + \text{borrows} - \text{reserves}} {\text{totalSupply}} \\
    \text{exchangeRateNew} = \frac{\text{cash} + \text{borrows} - \text{reserves}+\text{borrows}\times\text{borrowRate}} {\text{totalSupply}} \\
    \text{exchangeRateNew} - \text{exchangeRateOld} = \frac{\text{borrows}\times\text{borrowRate}} {\text{totalSupply}}$
    - 결과적으로 exchange rate는 이자의 총량을 cToken 총 공급으로 나눈만큼 증가했다. exchange rate가 증가했으므로 처음 cToken을 mint한 수량보다 이자를 포함한 수량을 redeem할 수 있다.
