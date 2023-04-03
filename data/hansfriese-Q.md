### [L-01] `MuteAmplifier.rescueTokens()` should check pre/post balances for 2 addresses tokens
https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L180

It's safe to check pre/post balances for 2 addresses tokens.

### [L-02] It checks the wrong conditions for fee transfers
https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L258

https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L298

https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L334

```solidity
// payout fee0 fee1
if ((fee0 > 0 || fee1 > 0) && fee0 <= totalFees0 && fee1 <= totalFees1) {
    address(IMuteSwitchPairDynamic(lpToken).token0()).safeTransfer(msg.sender, fee0);
    address(IMuteSwitchPairDynamic(lpToken).token1()).safeTransfer(msg.sender, fee1);

    totalClaimedFees0 = totalClaimedFees0.add(fee0);
    totalClaimedFees1 = totalClaimedFees1.add(fee1);

    emit FeePayout(msg.sender, fee0, fee1);
}
```

When it checks for fee payouts, it doesn't consider `totalClaimedFees0/1` at all. It should be checked like the below.

```solidity
// payout fee0 fee1
if ((fee0 > 0 || fee1 > 0) && totalClaimedFees0 + fee0 <= totalFees0 && totalClaimedFees1 + fee1 <= totalFees1) {
    address(IMuteSwitchPairDynamic(lpToken).token0()).safeTransfer(msg.sender, fee0);
    address(IMuteSwitchPairDynamic(lpToken).token1()).safeTransfer(msg.sender, fee1);

    totalClaimedFees0 = totalClaimedFees0.add(fee0);
    totalClaimedFees1 = totalClaimedFees1.add(fee1);

    emit FeePayout(msg.sender, fee0, fee1);
}
```

### [L-03] `MuteBond.setMaxPrice()` and `MuteBond.setStartPrice()` should check if `maxPrice >= startPrice`
https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L99-L113

```solidity
    function setMaxPrice(uint _price) external {
        require(msg.sender == customTreasury.owner());
        maxPrice = _price;
        emit MaxPriceChanged(_price);
    }

    /**
     *  @notice sets the start price for the LP token
     *  @param _price uint
     */
    function setStartPrice(uint _price) external {
        require(msg.sender == customTreasury.owner());
        startPrice = _price;
        emit StartPriceChanged(_price);
    }
```

When the above functions are called by the owner, `maxPrice >= startPrice` should be checked like [here](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L82).

Otherwise, [this one](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L229) will revert during deposit.

### [L-04] `MuteBond.setEpochDuration()` should check if `epochDuration > 0`
https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L129

`epochDuration` should be greater than 0, otherwise, [this line](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L234) will revert during deposit.

### [L-05] `MuteBond.setStartPrice()` should check if `startPrice > 0`
https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L109

If `startPrice = 0`, [bondPrice](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L234) might be 0 at the start time of epoch and [this line](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L249) will revert during deposit.

### [L-06] Should check `>` instead of `>=`
https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/dao/dMute.sol#L99-L100

```solidity
    require(lock_info.amount >= 0 , "dMute::Redeem: INSUFFICIENT_AMOUNT");
    require(lock_info.tokens_minted >= 0 , "dMute::Redeem: INSUFFICIENT_MINT_AMOUNT");
```

Currently, the above conditions will be true with the empty info. There is no serious impact but it will work without reverting for the wrong indices. It should use `>` instead.