---
date: '2025-10-07T14:36:52+09:00'
title: 'Compound v2 컨트랙트 로직 - 2'
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

> 컴파운드 v2의 mint, borrow, repay, redeem, liquidate 기능 컨트랙트 내부 구현을 정리한 내용입니다. 1편은 [여기](../compound-v2-1)에서 확인할 수 있습니다.

### mint

underlying 토큰(기초 자산)을 넣고 cToken을 민팅하는 식으로 underlying 토큰을 cToken으로 바꾸는 기능. 예를들어서 DAI 토큰을 cDAI 컨트랙트에 넣고 cDAI 토큰을 mint할 수 있다. underlying 토큰 수량을 exchange rate으로 나누면 상응하는 cToken의 갯수를 얻을 수 있다.

```solidity
function mintInternal(uint mintAmount) internal nonReentrant {
    accrueInterest();
    // mintFresh emits the actual Mint event if successful and logs on errors, so we don't need to
    mintFresh(msg.sender, mintAmount);
}

function mintFresh(address minter, uint mintAmount) internal {
    /* Fail if mint not allowed */
    uint allowed = comptroller.mintAllowed(address(this), minter, mintAmount);
    if (allowed != 0) {
        revert MintComptrollerRejection(allowed);
    }
```

`cToken` 컨트랙트 함수에서 `comptroller.<function>Allowed` 함수를 호출하는 패턴이 많다. 마켓 자체에 대한 작업을 하거나 COMP 토큰 분배 기능을 주로 수행한다.

```solidity
function mintAllowed(address cToken, address minter, uint mintAmount) override external returns (uint) {
    // Pausing is a very serious situation - we revert to sound the alarms
    require(!mintGuardianPaused[cToken], "mint is paused");

    // Shh - currently unused
    minter;
    mintAmount;

    if (!markets[cToken].isListed) {
        return uint(Error.MARKET_NOT_LISTED);
    }

    // Keep the flywheel moving
    updateCompSupplyIndex(cToken);
    distributeSupplierComp(cToken, minter);

    return uint(Error.NO_ERROR);
}
```

`mintAllowed` 함수에서는 1. pause 상태인지, 2. cToken이 comptroller 마켓 리스트에 있는지를 체크한다. 그다음 나오는 COMP과 관련된 로직은 나중에 몰아서 보는 쪽이 좋을 것 같다.

```solidity
// mintAllowed
    /* Verify market's block number equals current block number */
    if (accrualBlockNumber != getBlockNumber()) {
        revert MintFreshnessCheck();
    }

    Exp memory exchangeRate = Exp({mantissa: exchangeRateStoredInternal()});

    /////////////////////////
    // EFFECTS & INTERACTIONS
    // (No safe failures beyond this point)

    // ...
    uint actualMintAmount = doTransferIn(minter, mintAmount);
    // ...
    uint mintTokens = div_(actualMintAmount, exchangeRate);
    // ...
    totalSupply = totalSupply + mintTokens;
    accountTokens[minter] = accountTokens[minter] + mintTokens;

    /* We emit a Mint event, and a Transfer event */
    emit Mint(minter, actualMintAmount, mintTokens);
    emit Transfer(address(this), minter, mintTokens);

    /* We call the defense hook */
    // unused function
    // comptroller.mintVerify(address(this), minter, actualMintAmount, mintTokens);
}
```

`doTransferIn`이 토큰 전송을 위해 외부 컨트랙트 호출을 수반하다 보니까 CEI 패턴을 지켰다. 로직은 `doTransferIn`함수를 통해 underlying token을 받고, 실제로 받은 토큰 수량을 exchange rate로 나눈 갯수만큼 cToken을 민팅한다. `doTransferIn` 함수는 ERC-20 `transferFrom` 로 기초 자산을 입금 받는데, 토큰의 `transfer` 함수 구현에 따라 전송 실패해도 아무것도 반환하지 않거나 input 토큰 수와 실제 받은 토큰 수량이 다를 수 있기 때문에 실제로 받은 토큰 수량을 반환한다.

### borrow

```solidity
function borrowInternal(uint borrowAmount) internal nonReentrant {
    accrueInterest();
    // borrowFresh emits borrow-specific logs on errors, so we don't need to
    borrowFresh(payable(msg.sender), borrowAmount);
}

function borrowFresh(address payable borrower, uint borrowAmount) internal {
    /* Fail if borrow not allowed */
    uint allowed = comptroller.borrowAllowed(address(this), borrower, borrowAmount);
    if (allowed != 0) {
        revert BorrowComptrollerRejection(allowed);
    }

    // check ...
```

같은 패턴으로 borrow → borrowInternal → accrueInterest → borrowFresh → borrowAllowed 호출 흐름이다.

```solidity
function borrowAllowed(address cToken, address borrower, uint borrowAmount) override external returns (uint) {
    // Pausing is a very serious situation - we revert to sound the alarms
    require(!borrowGuardianPaused[cToken], "borrow is paused");

    if (!markets[cToken].isListed) {
        return uint(Error.MARKET_NOT_LISTED);
    }

    if (!markets[cToken].accountMembership[borrower]) {
        // only cTokens may call borrowAllowed if borrower not in market
        require(msg.sender == cToken, "sender must be cToken");

        // attempt to add borrower to the market
        Error err = addToMarketInternal(CToken(msg.sender), borrower);
        if (err != Error.NO_ERROR) {
            return uint(err);
        }

        // it should be impossible to break the important invariant
        assert(markets[cToken].accountMembership[borrower]);
    }

    if (oracle.getUnderlyingPrice(CToken(cToken)) == 0) {
        return uint(Error.PRICE_ERROR);
    }

    uint borrowCap = borrowCaps[cToken];
    // Borrow cap of 0 corresponds to unlimited borrowing
    if (borrowCap != 0) {
        uint totalBorrows = CToken(cToken).totalBorrows();
        uint nextTotalBorrows = add_(totalBorrows, borrowAmount);
        require(nextTotalBorrows < borrowCap, "market borrow cap reached");
    }

    (Error err, , uint shortfall) = getHypotheticalAccountLiquidityInternal(borrower, CToken(cToken), 0, borrowAmount);
    if (err != Error.NO_ERROR) {
        return uint(err);
    }
    if (shortfall > 0) {
        return uint(Error.INSUFFICIENT_LIQUIDITY);
    }

    // Keep the flywheel moving
    Exp memory borrowIndex = Exp({mantissa: CToken(cToken).borrowIndex()});
    updateCompBorrowIndex(cToken, borrowIndex);
    distributeBorrowerComp(cToken, borrower, borrowIndex);

    return uint(Error.NO_ERROR);
}
```

`markets[cToken].accountMembership[borrower]` 가 false라면 cToken을 아직 유동성 계산에 포함시키지 않았다는 뜻이다. 대출을 받게 되면 이 자산의 담보, 부채를 계산해야 하기 때문에 유동성 계산에 포함시켜야 한다. `addToMarketInternal` 함수로 유동성 계산에 market(cToken)을 포함시킨다.

market의 최대 대출액을 넘기지 않는지 체크한 후에 `getHypotheticalAccountLiquidityInternal` 함수를 호출한다. 로직은 유동성이 충분해서 신규 대출을 받아도 담보 가치를 넘지 않는지를 체크하는 것이다.

```solidity
function getHypotheticalAccountLiquidityInternal(
    address account,
    CToken cTokenModify,
    uint redeemTokens,
    uint borrowAmount) internal view returns (Error, uint, uint) {

    AccountLiquidityLocalVars memory vars; // Holds all our calculation results
    uint oErr;

    // For each asset the account is in
    CToken[] memory assets = accountAssets[account];
    for (uint i = 0; i < assets.length; i++) {
        CToken asset = assets[i];

        // Read the balances and exchange rate from the cToken
        (oErr, vars.cTokenBalance, vars.borrowBalance, vars.exchangeRateMantissa) = asset.getAccountSnapshot(account);
        if (oErr != 0) { // semi-opaque error code, we assume NO_ERROR == 0 is invariant between upgrades
            return (Error.SNAPSHOT_ERROR, 0, 0);
        }
        vars.collateralFactor = Exp({mantissa: markets[address(asset)].collateralFactorMantissa});
        vars.exchangeRate = Exp({mantissa: vars.exchangeRateMantissa});

        // Get the normalized price of the asset
        vars.oraclePriceMantissa = oracle.getUnderlyingPrice(asset);
        if (vars.oraclePriceMantissa == 0) {
            return (Error.PRICE_ERROR, 0, 0);
        }
        vars.oraclePrice = Exp({mantissa: vars.oraclePriceMantissa});
```

이 함수는 view 함수라서 상태를 변화시키지는 않고 단순히 특정 주소의 유동성이 충분한지 즉, 부채가 대출 한도를 초과하지 않는지를 체크하는 용도다. 

- `account` : 대상 유저의 주소
- `cTokenModify` : 변화가 발생하는 cToken 컨트랙트의 주소. 여기서는 redeem을 호출한 cToken 컨트랙트의 주소
- `redeemTokens` : 회수할 cToken의 수량
- `borrowAmount` : 빌릴 기초 자산의 수량

Compound는 담보가 부채보다 커야 하는 과담보 대출을 제공하는 프로토콜이다. 대출 한도는 담보의 시장가격을 모두 더한 값이 아니고, 각 담보 자산의 위험성에 따라 책정된 담보 비율에 따라 시장가격의 0%~100%가 인정되어 더해진 값이다. 따라서 실제로는 부채 만큼 담보를 잡아야하는 것이 아니라 담보 비율을 감안해서 더 많이 담보를 잡아야한다. 위험한 자산은 빠른 시간에 가격이 폭락할 리스크가 있기 때문에 담보 비율 낮게 설정해서 부채가 담보 가치보다 커져버릴 가능성을 줄인다. 이더리움은 POS 전환 이후인 현재에도 블록 타임이 12초이기 때문에 큰 변동성에 대처하기 위해서는 당연한 설계다.

유저가 담보/부채를 가지고 있는 자산들을 순회하면서 가지고 있는 담보 비율을 감안한 담보 가치와 부채를 구한다. 이때 오라클에서 받아온 기초 자산의 가격을 이용해서 단위를 통일해서 계산한다. 주석에는 ether로 normalize 한다고 되어있지만, 현재는 체인링크 USD 오라클이 등록되어 있는 걸 보면 USD로 통일하는 것 같다.

```solidity
// getHypotheticalAccountLiquidityInternal
        // Pre-compute a conversion factor from tokens -> ether (normalized price value)
        vars.tokensToDenom = mul_(mul_(vars.collateralFactor, vars.exchangeRate), vars.oraclePrice);

        // sumCollateral += tokensToDenom * cTokenBalance
        vars.sumCollateral = mul_ScalarTruncateAddUInt(vars.tokensToDenom, vars.cTokenBalance, vars.sumCollateral);

        // sumBorrowPlusEffects += oraclePrice * borrowBalance
        vars.sumBorrowPlusEffects = mul_ScalarTruncateAddUInt(vars.oraclePrice, vars.borrowBalance, vars.sumBorrowPlusEffects);

        // Calculate effects of interacting with cTokenModify
        if (asset == cTokenModify) {
            // redeem effect
            // sumBorrowPlusEffects += tokensToDenom * redeemTokens
            vars.sumBorrowPlusEffects = mul_ScalarTruncateAddUInt(vars.tokensToDenom, redeemTokens, vars.sumBorrowPlusEffects);

            // borrow effect
            // sumBorrowPlusEffects += oraclePrice * borrowAmount
            vars.sumBorrowPlusEffects = mul_ScalarTruncateAddUInt(vars.oraclePrice, borrowAmount, vars.sumBorrowPlusEffects);
        }
    }

    // These are safe, as the underflow condition is checked first
    if (vars.sumCollateral > vars.sumBorrowPlusEffects) {
        return (Error.NO_ERROR, vars.sumCollateral - vars.sumBorrowPlusEffects, 0);
    } else {
        return (Error.NO_ERROR, 0, vars.sumBorrowPlusEffects - vars.sumCollateral);
    }
}
```

```solidity
// cTokenBalance * exchangeRate = underlying token 갯수
// cTokenBalance * exchangeRate * oraclePrice = underlying token USD 기준 가치
// collateralFactor(0.0~1.0)를 곱해서 담보 비율 반영
tokensToDenom = collateralFactor * exchangeRate * oraclePrice

// cToken 담보 USD 가치 누적
sumCollateral += tokensToDenom * cTokenBalance

// sumBorrowPlusEffects = 부채 + borrow/redeem 으로 인한 효과
// 해당 cToken 부채 USD 가치 누적
sumBorrowPlusEffects += oraclePrice * borrowBalance

// if asset == cTokenModify
// 감소한 redeem USD 가치 만큼 부채에 누적
// 이걸 좌변으로 넘기면 sumCollateral에서 빠지는 효과라서 이렇게 구현한 것 같음
sumBorrowPlusEffects += tokensToDenom * redeemTokens
// 증가한 부채 누적
sumBorrowPlusEffects += oraclePrice * borrowAmount
```

유저가 유동성 체크에 포함한 자산들을 순회하면서 담보 가치와 부채 각각 합계를 계산한다.

만약에 담보의 가치가 하락하거나 대출한 자산의 가격이 오르면 부채가 대출 한도를 넘어서 청산 가능한 상태가 될 위험이 있기 때문에 대출자는 부채를 상환하거나 담보를 추가로 더 넣어야 한다. 

```solidity
// borrowFresh
    /*
     * We calculate the new borrower and total borrow balances, failing on overflow:
     *  accountBorrowNew = accountBorrow + borrowAmount
     *  totalBorrowsNew = totalBorrows + borrowAmount
     */
    uint accountBorrowsPrev = borrowBalanceStoredInternal(borrower);
    uint accountBorrowsNew = accountBorrowsPrev + borrowAmount;
    uint totalBorrowsNew = totalBorrows + borrowAmount;

    /////////////////////////
    // EFFECTS & INTERACTIONS
    // (No safe failures beyond this point)

    /*
     * We write the previously calculated values into storage.
     *  Note: Avoid token reentrancy attacks by writing increased borrow before external transfer.
    `*/
    accountBorrows[borrower].principal = accountBorrowsNew;
    accountBorrows[borrower].interestIndex = borrowIndex;
    totalBorrows = totalBorrowsNew;

    /*
     * We invoke doTransferOut for the borrower and the borrowAmount.
     *  Note: The cToken must handle variations between ERC-20 and ETH underlying.
     *  On success, the cToken borrowAmount less of cash.
     *  doTransferOut reverts if anything goes wrong, since we can't be sure if side effects occurred.
     */
    doTransferOut(borrower, borrowAmount);

    /* We emit a Borrow event */
    emit Borrow(borrower, borrowAmount, accountBorrowsNew, totalBorrowsNew);
}
```

`borrowBalanceStoredInternal` 함수는 이자를 반영한 부채를 반환한다. 신규 대출액만큼 전체 대출액과 borrower의 대출액에 추가해주고 borrower의 대출 원금과 interestIndex를 갱신한다. 이제 새롭게 갱신된 대출 원금에 대해서 이자가 발생한다.

### repay (상환)

부채를 상환하는 기능

```solidity
// repayBorrowFresh
  uint accountBorrowsPrev = borrowBalanceStoredInternal(borrower);

  /* If repayAmount == -1, repayAmount = accountBorrows */
  uint repayAmountFinal = repayAmount == type(uint).max ? accountBorrowsPrev : repayAmount;

  // ...
  uint actualRepayAmount = doTransferIn(payer, repayAmountFinal);

  /*
   * We calculate the new borrower and total borrow balances, failing on underflow:
   *  accountBorrowsNew = accountBorrows - actualRepayAmount
   *  totalBorrowsNew = totalBorrows - actualRepayAmount
   */
  uint accountBorrowsNew = accountBorrowsPrev - actualRepayAmount;
  uint totalBorrowsNew = totalBorrows - actualRepayAmount;

  /* We write the previously calculated values into storage */
  accountBorrows[borrower].principal = accountBorrowsNew;
  accountBorrows[borrower].interestIndex = borrowIndex;
  totalBorrows = totalBorrowsNew;
```

로직은 단순히 부채를 갚은 만큼 빼 주고 index를 갱신하는 것이다.

### redeem

유동성 계산에 포함되지 않은 market의 경우는 마음대로 cToken을 redeem해도 문제 없다. 유동성 계산에 포함된 cToken의 경우에는 redeem하면 담보를 빼는 것이기 때문에 redeem하더라도 부채가 대출 한도 이하인지 확인한 후에 cToken을 redeem할 수 있다.


### liquidate (청산)

Compound 같은 과담보 랜딩 프로토콜에서는 열려있는 대출 포지션에서 대출 받은 기초 자산의 가격이 상승하거나 담보로 잡아둔 자산의 가치가 하락하면, 프로토콜은 담보 가치보다 부채가 많아질 위험에 놓인다. 이렇게 되면 프로토콜 입장에서는 어떻게든 포지션의 부채를 줄여야 한다. 그러나 매 블록마다 포지션이 열린 모든 유저 각각마다 유동성을 계산해서 처리하는 것은 가스비 이슈로 현실적이지 않다. 이러한 이유로 랜딩 프로토콜에는 다른 유저의 포지션 부채 일부를 대신 상환해 주고 포지션의 담보를 시세보다 약간 저렴하게 구매할 수 있게 하는 청산 기능이 존재한다. 가격 인센티브 때문에 외부의 청산자들은 off-chain에서 청산 가능한 포지션을 트래킹하고 트랜잭션을 날려 해당 포지션을 청산하고 이익을 얻는다.

Compound에서의 대출 한도는 담보 비율을 적용한 담보 자산의 가격 총합과 같다. 자산 중 높은 변동성, 낮은 유동성 등의 이유로 위험한 자산은 빠른 시간에 담보가 부채를 초과할 위험성이 있다. 그래서 자산의 위험성에 따라 담보 비율을 낮거나 높게 적용해서 리스크를 관리한다. 매우 위험한 자산은 낮은 담보 비율을 가지고, 대출 한도를 높이기 위해서는 덜 위험한 자산보다 더 높은 값어치의 자산을 담보로 잡아야 한다. Compound의 청산 기준은 단순히 부채가 대출 한도를 초과하는 경우다.

청산 가능한 상태가 되면 대신 상환을 해줘서 부채가 담보 비율을 넘지 않도록 하는 방식이다.

```solidity
function liquidateBorrowInternal(address borrower, uint repayAmount, CTokenInterface cTokenCollateral) internal nonReentrant {
    accrueInterest();

    uint error = cTokenCollateral.accrueInterest();
    if (error != NO_ERROR) {
        // accrueInterest emits logs on errors, but we still want to log the fact that an attempted liquidation failed
        revert LiquidateAccrueCollateralInterestFailed(error);
    }

    // liquidateBorrowFresh emits borrow-specific logs on errors, so we don't need to
    liquidateBorrowFresh(msg.sender, borrower, repayAmount, cTokenCollateral);
}
```

특정 cToken 컨트랙트의 `liquidateBorrow` 함수를 호출한 것은 해당 cToken의 기초자산을 대출한 포지션을 청산하겠다는 의미다. 위 함수의 인자 `borrower` 는 청산할 대출자의 주소, `repayAmount` 는 상환할 기초자산 액수, `cTokenCollateral` 은 청산한 다음에 할인된 가격에 압류할 담보 cToken의 주소.

seize 하면서 담보 cToken의 상태도 변화시키기 때문에  `cTokenCollateral.accrueInterest()` 호출로 담보의 이자도 처리해준다.

```solidity
function liquidateBorrowFresh(address liquidator, address borrower, uint repayAmount, CTokenInterface cTokenCollateral) internal {
    /* Fail if liquidate not allowed */
    uint allowed = comptroller.liquidateBorrowAllowed(address(this), address(cTokenCollateral), liquidator, borrower, repayAmount);
    if (allowed != 0) {
        revert LiquidateComptrollerRejection(allowed);
    }

    // check ...
```

```solidity
function liquidateBorrowAllowed(
    address cTokenBorrowed,
    address cTokenCollateral,
    address liquidator,
    address borrower,
    uint repayAmount) override external returns (uint) {
    // Shh - currently unused
    liquidator;

    if (!markets[cTokenBorrowed].isListed || !markets[cTokenCollateral].isListed) {
        return uint(Error.MARKET_NOT_LISTED);
    }

    uint borrowBalance = CToken(cTokenBorrowed).borrowBalanceStored(borrower);

    /* allow accounts to be liquidated if the market is deprecated */
    if (isDeprecated(CToken(cTokenBorrowed))) {
        require(borrowBalance >= repayAmount, "Can not repay more than the total borrow");
    } else {
        /* The borrower must have shortfall in order to be liquidatable */
        (Error err, , uint shortfall) = getAccountLiquidityInternal(borrower);
        if (err != Error.NO_ERROR) {
            return uint(err);
        }

        if (shortfall == 0) {
            return uint(Error.INSUFFICIENT_SHORTFALL);
        }

        /* The liquidator may not repay more than what is allowed by the closeFactor */
        uint maxClose = mul_ScalarTruncate(Exp({mantissa: closeFactorMantissa}), borrowBalance);
        if (repayAmount > maxClose) {
            return uint(Error.TOO_MUCH_REPAY);
        }
    }
    return uint(Error.NO_ERROR);
}
```

 `getAccountLiquidityInternal` 함수는 내부에서 `getHypotheticalAccountLiquidityInternal` 함수를 호출해서 결과적으로는 부채가 대출 한도를 초과했는가를 체크해서 청산 가능한 상태인지 확인한다.

한 번에 모든 초과분을 청산할 수는 없고 close factor로 한 번에 청산할 수 있는 양의 상한이 정해져 있다.

```solidity
// liquidateBorrowFresh
    /* Fail if repayBorrow fails */
    uint actualRepayAmount = repayBorrowFresh(liquidator, borrower, repayAmount);

    /////////////////////////
    // EFFECTS & INTERACTIONS
    // (No safe failures beyond this point)

    /* We calculate the number of collateral tokens that will be seized */
    (uint amountSeizeError, uint seizeTokens) = comptroller.liquidateCalculateSeizeTokens(address(this), address(cTokenCollateral), actualRepayAmount);
    require(amountSeizeError == NO_ERROR, "LIQUIDATE_COMPTROLLER_CALCULATE_AMOUNT_SEIZE_FAILED");

    /* Revert if borrower collateral token balance < seizeTokens */
    require(cTokenCollateral.balanceOf(borrower) >= seizeTokens, "LIQUIDATE_SEIZE_TOO_MUCH");

    // If this is also the collateral, run seizeInternal to avoid re-entrancy, otherwise make an external call
    if (address(cTokenCollateral) == address(this)) {
        seizeInternal(address(this), liquidator, borrower, seizeTokens);
    } else {
        require(cTokenCollateral.seize(liquidator, borrower, seizeTokens) == NO_ERROR, "token seizure failed");
    }

    /* We emit a LiquidateBorrow event */
    emit LiquidateBorrow(liquidator, borrower, actualRepayAmount, address(cTokenCollateral), seizeTokens);
}
```

원하는 만큼 상환하고 상환한 만큼 (actual이 붙은 이유는 구현에 따라 transferFrom에서 보낸 만큼 받지 않을 수도 있기 때문이다) `liquidateCalculateSeizeTokens` 는 가격 할인을 감안해서 담보 cToken을 얼마나 가져갈 수 있는지 계산하는 함수다. 아래는 주석으로 달려있는 수식이다.

```solidity
/*
 * Get the exchange rate and calculate the number of collateral tokens to seize:
 *  seizeAmount = actualRepayAmount * liquidationIncentive * priceBorrowed / priceCollateral
 *  seizeTokens = seizeAmount / exchangeRate
 *   = actualRepayAmount * (liquidationIncentive * priceBorrowed) / (priceCollateral * exchangeRate)
 */
```

 `seizeAmount` 은 가격 인센티브 감안해서 담보 underlying token을 몇 개를 압류할 것인가이고 `seizeTokens` 는 cToken으로 치면 몇 개인가다. 확인해보니 현제는 liquidation incentive는 1.08 로 되어 있다.

```solidity
function seizeInternal(address seizerToken, address liquidator, address borrower, uint seizeTokens) internal {
    /* Fail if seize not allowed */
    uint allowed = comptroller.seizeAllowed(address(this), seizerToken, liquidator, borrower, seizeTokens);
    if (allowed != 0) {
        revert LiquidateSeizeComptrollerRejection(allowed);
    }

    /* Fail if borrower = liquidator */
    if (borrower == liquidator) {
        revert LiquidateSeizeLiquidatorIsBorrower();
    }

    /*
     * We calculate the new borrower and liquidator token balances, failing on underflow/overflow:
     *  borrowerTokensNew = accountTokens[borrower] - seizeTokens
     *  liquidatorTokensNew = accountTokens[liquidator] + seizeTokens
     */
    uint protocolSeizeTokens = mul_(seizeTokens, Exp({mantissa: protocolSeizeShareMantissa}));
    uint liquidatorSeizeTokens = seizeTokens - protocolSeizeTokens;
    Exp memory exchangeRate = Exp({mantissa: exchangeRateStoredInternal()});
    uint protocolSeizeAmount = mul_ScalarTruncate(exchangeRate, protocolSeizeTokens);
    uint totalReservesNew = totalReserves + protocolSeizeAmount;

    /////////////////////////
    // EFFECTS & INTERACTIONS
    // (No safe failures beyond this point)

    /* We write the calculated values into storage */
    totalReserves = totalReservesNew;
    totalSupply = totalSupply - protocolSeizeTokens;
    accountTokens[borrower] = accountTokens[borrower] - seizeTokens;
    accountTokens[liquidator] = accountTokens[liquidator] + liquidatorSeizeTokens;

    /* Emit a Transfer event */
    emit Transfer(borrower, liquidator, liquidatorSeizeTokens);
    emit Transfer(borrower, address(this), protocolSeizeTokens);
    emit ReservesAdded(address(this), protocolSeizeAmount, totalReservesNew);
}

// CTokenStorage
uint public constant protocolSeizeShareMantissa = 2.8e16; //2.8%
```

`seizeInternal` 함수는 구매하는 담보 중 프로토콜이 가져가는 부분을 빼고 청산자의 cToken balance를 증가시키고 borrower의 balance를 감소시킨다.
