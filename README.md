# Compound V2 Notebook

This is my personal notes about some important points in Compound V2 and all of these are based on my personal understanding. I write down these notes to make me able to refresh my memory anytime in the future when needed.

You can find Compound V2 code here: [Compound V2](https://github.com/compound-finance/compound-protocol/tree/master).

## CToken (Ceth and CErc20)

- `CToken.sol` is the bedstone of all of CTokens. It defines the core logic in CToken (including basic ERC20 functions, and CToken specific functions related to mint, redeem, borrow and repay as well as some admin functions).

- `CEther.sol` inherits from CToken and defines the public interfaces for end users, enabling them to mint, redeem, borrow and repay. It also define the method `doTransferIn()` and `doTransferOut()` to facilitate the ETH transfers.

- All other CErc20s use a proxy-style. Like `CEther.sol`, `CErc20.sol` defines the public user interfaces and `doTransferIn()`, `doTransferOut()`. `CErc20Delegate.sol` inherits from the `CErc20.sol` and intents to be the implementation. `CErc20Delegator.sol` serves as the proxy contract and it will delgate all the call to implementation contract.

- Rates in Compound V2 usually represented as mantissa, meaning the number is scaled by `1e18`.

- Among these basic ERC20 functions, noted when making a CToken transfer, `transferTokens()` will check with Comptoller to see if the CToken transfer is allowed in case a borrower be under-collateralized.

- Exchange rate for CToken is calculated as:
  ExchangeRate = (Cash + Borrow - Reserve)/CToken Supply.
  Cash is the underlying token hold in contract, borrow is the outstanding borrow amount, reserve is part of cash which belongs to Compound protocol.

- To calculate each one's borrow amount, CToken implements a struct.

  ```
  struct BorrowSnapshot {
      uint principal;
      uint interestIndex;
  }
  ```

  Principal is the borrowers borrow amount accured with interest and interestIndex is the borrowIndex as of the most recent balance-changing action. There is also a global borrowIndex. For example, in block 10 a borrower has principal of 1000 and interestIndex of 1.02, in block 20 the global borrowIndex is 1.04, then the borrower's debt in block 20 will be calculated as 1000 \* 1.04 / 1.02.

* `accrueInterest()` applies accrued interest to total borrows and reserves. It will update the global borrowIndex, accrualBlockNumber, totalBorrows and totalReserves. This function will be called anytime a user interact with Compound (mint, redeem, borrow, repay and liquidate).

* For each user interactions with protocol (mint, redeem, borrow, repay and liquidate, seize included in liquidate), the contract will check with Comptroller to see if this operation is allowed. For liquidation and seize there are some more extra checks in `CToken.sol` since they're more complicated.

## Comptroller

- Compound V2 Comptroller also implements proxy contract. `Unitroller.sol` is the proxy contract and it will delegatecall `Comptroller.sol` or `ComptrollerG7.sol`, which are the implemented logic contract.

- User must enter a market (CToken) in order to borrow assets. If a user wants his/her CToken counted as collateral he/she also must enter that specific market.

- When exiting a market, Comptroller will make sure the user has no debt on that market, also when the liqudity from that exited market subtracted the user account is not underwater.

- When checking if mint is allowed, Comptroller will check if the CToken given is listed.

- When checking if redeem is allowed, Comptroller will check if the CToken given is listed, if the user is in the market, and make sure the hypothetical liquidity after redeeming will not generate a positive shortfall, meaning the account is underwater.

- When checking if borrow is allowed, Comptroller will check borrow is not paused, CToken is listed, borrower is in the market, borrowCap won't exceed and the hypothetical liquidity after borrowing won't generate a positive shortfall.

- When checking if repay is allowed, Comptroller will only check if the CToken given is listed.

- When checking if liquidate is allowed, Comptroller will check the liquidated token and borrower collateral token are listed, then make sure the account to be liquidated is indeed underwater, and the repayAmount by liquidator doesn't exceed the limit determined by `CloseFactor`.

- When checking if seize is allowed, Comptroller will check seize is not paused, collateral and borrowed tokens are listed, Comptrolled used by collateral and borrowed tokens are same.

- When checking if transfer is allowed, Comptroller will check transfer is not paused, and then apply the same check as redeem with transfer amount.

- `getHypotheticalAccountLiquidityInternal()` will calculate the liquidity of a user with assumed redeem and/or borrow tokens amount. It will iterate through the markets user entered, obtained the collateral price from oracle and calculate user's collateral value and borrowed value. It will return errorCode, user's liquidity(the value a user can borrow) and user's shortfall(positive meaning underwater).

- `liquidateCalculateSeizeTokens()` will calculate seized tokens during liquidation. It computes the USD value of repayAmount considering incentive and converts the amount to seized CTokens. During liquidation, the incentive a liquidator obtained is in CTokens.

- `updateCompSupplyIndex()` will update the Comp amount received per CToken. It computes the total Comp rewards by multiplying passed blocks and CompSpeed, then divided it by total CToken supply to obtain the result. Accrual index and block are then updated.

- `updateCompBorrowIndex()` will update the Comp amount received per CToken. It computes the total Comp rewards by multiplying passed blocks and CompSpeed, then divided it by total borrowAmount to obtain the result. Accrual index and block are then updated.

- Calculation of a supplier Comp rewards uses an index similar to borrow index. There is a global index and a local supplier index, rewards are computed by multiplying the difference in index and suppliers balance.

- Calculation of a borrower Comp rewards uses an index similar to borrow index. There is a global index and a local borrower index, rewards are computed by multiplying the difference in index and borrower's borrow amount.

## Price Oracle

- Compound Price Oracle is not in its github repo. You can find the contract here: [Compound V2 Price Oracle](https://etherscan.deth.net/address/0x50ce56A3239671Ab62f185704Caedf626352741e#code).

- `UniswapConfig.sol` includes configuration for each CToken the protocol will use. The maximum number of tokens is hardcoded as 29. Each CToken has a configuration struct like below.

  ```
  struct TokenConfig {
          // The address of the Compound Token
          address cToken;
          // The address of the underlying market token. For this `LINK` market configuration, this would be the address of the `LINK` token.
          address underlying;
          // The bytes32 hash of the underlying symbol.
          bytes32 symbolHash;
          // The number of smallest units of measurement in a single whole unit.
          uint256 baseUnit;
          // Where price is coming from.  Refer to README for more information
          PriceSource priceSource;
          // The fixed price multiple of either ETH or USD, depending on the `priceSource`. If `priceSource` is `reporter`, this is unused.
          uint256 fixedPrice;
          // The address of the pool being used as the anchor for this market.
          address uniswapMarket;
          // The address of the `ValidatorProxy` acting as the reporter
          address reporter;
          // Prices reported by a `ValidatorProxy` must be transformed to 6 decimals for the UAV.  This is the multiplier to convert the reported price to 6dp
          uint256 reporterMultiplier;
          // True if the pair on Uniswap is defined as ETH / X
          bool isUniswapReversed;
      }
  ```

  Noted the value of some stablecoins are hardcoded to equal to 1 USD. For example, USDC Config will have TokenConfig.fixedPrice as 1000000, since price has 6 decimals, the value is 1 USD per USDC.

- Each token whose price is not hardcoded relies on external reporter to report the price. Reporter is configured in Token.Config struct. Only authorized reporter can call `validate()` in `UniswapAnchoredView.sol` to update the price.

- When a reporter update token price through `validate()` function in `UniswapAnchoredView.sol`, the function will first check if a failedoverActive is true, if it is, the price will be updated by Uniswap V3 TWAP and reporter's price is discarded. If failedoverActive is false, the function will compare reporter's price with Uniswap V3 TWAP, if the difference is within certain range price is updated, otherwise no update.

- Only `owner` can update activate Failover Price Oracle in `UniswapAnchoredView.sol`. The contract won't automatically change to failover Price Oracle even if the reporters continiously report deviated price. If the owner set `failoverActive` to true, then anyone can call `pokeFailedOverPrice()` to update the token price.

- There is a `prices` map in `UniswapAnchoredView.sol`. The prices stored there have 6 decimals format. For example, DAI price is 1 USD, the price in that map will return 1000000.

- `getUnderlyingPrice()` will be called by Compound V2 Comptroller. It will return the price in the format ${raw price} \* 1e36 / baseUnit. For example, if COMP price is 4 USD and has 18 decimals, the function will return 4 \* 1e36 / 1e18 = 4 \* 1e18.

## Governance

- `Comp.sol` is the contract for COMP token. COMP is the governance token for Compound V2 Token. User's vote power is defined by COMP token amount. Users can delegate their vote to any address (self included, but address(0) not allowed). This can be done by calling `delegate()` function or calling `delegateBySig()` with a valid offchain signature.

- An account can have many checkpoints. Anytime this account transfer COMP token or receive COMP, or its vote power updated a new checkpoint and corresponding vote power will be recorded in a map. Previous vote power balance can also be quiried from this map.

- `getPriorVotes()` in `Comp.sol` can get a user vote power associated with any previous block number. It performs a binary search to get the vote power from the closest checkpoint before the given block. For example, a user has 2 checkpoints, No. 1 has 1000 vote power from block 10, No. 2 has 2000 vote power from block 20. When given block 18, the function will return No. 1 checkpoint result, which is 1000 vote power.

- GovernorBravo in Compound V2 also implements proxy-style. `GovernorBravoDelegator.sol` is the proxy contract and will delegatecall `GovernorBravoDelegateG1/G2.sol` as the implementation contract.

- To propose a proposal, an account must have a COMP amount larger than the proposal threshold counting from one block before the current block, this makes it impossible to use flashloan to make a proposal.

- In order to make the proposal succeed, the proposal must have forVotes > againstVotes and forVotes amount must larger than quorumVotes, which is a constant of 400000 COMP token.

- After enough time passed, anyone can call `queue()` and `execute()` to make the Timelock to queue and execute corresponding proposals. Timelock usually is the admin for Compound V2 contracts like Comptroller, CToken contracts, etc. Timelock is only controlled by the GovernorBravo contract.
