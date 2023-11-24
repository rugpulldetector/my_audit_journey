## Summary

`AuctionDemo.participateToAuction()` can be sandwitched by malicious actor to block any bid from being made.

## Severity

Medium

## Proof of Concept
Sandwiching bundle can be comprised of following txs.

1) Frontrunning tx with participateToAuction()
2) Victim's participateToAuction()
3) Backrunning tx with cancelBid()

https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/AuctionDemo.sol#L57-L61
```solidity
    function participateToAuction(uint256 _tokenid) public payable {
        require(msg.value > returnHighestBid(_tokenid) && block.timestamp <= minter.getAuctionEndTime(_tokenid) && minter.getAuctionStatus(_tokenid) == true);
        auctionInfoStru memory newBid = auctionInfoStru(msg.sender, msg.value, true);
        auctionInfoData[_tokenid].push(newBid);
    }
```

## Impact
Denial of Service
As there's no minimum bidding amount ratio or time limit as compared to previous bid, a malicious actor can block any bid to be made easily.

## Tools Used
Manual Review

## Recommendation
Thers should be minimum bidding amount ratio or time limit as compared to previous bid.