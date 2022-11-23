# TestAuction audit report.

# Summary

This is the report from a security audit performed on [TestAuction](https://gist.github.com/yuriy77k/edf8b3bcddbc3d43967f5765edf4727e) by [Oleg](https://github.com/DedicatedDev).

The audit focused primarily on the security of funds and fault tolerance of the auction contract.

# In scope

1. [testAuctoin.sol](https://gist.github.com/yuriy77k/edf8b3bcddbc3d43967f5765edf4727ehttps://github.com/EthereumCommonwealth/ethereum-classic-multisig/blob/34f1b074d510a1e7d4fbeb0acf3708be02697cc8/contracts/MultisigWallet.sol)

# Findings

In total, **5 issues** were reported including:

- 4 high severity issues.

- 1 medium severity issues.

- 1 minor observation.

No critical security issues were found.

## Security issues

### 1. Reentrancy.

#### Severity: high

#### Description

RefundAll function(contracts/testAuction.sol#73) don't be protected from Reenrancy call.

`       - IERC20(USDT_TOKEN).transfer(addressList[i],user.usdtAmount) (contracts/testAuction.sol#73)
       External calls sending eth:
       - address(addressList[i]).transfer(user.bnbAmount) (contracts/testAuction.sol#69)
       State variables written after the call(s):
       - user.usdtAmount = 0 (contracts/testAuction.sol#74)`

`        External calls:
        - IERC20(offeringToken).transfer(msg.sender,amountOfTokens) (contracts/testAuction.sol#88)
        State variables written after the call(s):
        - user.tokenAmount = amountOfTokens (contracts/testAuction.sol#89)`

`        External calls:
        - IERC20(USDT_TOKEN).transferFrom(msg.sender,address(this),_amount) (contracts/testAuction.sol#120)
        State variables written after the call(s):
        - user.usdtAmount += _amount (contracts/testAuction.sol#122)`

Detailed description can be found [here](https://github.com/crytic/slither/wiki/Detector-Documentation#reentrancy-vulnerabilities).

#### Recommendation

Implement mutex or update a state variable before transaction.

### 2. Ignores return value ERC20 Token transfer function.

#### Severity: high

#### Description

`SetCaps(uint256, uin256)` function of `testAuction.sol` contract ignores transaction result.
ERC20 transferFrom function has return value which describe transaction succeed or not. So before update a `supplyToDistribute` variable, need to check this value first of all.

Same issue repeated at `claimTokens()`, `depositAuction(uint256,bool)`

Detailed description can be found [here](https://github.com/crytic/slither/wiki/Detector-Documentation#unchecked-transfer).

#### Recommendation

Please implement return value check logic with `require` or `if`

### 3. Empty fund deposit attack .

#### Severity: high

#### Description

When deposit to auction, attacker make a bid with the fake amount of deposit because contract did not check  `msg.value` is equal to `amount` or not at `depositAuction(uint256 _amount, bool _isUSDT) `. Attacker can make fake bid without any kind of deposit of BNB token. 

`        } else {
            totalBNB += _amount;
            user.bnbAmount += _amount;
        }
`


Detailed description can be found [here](https://github.com/EthereumCommonwealth/ethereum-classic-multisig/issues/3).

#### Recommendation

Please implement to check `msg.value` is same with `amount` or not.

### 4. Oracle Manipulation.

#### Severity: high

#### Description

`isSuccess` variable is determined from single source oracle.
If attacker manipulation oracle source by flash loan, even though the Auction did not reach to the real cap, the auction will be ended. 
Auction creator will lost his tokens by a silly price. 

Detailed description can be found [here](https://consensys.github.io/smart-contract-best-practices/attacks/oracle-manipulation/).

#### Recommendation

User multi source oracles to determine raised usd amount for the auction.

### 5. Contract Logic issue.

#### Severity: medium

#### Description

Heavy load loop.
`refundAll()` function has a heavy loop.
There is no limitation of `addressList` so if this variable has enough length, this transaction will fail always because of the block gas limit.
It will affect user's portfolio amount after cancel the Auction.
There is additional option to get refund except for this function. Thus client will lost his deposit.

This might be depends on specific business logic which contract used.
But after deploy contract, it might be used for the general purpose because there is no cap about the number of deposit to the Auction.

Detailed description can be found [here](https://github.com/kadenzipfel/smart-contract-attack-vectors/blob/master/attacks/dos-gas-limit.md).

#### Recommendation

It can be changed as a `claim` logic by users.


### 6. Zero Address check.

#### Severity: not security issue. 

#### Description

Implement validation about `offerToken` address. The Auction can be created with the wrong token address. 
It might trigger losses in bider portfolio.  

Detailed description can be found [here](https://github.com/crytic/slither/wiki/Detector-Documentation#missing-zero-address-validation).

#### Recommendation

Implement zero token address validation. Additionally might be better to add ERC165 interface to get interface id. 

# Specification

About special actions, it might be better to add events. 

# Conclusion

This contract include serious security vulnerabilities. Thus, it is not ready for production. 

It is highly recommended to complete a bug bounty before use.
