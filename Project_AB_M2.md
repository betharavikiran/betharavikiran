### ArrakisMath inconsistencies for different tokens decimals ###

### Description ###
The library code assumes that token will have a max of 18 decimal positions which can cause the integer overflow.
But, there are tokens that have more than 18 decimal positions.

```
 function _normalizeTokenDecimals(uint256 amount, uint256 decimals) internal pure returns (uint256) {
        return amount * 10 ** (18 - decimals);
    }
and

if (token1 < token0) {
            price = (2 ** 192) / ((sqrtPriceX96) ** 2 / uint256(10 ** (token0Decimals + 18 - token1Decimals)));
        } else {
            price = ((sqrtPriceX96) ** 2) / ((2 ** 192) / uint256(10 ** (token0Decimals + 18 - token1Decimals)));
        }

```
### Recommendation ###
At the entry point of each applicable library function, validate if the token has decimal positions to be less than or equal to 18 decimals.
For any tokens have larger than 18 decimals reverts with error stating the token is not supported.