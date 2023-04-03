## Remove `SafeMath` library
Gas saved: 2.1K for each tx that uses `SafeMath` + 100 units per call.

It seems like safe math doesn't serve any purpose here and it can simply be replaced with normal math operands. The usage of library is pretty expensive since it makes a delegate call to an external address.

## Make `MuteAmplifier` variables immutable when possible
Gas saved: up to 20K per tx

The following variables can be immutable:
* `lpToken`
* `muteToken`
* `dToken`
* `_stakeDivisor`
* `management_fee`
* `treasury`

### Refactor `initializeDeposit()` into the constructor to make more variables immutable
Gas saved: ~7K per tx
The logic of `initializeDeposit()` cen be integrated into the constructor to make the following variables immutable:
* `totalRewards`
* `startTime`
* `endTime`

The only issue is sending tokens before the contract creation.
This can be solved in one of the following ways:
* Pre-calculate the address of the contract and approve the tokens to the address before creating it
* Add a deployer contract that will create the `MuteAmplifier` contract. The deployer contract will hold the tokens and will contain a callback that when called would send the tokens to the sender. The callback will revert if it'll be called at any time except during deployment (it'll use a storage variable to indicate if we're during a deployment or not)

## Use `lock_index` to look for empty elements
Gas saved: 2.2K * (amount of non-empty elements)

At [dMute.redeemTo()](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/dao/dMute.sol#L112-L120) the function iterates through the entire array to find empty elements, this means the entire array is being read in search for the empty elements.
Instead, we can find the empty elements using the `lock_index` and iterate through them only.


## At `MuteAmplifier.update()` skip updating if done in the same block
Gas saved: A few thousands

When `update()` is being called a second time in the same block both the `_totalWeight` and `_mostRecentValueCalcTime` don't really change. But the logic to update them still run and uses a few thousands of gas units.
Skipping that when `sinceLastCalc` is zero can save that gas.

Additionally, the remaining logic regarding fees might be skipped too (not sure about that).

Anyways, if the `fee0` or `fee1` are zero, some of the logic can be skipped too and save gas.

## `MuteBond.deposit()` makes an unnecessary call to `payoutFor()` during max buy
Gas saved: a few hundreds probably

In the following code the `payoutFor()` is unnecessary when `max_buy` is true and therefore should be inside the else block.

```solidity
        uint payout = payoutFor( value );
        if(max_buy == true){
          value = maxPurchaseAmount();
          payout = maxDeposit();
        } else {
```

## Storage variable caching
Gas saved: 100 per cache

The following variables can be cached into memory instead of being read (or written) twice:


Here 2 storage variables are being read twice:
[amplifier/MuteAmplifier.sol#L370-L373](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L370-L373)
```
        // current rewards based on multiplier
        reward = lpTokenOut.mul(_totalWeight.sub(_userWeighted[account])).div(calculateMultiplier(account, true));
        // max possible rewards
        remainder = lpTokenOut.mul(_totalWeight.sub(_userWeighted[account])).div(10**18);
```
`_userStakedBlock` is being read twice here:
[amplifier/MuteAmplifier.sol#L480-L481](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L480-L481)
```solidity
        uint256 staked_block =  _userStakedBlock[account] == block.number ? _userStakedBlock[account] - 1 : _userStakedBlock[account];

```

`epochStart` is being read and written a few times:
[bonds/MuteBond.sol#L185-L197](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L185-L197)
```solidity
        // adjust price by a ~5% premium of delta
        uint timeElapsed = block.timestamp - epochStart;
        epochStart = epochStart.add(timeElapsed.mul(5).div(100));
        // safety check
        if(epochStart >= block.timestamp)
          epochStart = block.timestamp;

        // exhausted this bond, issue new one
        if(terms[epoch].payoutTotal == maxPayout){
            terms.push(BondTerms(0,0,0));
            epochStart = block.timestamp;
            epoch++;
        }
```

## `_applyReward()` doing `sstore`s twice when called from `stake()`
Gas saved: ~400

When `_applyReward()` is being called from `stake()` [the following lines](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L383-L387) are writing to variables that are then written again at `_stake()`:
```solidity
        _totalStakeLpToken = _totalStakeLpToken.sub(lpTokenOut);

        _userStakedLpToken[account] = 0;

        _userAccumulated[account] = 0;
```

Rewriting `_applyReward()` so that it won't do those changes (e.g. pass a parameter to indicate whether it's being called from `stake()` or not) and instead doing them at `_stake()` can save gas.

## Replace string errors with custom errors
Gas saved per instance: Probably a few thousands for deployment + a few hundreds when the function reverts

Custom errors are cheaper both for deployment and reverts, since strings take up lots of bytes code while custom reverts just a single byte.


## `MuteBond`'s `epoch()` and `currentEpoch()` are the same
Gas saved: a few units per tx

`epoch()` and `currentEpoch()` are basically the same, each public/external function adds a conditional check for each tx (since the contract compares the call data against the function has to find the right one). Removing this duplication can save a few gas units.

## Unnecessary multiplication and division by 1e18
Gas saved: ~20 units (200+ units when using `SafeMath`)

In [the following line](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/dao/dMute.sol#L57) the multiplication and division by 1e18 is unnecessary.
Instead, just make sure all multiplications are done before division, that will prevent rounding in the same manner.

```diff
-        uint256 base_tokens = _amount.mul(_lock_time.mul(10**18).div(max_lock)).div(10**18);
+        uint256 base_tokens = _amount.mul(_lock_time).div(max_lock);
```

