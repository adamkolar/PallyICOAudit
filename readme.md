# Pally ICO Audit
The purpose of the contracts is to distribute tokens of the Pally platform in accordence with the terms delared on the projects' website: https://www.pally.co/

# Audited contracts

 * [Crowdsale.sol](./contracts/Crowdsale.sol)
 * [PallyCoin.sol](./contracts/PallyCoin.sol)
 * [RefundVault.sol](./contracts/RefundVault.sol)

# General philosophical premises
## Trustlestness
The unique value proposition of smart contract applications is that compliance with pre-agreed terms is not a question of voluntary decision, but is ensured algorithmically through the logic of code. This protection should work both ways. Paricularly in the case of ICOs, not only should be sellers confident that buyers will comply with the terms of the sale, but also buyers should be able to verify that sellers will be forced to follow rules they set for themselves. This requires a bit of a philosophical adjustment on the part of developers, because they have to create code that is secure not only in relation to users, but also in relation to owners and system administrators. After all, the vision of blockchain enabled decentralized society is that the difference between authorities and ordinary participants will progressively diminish. A lot of the issues elaborated in this audit relate to this general principle.
## Security > convenience
In smart contract developement, anytime there's a decision between security and user convenience, the security should be preferred. And because almost all convenince related functionality increases complexity of the code and therefore makes it more difficult to ensure its security, it should probably be avoided alltogether. For example, if we want to limit maximum contribution to an ICO to some set amount, we should simply revert transaction when the limit is exceeded instead of returning the amount that exceeds the limit. That is not to say creating user friendly applications isn't important, but if at all possible, the user-friendliness should be implemented in the frontend part of the application, not in the critical smart contract code. Some issues of the audited code are related to this principle.


# Aims of the contracts and compliance

Below is the list of explicit and implicit aims of the audited code, checkmarks indicate if code ensures compliance with these aims.

- [ ] &nbsp;Distribute at most 10M PAL amongst presale participants
- [ ] &nbsp;Distribute at least 17M PAL amongst ICO participants at a predetermined price, otherwise allow participants to withdraw their ETH contributions 
- [X] &nbsp;Distribute at most 50M PAL amongst ICO participants
- [X] &nbsp;Limit sale period by start and end time (28 days in October and November 2017)
- [ ] &nbsp;Price of the tokens should change in a predictable way depending on the current amount of PAL sold:
   - Tier 1:     1PAL = $0.075 (12.5M tokens to be sold in this tier)
   - Tier 2:     1PAL = $0.080 (12.5M tokens to be sold in this tier)
   - Tier 3:     1PAL = $0.090 (12.5M tokens to be sold in this tier)
   - Tier 4:     1PAL = $0.100 (12.5M tokens to be sold in this tier)
- [ ] &nbsp;Prevent small amount of large buyers from claiming most of the supply for themselves, allow wide participation

# List of issues

## 1. Owner can change crowdsale address in the token contract at any time 

first pointed out by: https://www.reddit.com/r/ethdev/comments/75k46s/up_to_10_eth_for_finding_vulnerabilities_pallyco/do80pbi/

### PallyCoin.sol / lines 44-50
```javascript
   /// @notice Function to set the crowdsale smart contract's address only by the owner of this token
   /// @param _crowdsale The address that will be used
   function setCrowdsaleAddress(address _crowdsale) external onlyOwner whenNotPaused { 
      require(_crowdsale != address(0));
      crowdsale = _crowdsale;
   }
```
### Problem
The fact it's possible to change the address of the crowdsale contract in the PallyCoin contract at any time, allows owners to mint arbitrary amount (due to Issue #2) of additional tokens potentially even after the crowdsale ends, through calling distributeICOTokens, negating terms of the sale.

### Recommendations
either/or
- Allow setting crowdsale address only once
- Allow setting crowdsale address only before the ICO starts


## 2. Due to ordering error, presale token limit can be overstepped by arbitrary amount

first pointed out by: https://www.reddit.com/r/ethdev/comments/75k46s/up_to_10_eth_for_finding_vulnerabilities_pallyco/do7hjvo/

### PallyCoin.sol / lines 59 - 62
```javascript
      // Check that the limit of 10M presale tokens hasn't been met yet
      require(tokensDistributedPresale < 10e24);

      tokensDistributedPresale = tokensDistributedPresale.add(tokens);
```
### Problem
Newly minted tokens should be first added to the **tokensDistributedPresale** and then compared against the limit, othwerwise the limit can be overstepped by arbitrary amount. The same issue is in the **distributeICOTokens** function, but is only problematic if Issue #1 isn't solved, then the check becomes redundant because it already happens in the **Crowdsale** contract.

### Recommendations
Put the check against the limit after the new tokens are added to **tokensDistributedPresale**

## 3. Price of the token can be changed by owner at any time

### Crowdsale.sol / lines 254 - 271
```javascript
   /// @notice Set's the rate of tokens per ether for each tier. Use it after the
   /// smart contract is deployed to set the price according to the ether price
   /// at the start of the ICO
   /// @param tier1 The amount of tokens you get in the tier one
   /// @param tier2 The amount of tokens you get in the tier two
   /// @param tier3 The amount of tokens you get in the tier three
   /// @param tier4 The amount of tokens you get in the tier four
   function setTierRates(uint256 tier1, uint256 tier2, uint256 tier3, uint256 tier4)
      external onlyOwner whenNotPaused
   {
      require(tier1 > 0 && tier2 > 0 && tier3 > 0 && tier4 > 0);
      require(tier1 > tier2 && tier2 > tier3 && tier3 > tier4);

      rate = tier1;
      rateTier2 = tier2;
      rateTier3 = tier3;
      rateTier4 = tier4;
   }
```
### Problem
Owner can change the price of the token during crowdsale, this is especially problematic considering the success of the crowdsale is defined as surpassing set minimal amount of sold tokens, it is imaginable that owners faced with possible failure of the crowdsale could drastically lower the price of the token to meet the minimal goal, breaking the pre-agreed terms with buyers.
### Recommendations
either/or
- Allow setting token prices only once
- Allow setting token prices only before the ICO starts

## 4. Insuficient anti-whale measures

### Problem
Protection against big buyers is implemented by limiting maximal contribution to a set amount of ETH per address, while this forces whales to split their buys into multiple transactions, it is not very effective without implementing additional measures.
### Recommendations
either/or
- Limit maximal gas price for the buy transaction
- Use some kind of whitelist system to limit buys only to addresses of registered users (this is not feasible due to time constraints)

## 5. Multiple non-critical issues in calculateExcessTokens function
### Crowdsale.sol / lines 300 - 330
```javascript
   /// @notice Buys the tokens for the specified tier and for the next one
   /// @param amount The amount of ether paid to buy the tokens
   /// @param tokensThisTier The limit of tokens of that tier
   /// @param tierSelected The tier selected
   /// @param _rate The rate used for that `tierSelected`
   /// @return uint The total amount of tokens bought combining the tier prices
   function calculateExcessTokens(
      uint256 amount,
      uint256 tokensThisTier,
      uint256 tierSelected,
      uint256 _rate
   ) public constant returns(uint256 totalTokens) {
      require(amount > 0 && tokensThisTier > 0 && _rate > 0);
      require(tierSelected >= 1 && tierSelected <= 4);

      uint weiThisTier = tokensThisTier.sub(tokensRaised).div(_rate);
      uint weiNextTier = amount.sub(weiThisTier);
      uint tokensNextTier = 0;
      bool returnTokens = false;

      // If there's excessive wei for the last tier, refund those
      if(tierSelected != 4)
         tokensNextTier = calculateTokensTier(weiNextTier, tierSelected.add(1));
      else
         returnTokens = true;

      totalTokens = tokensThisTier.sub(tokensRaised).add(tokensNextTier);

      // Do the transfer at the end
      if(returnTokens) msg.sender.transfer(weiNextTier);
   }
```
### Problems
1) Function is defined as constant but contains msg.sender.transfer, which is not allowed, luckilly this is practicallye a non-issue because of the problem 2)
2) Function is never called from inside the contract with the tierSelected = 4, which means code path that begins on the line 324 is effectively dead and never used, it can be called with this argument from outside, but that call allways results in error due to the attempt to transfer inside a constant function
3) Comment on the line 328 betrays misunderstanding of best coding practices, it is true that external calls should be executed last in order to prevent re-entrancy attacks, but they should be at the end of the whole code execution, putting them at the end of a function that is called in the middle of another function negates the security practice.

### Recommendations
To my best knowledge, the code can be left as is, but I would recommend removing the transfer line because it seems to be useless and in future versions of solidity compilers might make the code impossible to compile.

## 6. In case buy exceeds one whole tier, it's theoretically possible to buy tokens from higher tier for lower tier price

### Crowdsale.sol / lines 300 - 350
```javascript
   /// @notice Buys the tokens for the specified tier and for the next one
   /// @param amount The amount of ether paid to buy the tokens
   /// @param tokensThisTier The limit of tokens of that tier
   /// @param tierSelected The tier selected
   /// @param _rate The rate used for that `tierSelected`
   /// @return uint The total amount of tokens bought combining the tier prices
   function calculateExcessTokens(
      uint256 amount,
      uint256 tokensThisTier,
      uint256 tierSelected,
      uint256 _rate
   ) public returns(uint256 totalTokens) {
      require(amount > 0 && tokensThisTier > 0 && _rate > 0);
      require(tierSelected >= 1 && tierSelected <= 4);

      uint weiThisTier = tokensThisTier.sub(tokensRaised).div(_rate);
      uint weiNextTier = amount.sub(weiThisTier);
      uint tokensNextTier = 0;
      bool returnTokens = false;

      // If there's excessive wei for the last tier, refund those
      if(tierSelected != 4)
         tokensNextTier = calculateTokensTier(weiNextTier, tierSelected.add(1));
      else
         returnTokens = true;

      totalTokens = tokensThisTier.sub(tokensRaised).add(tokensNextTier);

      // Do the transfer at the end
      if(returnTokens) msg.sender.transfer(weiNextTier);
   }

   /// @notice Buys the tokens given the price of the tier one and the wei paid
   /// @param weiPaid The amount of wei paid that will be used to buy tokens
   /// @param tierSelected The tier that you'll use for thir purchase
   /// @return calculatedTokens Returns how many tokens you've bought for that wei paid
   function calculateTokensTier(uint256 weiPaid, uint256 tierSelected)
        internal constant returns(uint256 calculatedTokens)
   {
      require(weiPaid > 0);
      require(tierSelected >= 1 && tierSelected <= 4);

      if(tierSelected == 1)
         calculatedTokens = weiPaid.mul(rate);
      else if(tierSelected == 2)
         calculatedTokens = weiPaid.mul(rateTier2);
      else if(tierSelected == 3)
         calculatedTokens = weiPaid.mul(rateTier3);
      else
         calculatedTokens = weiPaid.mul(rateTier4);
   }
```
### Problem
In case weiNextTier in **calculateExcesTokens** function is larger than next limitTier[next tier number]/rateTier[next tier number] it is possible to buy tokens from higher tier for lower tier price.

### Recommendations
Simplest fix would be to revert at the end of **calculateTokensTier** function if calculatedTokens exceeds limitTier[tier number]. However, considering the maximum amount of ETH contribution in one transaction is limited to 1000 ETH and with the current price of ETH in mind and declared dollar price of tokens, the total price of tokens in the cheapest tier should be over 3000 ETH, this vulnerability shouldn't be exploitable.

## 7. Misc
- variable limitTier4 is never used
- check for maximum sale limit happens needlessly both in PallyCoin contract and Crowdsale contract

# Summary
Beyond the issues mentioned, the contract was checked for overflow/underflow issues, potential DoS attacks and re-entrancy attacks. None were discovered. 
Most issues of the code are related to ommisions or wrong implementations of buyer protecting functionality. Contract also contains overly complex convenience functionality that could be removed. Nonetheless, in its current state, I've identified no critical issues that could allow buyers to damage interests of the seller, or buyers to damage interests of other buyers. Considering the time constraints, I recommend all fixes of the mentioned issues to be as simple as possible.

# Disclaimer 
This audit was produced as an emergency measure under very strict time constraints and was focused on identifying only serious issues, it doesn't contain efficiency, testing or diagnostics recommendations and isn't exhaustive in any sense of the word.
