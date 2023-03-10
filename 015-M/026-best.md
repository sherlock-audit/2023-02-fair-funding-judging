MyFDsYours

high

# Returned transfer function is not checked occured a potential loss of funds for last_bidder

## Summary
Returned transfer function is not checked occured a potential loss of funds for last_bidder 

## Vulnerability Detail

In the AuctionHouse.vy contract if a user call bid function to create a new bid with greater amount, the last_bidder is refunded  but the boolean value returned by the transfer function is not checked assuming it's always returned true.

## Impact
If transfer function failed or not response properly *last_bidder* will loss his last deposit funds and he will not be able to call any function to get refunded. 

## Code Snippet

[AuctionHouse.vy#L164-L167](https://github.com/sherlock-audit/2023-02-fair-funding/blob/main/fair-funding/contracts/AuctionHouse.vy#L164-L167)
```vyper
# refund last bidder
if last_bidder != empty(address) and last_bid > 0:
    ERC20(WETH).transfer(last_bidder, last_bid)
    log BidRefunded(_token_id, last_bidder, last_bid)

```

## Tool used

Manual Review

## Recommendation

You can add  an assert to the transfer function like this : 
```diff
diff --git a/fair-funding/contracts/AuctionHouse.vy b/fair-funding/contracts/AuctionHouse.vy                                              
index fe470ee..082a988 100644
--- a/fair-funding/contracts/AuctionHouse.vy
+++ b/fair-funding/contracts/AuctionHouse.vy
@@ -163,7 +163,7 @@ def bid(_token_id: uint256, _amount: uint256):
 
     # refund last bidder
     if last_bidder != empty(address) and last_bid > 0:
-        ERC20(WETH).transfer(last_bidder, last_bid)
+        assert ERC20(WETH).transfer(last_bidder, last_bid), "refunded transfer failed"
         log BidRefunded(_token_id, last_bidder, last_bid)
 
     # collect bid from current bidder
```

But instead of transferring funds to last_bidder directly in bid function, I highly recommend you to : 
- create a mapping that tracks balances of refunded addresses. 
- update refundable balance of the last_bidder by adding last_bid (in the bid function)
- create an other function (eg. name it refund_bidder) acting like a withdraw function that can permit to last_bidder's to get refunded (for the last bid or for several).
