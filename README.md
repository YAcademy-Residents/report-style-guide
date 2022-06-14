# Report Style Guide

## Code notation

- Contract names should not be written in code notation and should include the extension (e.g., .sol)
- Functions and variables should both be written with code notation
- Functions should be written as `functionabc()` and not `functionabc`
- When writing a code block, specify the solidity language (or the relevant language) to enable [code syntax highlighting](https://www.markdownguide.org/extended-syntax#syntax-highlighting)

## Report sections

### Layout
- Each finding should include a number/severity/title, a brief description, a proof of concept, an impact, a recommendation, and a developer response. The description "Developer response" is preferred over "Comments" to make clear who the author is.

### Number/Title
- Title should be: \[Number of finding\]. \[Severity\] - \[Title\] (\[Resident\])
- The first letter of the title should be capitalized and the remainder lower case

### Technical Details
- Links to specific lines of code should reference a specific commit hash on the main repository or should reference the yAcademy fork of the repository, where the repo remains frozen on the commit hash for the repo
  - For example, use GitHub permalinks with the commit hash that the repo is focused on
- Code blocks may be used instead of links, but should indicate the file and line number for the code block
- Breaks in the code should be indicated by "..." or "\*\*\*" with new lines before and after.
- Lines of code should be referred to as L101-L201 and not LL101-201. 

### Impact
- The "Impact" section of the report should have a format following these examples
```
Low. The rationale goes here, ending with a period.

High. This explanation has two sentences. They both end with periods.
```

### General
- Sentences with periods are preferred for most sections other than title.
- Cross references to other findings should be "high finding 1" or "H1".

## Examples:

### 1. High - The swap and stake mechanisms in OpenMevZapper.sol leave funds in the contract (Jackson)

Half of the input amount in both `swapAndStakeLiquidity()` and `swapETHAndStakeLiquidity()` is used as the `swapAmountIn` when atomically swapping and staking.  However, this leaves funds in the contract due to the reserve asset ratio change post-swap.  See ["Optimal One-sided Supply to Uniswap"](https://blog.alphaventuredao.io/onesideduniswap/) for more information.

#### Technical Details

Both [`swapAndStakeLiquidity()`](https://github.com/manifoldfinance/OpenMevRouter/blob/8648277c0a89d0091f959948682543bdcf0c280b/contracts/OpenMevZapper.sol#L126-L159) and [`swapETHAndStakeLiquidity()`](https://github.com/manifoldfinance/OpenMevRouter/blob/8648277c0a89d0091f959948682543bdcf0c280b/contracts/OpenMevZapper.sol#L165-L195) take the input tokens or ETH sent by a user, divide it by 2, swap it into the B token, and stake these tokens as a pair.  However, this approach leaves some of the B token in the contract due to the reserve asset ratio change before and after the swap.

#### Impact

High.  The funds are not returned to the user, and will likely be swept by Sushi governance during a call to `harvest()`.

#### Recommendation

Use the formula found in ["Optimal One-sided Supply to Uniswap"](https://blog.alphaventuredao.io/onesideduniswap/) for the `swapAmountIn`, rather than ` / 2`.


```solidity
sqrt(
    reserveIn.mul(userIn.mul(3988000) + reserveIn.mul(3988009)))
        .sub(reserveIn.mul(1997)) / 1994;
```

#### Developer Response (Sandy)

Fixed [here](https://github.com/manifoldfinance/OpenMevRouter/commit/d95ec8543337787dcb3f7499f6f4ec6d69eb7b52) and [here](https://github.com/manifoldfinance/OpenMevRouter/commit/958a70d6034db745555cf8b9effcb97bd2c59e20).

### 1. Low - No check for `_amount > type(uint128).max` (engn33r)

IncurDebt.sol has functions that receive uint256 values and typecasts them to uint128 for some operations but not others. 

#### Technical Details

In L249-L251 and L389-L390, \_amount is typecast to uint128 for updating the borrower's collateral but not for transferring funds.

```solidity
        borrowers[msg.sender].collateralInGOHM += uint128(_amount);
        IERC20(gOHM).safeTransferFrom(msg.sender, address(this), _amount);

        ***

        borrower.collateralInGOHM -= uint128(_amount);
        IERC20(gOHM).transfer(msg.sender, _amount);
```

#### Impact

Low. It is very unlikely a user will have more than type(uint128).max gOHM, but loss of funds would be possible if that happened. Withdraw has additional checks that should prevent users from transferring out more than their collateral.

#### Recommendation

Typecast \_amount to uint128 for all operations or revert when `_amount > type(uint128).max`.

#### Developer Response

Fixed.