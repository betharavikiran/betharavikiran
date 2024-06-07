### C4 Renzo finding: RestakeManager::calculateTVLs() can be incorrect due to admin of collateralTokens list ###

C4 finding submitted:
     risk = 2 (Med Risk)

     # Lines of code

https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L244-L263
https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L274-L358


# Vulnerability details

## Impact
The TVL calculation could result in incorrect TVL if the Admin removes the collateral from CollateralList using removeCollateralToken() function.

Since the calculateTVLs() function loops through the CollateralList to arrive at the TVL, if a token is removed from the list with out checking for balance in the strategies, then the TVL computed will be incorrect.

## Proof of Concept
As we can see from the below code snippet, to compute TVL, for each of the tokens in the list, the balance with each strategy is checked and accounted for TVL. If token is remove from this list with out checking for any balance in strategies, then potentially, the TVL value is under accounted and hence incorrect.

```
   function calculateTVLs() public view returns (uint256[][] memory, uint256[] memory, uint256) {
     ...
===>       // Iterate through the tokens and get the value of each
           uint256 tokenLength = collateralTokens.length;
           for (uint256 j = 0; j < tokenLength; ) {
               // Get the value of this token

               uint256 operatorBalance = operatorDelegators[i].getTokenBalanceFromStrategy(
                   collateralTokens[j]
               );

               // Set the value in the array for this OD
               operatorValues[j] = renzoOracle.lookupTokenValue(
                   collateralTokens[j],
                   operatorBalance
               );
     ...
   }
```

## Tools Used
Manual

## Recommended Mitigation Steps
Before removing the token from the CollateralList, the function should check against all strategies to ensure there is no balance invested in any of the strategies. This check will ensure that token being removed does not have any funds locked in strategies.


## Assessed type

Other
