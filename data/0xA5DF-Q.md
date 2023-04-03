## Bonds might have a very small amount left to end the current era
There might be a case where a user bought a bond in an amount that leaves the contract a very low (up to 1 unit) amount away from ending an era.
In that case it's still possible to end the era using `max_buy`, but it might not be worth the gas fee to purchase such a low amount.

As mitigation - end the era when the total payout of the current era is close enough to the max payout, even if it's not the exact amount.


## `bondPrice()` naming might be confusing
The name 'bond price' is a bit confusing since the user that calls `deposit()` is considered to be the bond buyer (according to the docs), meaning a higher price is actually in favor of the buyer (they get a higher payout).
Therefore, it might be better to change the name to a more clear name (e.g. 'LP price' since it indicates the payout the user gets per LP).