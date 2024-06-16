# First Flight #10: One Shot - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. `RapBattle::goOnStageOrBattle()` allows anyone to use an NFT they do not own to become the challenger](#H-01)
    - ### [H-02. Due to poor randomness and incorrect winner selection both parties can gain unfair advantages in `RapBattle::goOnStageOrBattle()`](#H-02)
    - ### [H-03. `RapBattle::goOnStageOrBattle()` allows an inexistent NFT to become the challenger without needing to risk any cred](#H-03)
- ## Medium Risk Findings
    - ### [M-01. Missing NFT existence check in more view functions ](#M-01)
- ## Low Risk Findings
    - ### [L-01. Incorrect event emission in `RapBattle::_battle()`](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #10

### Dates: Feb 22nd, 2024 - Feb 29th, 2024

[See more contest details here](https://www.codehawks.com/contests/clstf5qd2000rakskkj0lkm33)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 3
   - Medium: 1
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. `RapBattle::goOnStageOrBattle()` allows anyone to use an NFT they do not own to become the challenger            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L49

## Summary

`RapBattle:goOnStageOrBattle()` allows anyone to use an NFT they do not own to become the challenger.

## Vulnerability Details

The vulnerability is caused by the fact that both `RapBattle::goOnStageOrBattle()` and `RapBattle::_battle()` do not require the challenger to prove he's actually the owner of the NFT with `tokenId` passed as a parameter.

This way, anyone can use an NFT they do not own (maybe with an higher stats compared to their) to become the challenger.

## Impact

Anyone can falsely use another user's NFT to become the challenger.

## Tools Used

- Manual Review

Add the following test to demonstrate the issue:

```js

function _mintAndStakeRapper(address _user, uint256 _stakingDays) public returns(uint256 mintedTokenId) {
        vm.startPrank(_user);
        mintedTokenId = oneShot.getNextTokenId(); 
        oneShot.mintRapper();
        assert(oneShot.ownerOf(mintedTokenId) == _user);
        oneShot.approve(address(streets), mintedTokenId);
        streets.stake(mintedTokenId);
        vm.warp(block.timestamp + (_stakingDays * 1 days) + 1 seconds);
        streets.unstake(mintedTokenId);
        vm.stopPrank();
    }

function test_challengerCanBeNonOwner() public {
        address user1 = makeAddr("user1");
        
        uint256 defenderNft = _mintAndStakeRapper(user, 2);
        uint256 user1Nft = _mintAndStakeRapper(user1, 2);
        uint256 challengerNft = _mintAndStakeRapper(challenger, 2);

        vm.startPrank(user);
        oneShot.approve(address(rapBattle), defenderNft);
        cred.approve(address(rapBattle), 2);
        rapBattle.goOnStageOrBattle(defenderNft, 2);
        vm.stopPrank();

        // challenger do not own user1Nft
        assert(oneShot.ownerOf(user1Nft) != challenger);

        vm.prank(challenger);
        rapBattle.goOnStageOrBattle(user1Nft, 2);
    }
```

## Recommendations

Force the challenger to transfer his NFT so you can be sure that:

- the NFT actually exists
- the challenger actually owns the NFT

Alternatively, if you don't want to waste gas for the two additional transfers required for this first fix, you can just add the following checks inside `goOnStageOrBattle`:

```diff
function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
        if (defender == address(0)) {
            // ...
        } else {
+           require(oneShotNft.ownerOf(_tokenId) == msg.sender);
+           require(credToken.balanceOf(msg.sender) >= _credBet);
            _battle(_tokenId, _credBet);
        }
    }
```

Note: if the `_tokenId` do not exists `ERC721::ownerOf()` will revert with `ERC721NonexistentToken`

Note: we can also assert before entering the battle that the challenger has enough cred tokens to pay if he loses. It isn't mandatory since the transaction will revert later even without it but, by checking before entering `_battle()`, we can save some gas.

## <a id='H-02'></a>H-02. Due to poor randomness and incorrect winner selection both parties can gain unfair advantages in `RapBattle::goOnStageOrBattle()`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L62

https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L70

## Summry

Due to how randomness is generated and how the winner is chosen, both defenders and challengers can get an unfair advantage on the other.

## Impact

Severely compomosises the normal battle functionality of the protocol.

## Vulnerability Details

The root cause is found in the fact that the "random" number computed inside `RapBattle::_battle()` is moduled with the sum of the two rapper's skill levels.

```js
uint256 random = uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % totalBattleSkill;
```

Firstly, generating a random number on-chain is dangerous, because any challenger can predict the outcome before the transaction is mined and simply revert if they lose.

The, this "random" number is compared with the defender' skill level and, if `<=`, the defender wins. This will cause the defender to always have more possibilities to win compared with a challenger in the same situation.

```js
if (random <= defenderRapperSkill) {
    // defender wins
}
else {
    // challenger wins
}
```

## Tools Used

- Manual Review

We first need to undestand that the modulo operation, will always return a number between 0 and n-1.

Ex: x % 10 will always return between 0 and 9

For the following scenario we'll consider the worst-case, where one rapper has the default skill level of 50 and the other has the maximum possible of 75:

<details> 
<summary>Equal skill levels</summary>

Both defender and challenger have an equal skill level of 50.

```js
skillLevelDefender = 50;
skillLevelAttacker = 50;
totalBattleSkill = 100; // number can range between 0 and 99

Defender wins: 
0, 1, 2, ..., 50 (51 possibilities)

Challenger wins: 
51, 52, ..., 99 (49 possibilities)
```

In this case the defender always have 1 more possibility to win due to the `<=` condition.

</details>


<details>
<summary> defender's level > challenger's level</summary>

Here, defender has an higher skill level than the challenger.

```js
skillLevelDefender = 75;
skillLevelAttacker = 50;
totalBattleSkill = 125; // number can range between 0 and 124

Defender wins: 
0, 1, 2, ... 75 (76 possibilities)

Challenger wins: 
76, 77, ..., 124 (49 possibilities)
```

Here the defender gains 1 more possibility to win!

</details>

<details>
<summary> Defender's level < challenger's level</summary>

Here, the challenger has a greated skill level than the defender.

```js
skillLevelDefender = 50;
skillLevelAttacker = 75;
totalBattleSkill = 125; // number can range between 0 and 124

Defender wins: 
0, 1, 2, ..., 50 (51 possibilities)

Challenger wins: 
51, 52, ..., 124 (74 possibilities)
```

Here the challenger has 1 less possibility to win!

</details><br/>

In summary, whatever the case, the defender always have more possibilities to win than normally.
But, due to predictable randomness, the challenger can simply revert if he doesn't win.

## Reccomended Mitigations

First thing first, the random number must be generated using Chainlink VRF so that it can't be predicted before the transaction is mined.

But, even with VRF in place, the winning check should be changed. We provide two possible soulutions:

1. After generating the random number add +1 to it
2. Change the `<=` with `<`, without neding the +1 in this case

More informations about Chainlink VRF [here](https://docs.chain.link/vrf)
## <a id='H-03'></a>H-03. `RapBattle::goOnStageOrBattle()` allows an inexistent NFT to become the challenger without needing to risk any cred            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L49

## Summary

Anyone can call `RapBattle::goOnStageOrBattle()`, as a challenger, passing a `tokenId` which isn't already been minted.

## Vulnerability Details

The vulnerability is made possible because only the defender is required to transfer his NFT (so it must exist in this case) and his creds to the contract.

Also, because of `OneShot::getRapperStats()` suffering from the same problem, any inexistent NFT will always have a pretty high skill score of 65.

Note: for this to work the attacker don't even need to have cred tokens and, if he loses, the transaction will simply revert because he doesn't have the tokens to send to the winning defender. 

## Impact

Anyone can infinitely challenge the defenders with an inexistent NFT until they win, causing them to claim free `Cred` tokens.

## Tools Used

- Manual Review

In the following test we demonstrate that an attacker is able to use an inexistent NFT with a skill score of 65 to enter the battle.
And thus, if he knows the defender has a lower skill level, he's more probable to win.

<details>
<summary>View POC test</summary>

```js
function _mintAndStakeRapper(address _user, uint256 _stakingDays) public returns(uint256 mintedTokenId) {
    vm.startPrank(_user);
    mintedTokenId = oneShot.getNextTokenId(); 
    oneShot.mintRapper();
    assert(oneShot.ownerOf(mintedTokenId) == _user);
    oneShot.approve(address(streets), mintedTokenId);
    streets.stake(mintedTokenId);
    vm.warp(block.timestamp + (_stakingDays * 1 days) + 1 seconds);
    streets.unstake(mintedTokenId);
    vm.stopPrank();
}

function test_challengerCanBeAnInexistentNFT() public {

    uint256 defenderTokenId = _mintAndStakeRapper(user, 1);

    // We take the next token to be minted for out "inexistent challenger"
    uint256 inexistentTokenId = oneShot.getNextTokenId();

    uint256 defenderSkillLevel = rapBattle.getRapperSkill(defenderTokenId);
    uint256 challengerSkillLevel = rapBattle.getRapperSkill(inexistentTokenId);
    uint256 defenderCred = cred.balanceOf(user);
    assert(challengerSkillLevel == 65);
    assert(challengerSkillLevel > defenderSkillLevel);
        
    // user is the defender
    vm.startPrank(user);
    cred.approve(address(rapBattle), defenderCred);
    oneShot.approve(address(rapBattle), defenderTokenId);
    rapBattle.goOnStageOrBattle(defenderTokenId, defenderCred);
    vm.stopPrank();

    assert(cred.balanceOf(challenger) == 0);

    vm.startPrank(challenger);
    rapBattle.goOnStageOrBattle(5, defenderCred);
    vm.stopPrank();
}  
```
</details>

## Recommendations

Force the challenger to transfer his NFT so you can be sure that:

- the NFT actually exists
- the challenger actually owns the NFT

Alternatively, if you don't want to waste gas for the two additional transfers required for this first fix, you can just add the following checks inside `goOnStageOrBattle`:

```diff
function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
        if (defender == address(0)) {
            // ...
        } else {
+           require(oneShotNft.ownerOf(_tokenId) == msg.sender);
+           require(credToken.balanceOf(msg.sender) >= _credBet);
            _battle(_tokenId, _credBet);
        }
    }
```

Note: if the `_tokenId` do not exists `ERC721::ownerOf()` will revert with `ERC721NonexistentToken`

Note: we can also assert before entering the battle that the challenger has enough cred tokens to pay if he loses. It isn't mandatory since the transaction will revert later even without it but, by checking before entering `_battle()`, we can save some gas.
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Missing NFT existence check in more view functions             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/OneShot.sol#L57

https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L86

## Summary

Both `OneShot::getRapperStats()` and `RapBattle::getRapperSkill()` do not check that the arbitrarily provided `tokenId` actually exists.

## Vulnerability Details

Due to missing checks that the passed tokenId is already been minted, those functions create inconsistent state in the protocol.

## Impact

`RapBattle::getRapperSkill()` makes possible for anyone to freely obtain a "non-minted rapper" with a high skill of 65, that will normally require 3 staking days to obtain.

## Tools Used

- Manual review

In the trace below, I call `OneShot::getRapperStats()` with a tokenId not yet minted, and the function returns a RapperStats struct with state-default values for his attributes (that corresponds to a skill level of 65):

```js
[PASS] test_noAccessControlOnRapperStats() (gas: 13753)
Traces:
  [13753] RapBattleTest::test_noAccessControlOnRapperStats()
    ├─ [2314] OneShot::getNextTokenId() [staticcall]
    │   └─ ← 0
    ├─ [5183] OneShot::getRapperStats(2) [staticcall]
    │   └─ ← RapperStats({ weakKnees: false, heavyArms: false, spaghettiSweater: false, calmAndReady: false, battlesWon: 0 })
    └─ ← ()
```

## Recommendations

Add existence checks to the pointed functions.

# Low Risk Findings

## <a id='L-01'></a>L-01. Incorrect event emission in `RapBattle::_battle()`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L67

## Summary

The function `RapBattle::_battle()` can incorrectly emit the Battle event in a particular case.

## Vulnerability Details

In the event, the defender wins if random is strictly less than his skill level.

```js
emit Battle(msg.sender, _tokenId, random < defenderRapperSkill ? _defender : msg.sender);
```

But, after the event is emitted, the if block uses a <= to determine the winner.

```js
if (random <= defenderRapperSkill) {
    // Here defender wins
}
```

So, if random happens to be equal to the defender skill level, the event will emit the incorrect winner.

## Impact

The users and the off-chain detection tools will register the incorrect winner.

## Reccomended Mitigation

Simply change the chech condition inside the event to match the correct one used after it:

```diff
-emit Battle(msg.sender, _tokenId, random < defenderRapperSkill ? _defender : msg.sender);
+emit Battle(msg.sender, _tokenId, random <= defenderRapperSkill ? _defender : msg.sender);
```


