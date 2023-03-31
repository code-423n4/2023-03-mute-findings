QA1. payout() lacks the emit of event of ``emit Payout(msg.sender, reward, remainder);``, which is emitted at L245 of the ``withdraw()`` function. Without this event, the reward payout cannot be consistently monitored. 

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L287-L295](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/amplifier/MuteAmplifier.sol#L287-L295)


The fix is as follows:

```diff
if (reward > 0) {
            uint256 week_time = 1 weeks;
            IDMute(dToken).LockTo(reward, week_time ,msg.sender);

            userClaimedRewards[msg.sender] = userClaimedRewards[msg.sender].add(
                reward
            );
            totalClaimedRewards = totalClaimedRewards.add(reward);
+           emit Payout(msg.sender, reward, remainder);    
        }
```

QA2. It is important to add ``require(_maxPrice >= _startPrice, "starting price < min") `` to both ``setMaxPrice()`` and ``setStartPrice()``. Otherwise, if the constraint is violated, the protocol will break. For example, ``bondPrice()`` will always fail. 


[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L99-L113](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L99-L113)

QA3. There is no sanity check for the input of ``setBondTimeLock()``. With inappropriate input, the protocol will break. For example, the ``timeToTokens()`` restricts the _lock_time to be between 1 week and 52 weeks. As a result, when ``_lock`` is set to be out of this range, the whole protocol will break.

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L139-L143](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/bonds/MuteBond.sol#L139-L143)

Mitigation: to be consistent, introduce the same range check for ``setBondTimeLock()``:
```diff

function setBondTimeLock(uint _lock) external {
        require(msg.sender == customTreasury.owner());
+       require(_lock >= 1 weeks && _lock <= 52 weeks) revert _lockOutOfRange(); 
        bond_time_lock = _lock;
        emit BondLockTimeChanged(_lock);
    }
```

QA3. There is a division-before-multiplication precision loss issue with ``timeToTokens()``

[https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/dao/dMute.sol#L49-L62](https://github.com/code-423n4/2023-03-mute/blob/4d8b13add2907b17ac14627cfa04e0c3cc9a2bed/contracts/dao/dMute.sol#L49-L62)

The problem lies in the following line:

```javascript
        uint256 base_tokens = _amount.mul(_lock_time.mul(10**18).div(max_lock)).div(10**18);
```

It has a division first: 

```javascript
_lock_time.mul(10**18).div(max_lock)
```
which has a precision loss up to ``max_lock = 3.145e7``. 

Then multiplication here:
```javascipt
_amount.mul(_lock_time.mul(10**18).div(max_lock))
```

Mitigation: use multiplication before division as follows:
```diff
- uint256 base_tokens = _amount.mul(_lock_time.mul(10**18).div(max_lock)).div(10**18);
+ uint256 base_tokens = _amount.mul(_lock_time).div(max_lock);
```