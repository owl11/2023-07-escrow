# CodeHawks Escrow Contract - Competition Details - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. multiple dynamic values in abi.encodePacked](#M-01)
    - ### [M-02. safeTransfer can be used to DOS](#M-02)
- ## Low Risk Findings
    - ### [L-01. add neccessary approval call prior to deployment](#L-01)
    - ### [L-02. limit time before escrow is usable](#L-02)
    - ### [L-03. Arbiter can be buyer or seller](#L-03)
    - ### [L-04. salt front-run mitigation](#L-04)
- ## Gas Optimizations / Informationals
    - ### [G-01. IERC20 present in SafeERC20](#G-01)
    - ### [G-02. duplicate code can be a function](#G-02)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Cyfrin

### Dates: Jul 24th, 2023 - Aug 5th, 2023

[See more contest details here](https://www.codehawks.com/contests/cljyfxlc40003jq082s0wemya)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 2
   - Low: 4
  - Gas/Info: 2


		
# Medium Risk Findings

## <a id='M-01'></a>M-01. multiple dynamic values in abi.encodePacked            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-escrow/blob/fa9f20a273f1e004c1b11985ef0518b2b8ae1b10/src/EscrowFactory.sol#L56

https://github.com/Cyfrin/2023-07-escrow/blob/fa9f20a273f1e004c1b11985ef0518b2b8ae1b10/src/EscrowFactory.sol#L71

https://github.com/crytic/slither/wiki/Detector-Documentation#abi-encodepacked-collision

## Summary
encoding multiple dynamic values using abi.encodepacked raises alarms in slither, as well as being highly unrecommended in the solidity docs 
## Vulnerability Details
could result in hash collisions 
## Impact
--
## Tools Used
slither static analyzer
## Recommendations
make the function internal instead of public, as well as using `abi.encode` instead of `abi.encodepacked` for multiple dynamic variables as recommended in the official solidity documentation 
## <a id='M-02'></a>M-02. safeTransfer can be used to DOS            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-escrow/blob/fa9f20a273f1e004c1b11985ef0518b2b8ae1b10/src/Escrow.sol#L120C1-L120C1

## Summary
Both `resolveDispute` and `confirmReceipt` depend on safeTransfer to succeed, since there are many erc20 tokens variants as well as other variants that can make these functions misbehave, or at best be DOS'd
## Vulnerability Details
either state transition functions can constantly revert depending on the safeTransfer, potential Dos, or simple ERC20 variants that behave unexpectedly.
## Impact
can render a smart contract unusable, or at worst, a malicious token owner can blacklist the seller and forbid the arbiter from resolving a dispute. Rendering the contract to favor their outcome.
## Tools Used
Manual review
## Recommendations
consider splitting critical state transitions into one function, that updates balance to each corresponding user with, and allowing each user to withdraw their tokens separately, in that case the enum `state.Completed` can be re-used so that it updates to completed once 3/3 withdrawals are confirmed

# Low Risk Findings

## <a id='L-01'></a>L-01. add neccessary approval call prior to deployment            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-escrow/blob/65a60eb0773803fa0be4ba72defaec7d8567bccc/src/EscrowFactory.sol#L20

## Summary
The contract assumes that the user has approved their tokens for the contract, while generally safe to assume, it is unsafe since some non-native defi users can easily assume approving the contract means sending the contract your tokens, like we've seen before with many examples.
## Vulnerability Details
could lead to stuck tokens in escrow factory contract
## Impact
loss of funds
## Tools Used
manual review
## Recommendations
add an approve call at the beginning of the contract, approving the amount `price` from the callers balance
## <a id='L-02'></a>L-02. limit time before escrow is usable            



## Summary
Initially, both users shouldn't do anything before the audit is complete. depending on the size of the audit/the codebase or any other external factors, it is safe to assume no one should call the contract within the an arbitrary time that the users can agree on
## Vulnerability Details
N/A
## Impact
limit rushing into conclusions immediately after making the contract
## Tools Used
manual review
## Recommendations
either limit it by blocks, or use solady lib for unix timestamps
## <a id='L-03'></a>L-03. Arbiter can be buyer or seller            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-escrow/blob/fa9f20a273f1e004c1b11985ef0518b2b8ae1b10/src/Escrow.sol#L103

https://github.com/Cyfrin/2023-07-escrow/blob/fa9f20a273f1e004c1b11985ef0518b2b8ae1b10/src/Escrow.sol#L43

## Summary
The arbiter arbitrary role can be set to any adderss, while that's useful, it can also be useful to limit both users from accidentally being the arbiter as well
## Vulnerability Details
If either the buyer inputs all the details and accidentally inputs either their address or the sellers, the risk of collusion becomes higher
## Impact
If the seller, they can simply not provide their services, turn malicous and `intiateDispute` -> `resolveDispute` with 0 going to the `buyerAward`

If it's the buyer, they can pretend that the escrow is going as planned, turn malicious and once they have gotten their services, `intiateDispute` -> `resolveDispute`, setting the `buyerAward`, equal to the `balanceOf(address(this))`- `arbiterFee`
## Tools Used
manual review
## Recommendations
while a malicious user can still use another address, it's better to limit them from at least inputting their own address, gaining an apprehend and perhaps malicious intent
## <a id='L-04'></a>L-04. salt front-run mitigation            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-escrow/blob/fa9f20a273f1e004c1b11985ef0518b2b8ae1b10/src/EscrowFactory.sol#L26

## Summary
Current implementation allows newEscrow deployer to arbitrarly set salt, which is a known vulnerbility
## Vulnerability Details
arbitrary salt exposes `newEscrow` deployment proccess to front-run attacks, where malicous user can bid more gas to deploy the same contract before the actual user, potentially taking control of the escrow contract as well as the tokens
## Tools Used
manual review
## Recommendations
Using a random salt for each escrow could prove very beneficial here, utilizing `block.prevrandao`to generate a completely random new salt for each new escrow 
more precisely `bytes32(uint256(keccak256(abi.encodePacked(block.prevrandao))))`
which would leave the salt completely out of any user's calldata as well remedy the potential vulnerability 

# Gas Optimizations / Informationals

## <a id='G/I-01'></a>G/I-01. IERC20 present in SafeERC20            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-escrow/blob/65a60eb0773803fa0be4ba72defaec7d8567bccc/src/Escrow.sol#L7

https://github.com/Cyfrin/2023-07-escrow/blob/65a60eb0773803fa0be4ba72defaec7d8567bccc/src/Escrow.sol#L8

## Summary
additional import statement can be removed and instead import directly from SafeERC20 since it is present there
## Vulnerability Details
N/A
## Impact
N/A
## Tools Used
manual review
## Recommendations
remove IERC20 import statement and instead import it from SafeERC20
`import {SafeERC20, IERC20}`
## <a id='G/I-02'></a>G/I-02. duplicate code can be a function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-escrow/blob/fa9f20a273f1e004c1b11985ef0518b2b8ae1b10/src/Escrow.sol#L98

https://github.com/Cyfrin/2023-07-escrow/blob/fa9f20a273f1e004c1b11985ef0518b2b8ae1b10/src/Escrow.sol#L110

https://github.com/Cyfrin/2023-07-escrow/blob/fa9f20a273f1e004c1b11985ef0518b2b8ae1b10/src/Escrow.sol#L125

https://github.com/Cyfrin/2023-07-escrow/blob/65a60eb0773803fa0be4ba72defaec7d8567bccc/src/Escrow.sol#L44C1-L44C1

## Summary
both `resolveDispute` and `confirmReceipt` as well as the constrcturor utilize similar code `i_tokenContract.balanceOf(address(this))`, make it a function to reduce Repetition
## Vulnerability Details
--
## Impact
improve readability 
## Tools Used
manual review
## Recommendations
add function `getContractBalance` that would be used in place of the duplicate code
