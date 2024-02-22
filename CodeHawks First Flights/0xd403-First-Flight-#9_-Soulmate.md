# First Flight #9: Soulmate - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Incorrect staking rewards calculation allows anyone to obtain more tokens](#H-01)
    - ### [H-02. Any non-minter can claim staking rewards by incorrectly identifying himself as the owner of the NFT at index 0](#H-02)
    - ### [H-03. Any non-minter can claim airdrop rewards by incorrectly identifying himself as the owner of the NFT at index 0](#H-03)
- ## Medium Risk Findings
    - ### [M-01. `Soulmate::mintSoulmateToken()` allows anyone to soulmate with himself](#M-01)
- ## Low Risk Findings
    - ### [L-01. Informational and Gas findings](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #9

### Dates: Feb 8th, 2024 - Feb 15th, 2024

[See more contest details here](https://www.codehawks.com/contests/clsathvgg0005yhmxmoe455mm)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 3
   - Medium: 1
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Incorrect staking rewards calculation allows anyone to obtain more tokens            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Staking.sol#L74

https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Staking.sol#L81

## Summary

`Staking::claimRewards()` fails to correctly compute the amount of staking rewards a user should get.

## Vulnerability Details

The vulnerability can be identified in the fact that the contract only tracks the last claim time (`lastClaim`) and computes the rewards based on the current staking balance (`userStakes`).
Anyone can call `Staking::deposit()` just before claming the rewards so that their `userStakes` is higher, meaning that they will receive more rewards.

## Impact

Anyone can claim more tokens than normal by depositing before withdrawing.

## Tools Used

Add the following unit test to `StakingTest.t.sol`:
```js
function test_incorrectStakingRewardsCalculations() public {
        uint256 tokensToDeposit = 100e18;
        uint256 weeksOfStaking = 10;
        uint256 expectedRewards = tokensToDeposit * 10;

        loveToken.testMint(soulmate1, tokensToDeposit); // soulmate1 starts with 100 tokens

        vm.startPrank(soulmate1);
        loveToken.approve(address(stakingContract), type(uint256).max);
        stakingContract.deposit(tokensToDeposit);

        // fast forward of 10 weeks
        vm.warp(block.timestamp + weeksOfStaking * 1 weeks + 1 seconds);

        // just before claiming, he deposits 100 tokens more
        loveToken.testMint(soulmate1, tokensToDeposit);
        stakingContract.deposit(tokensToDeposit);
        stakingContract.claimRewards(); // here userStakes[msg.sender] = 200
        vm.stopPrank();

        uint256 actualRewards = loveToken.balanceOf(soulmate1);
        assert(actualRewards > expectedRewards);
        assert(expectedRewards == 1000e18);
        assert(actualRewards == 2000e18);
    }
```

In the example above the user initially deposits 100 tokens. Normally, after 10 weeks, he sould be able to claim 1000 but, before claiming, he deposits 100 token more. This causes `Staking::claimRewards()` to calculate the rewards on 200 tokens of initial deposit (even if only half of them actually stayed in the contract for 10 weeks) and he's able to claim 2000 tokens.

## Recommendations
Add another mapping that tracks the amount of claimable rewards for each user and use this mapping to correctly calculate the amount of staking rewards.

With the following precautions:
- `Soulmate::ownerToId()` should revert if 0 is returned because we want to avoid non-minters to be able to use the staking contract functions
- `Staking::deposit()` should update both the amount of claimable rewards and lastClaim BEFORE updating the `userStakes` mapping
- `Staking::deposit()` should update both the amount of claimable rewards and lastClaim BEFORE updating the `userStakes` mapping

<details>
<summary>Example implementation</summary>

```js
error Staking_notElgible();

mapping(address user => uint256 loveToken) public userStakes;
mapping(address user => uint256 claimable) public claimableRewards;
mapping(address user => uint256 timestamp) public lastDeposit;

function deposit(uint256 amount) public {
    if (loveToken.balanceOf(address(stakingVault)) == 0)
        revert Staking__NoMoreRewards();
        
    // this will revert if 0, so it checks if this user has an NFT
    require(soulmateContract.ownerToId(msg.sender) > 0);

    if (lastDeposit[msg.sender] > 0) {
        uint256 timeInWeeksSinceLastDeposit = ((block.timestamp - lastDeposit[msg.sender]) / 1 weeks);
        claimableRewards[msg.sender] += userStakes[msg.sender] * timeInWeeksSinceLastDeposit;
    }
        
    userStakes[msg.sender] += amount;
    lastDeposit[msg.sender] = block.timestamp;
    loveToken.transferFrom(msg.sender, address(this), amount);

    emit Deposited(msg.sender, amount);
}

function withdraw(uint256 amount) public {
    require(userStakes[msg.sender] > 0);
    uint256 timeInWeeksSinceLastDeposit = ((block.timestamp - lastDeposit[msg.sender]) / 1 weeks);
    claimableRewards[msg.sender] += userStakes[msg.sender] * timeInWeeksSinceLastDeposit;
    userStakes[msg.sender] -= amount;
    loveToken.transfer(msg.sender, amount);
    emit Withdrew(msg.sender, amount);
}

function claimRewards() public {
    if (lastDeposit[msg.sender] == 0) 
        revert Staking_notElgible();
        
    uint256 timeInWeeksSinceLastDeposit = ((block.timestamp - lastDeposit[msg.sender]) / 1 weeks);
        
    uint256 oldClaimableRewards = claimableRewards[msg.sender];
    if (timeInWeeksSinceLastDeposit < 1 && oldClaimableRewards == 0)
        revert Staking__StakingPeriodTooShort();

    uint256 amountToClaim = claimableRewards[msg.sender] + ( userStakes[msg.sender] * timeInWeeksSinceLastDeposit);

    claimableRewards[msg.sender] = 0; // update the claimable rewards
        lastDeposit[msg.sender] = block.timestamp;

    loveToken.transferFrom(
        address(stakingVault),
        msg.sender,
        amountToClaim
    );

    emit RewardsClaimed(msg.sender, amountToClaim);
}
```

</details>
## <a id='H-02'></a>H-02. Any non-minter can claim staking rewards by incorrectly identifying himself as the owner of the NFT at index 0            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Soulmate.sol#L26

https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Staking.sol#L74

## Summary

`Staking::claimRewards()` allows non-minters to fraudulently claim airdrop rewards by falsely identifying themselves as the owner of the NFT at index 0.

## Vulnerability Details

This vulnerability is made possible because:

- `Soulmate::ownerToId()`  incorrectly returns 0 for both the actual owners of the corresponding NFT's index and for individuals who have not yet minted an NFT
- the staking contract incorrectly initialize the `lastClaim` mapping

If a non-minter tries to call `Staking::claimRewards()` his staking time will be initialised at the minting time of the NFT at index 0
```js
if (lastClaim[msg.sender] == 0) {
    lastClaim[msg.sender] = soulmateContract.idToCreationTimestamp(soulmateId);
}
```

Also, due to the second point, anyone can obtain more rewards (even legitimate users), by depositing more tokens just before claiming staking rewards.

## Impact

The vulnerability enables unauthorised individuals to illegitimately accumulate rewards over time, thereby increasing the daily number of claimable tokens.

## Tools Used

Add the following code to `StakingTest.t.sol`:

```js
function test_CanClaimStakingRewardsByDepositingInSameTxn() public {
    loveToken.testMint(attacker, 100e18);
    uint256 initialBalance = loveToken.balanceOf(attacker);
    vm.startPrank(attacker);
    loveToken.approve(address(stakingContract), initialBalance); 
    stakingContract.deposit(initialBalance); // deposit all in staking contract
    stakingContract.claimRewards(); // and claim all rewards
    stakingContract.withdraw(initialBalance); // and withdraw the deposited funds
    vm.stopPrank();
    uint256 finalBalance = loveToken.balanceOf(attacker);
    console2.log("Balance after claiming staking: ", finalBalance);
}
```

PS: I added a `testMint()` function to the LoveToken contract just for testing purposes 

## Recommendations
1. make `Soulmate::nextId` start from 1 instead of 0
2. rename the mapping `Soulmate::ownerToId` to `Soulmate::_ownerToId` and make it private
3. add a wrapper public function to the now private mapping `Soulmate::_ownerToId` that throws if the returned id is 0

```js
mapping(address owner => uint256 id) private _ownerToId;

function ownerToId(address _owner) public view returns (uint256) {
        uint256 id = _ownerToId[_owner];
        if (id != 0) return id;
        else revert Soulmate_invalidId();
    }
```

By renaming you don't have to change all the other contracts that requires `Soulmate::ownerToId()` to work.
## <a id='H-03'></a>H-03. Any non-minter can claim airdrop rewards by incorrectly identifying himself as the owner of the NFT at index 0            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Soulmate.sol#L26

https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Airdrop.sol#L58

## Summary

`Airdrop::claim()` allows non-minters to fraudulently claim airdrop rewards by falsely identifying themselves as the owner of the NFT at index 0.

## Vulnerability Details

The `Soulmate::ownerToId()` function incorrectly returns 0 for both the actual owners of the corresponding NFT's index and for individuals who have not yet minted an NFT. Consequently, non-minters can exploit this flaw by falsely asserting ownership of the NFT at index 0, thereby claiming airdrop rewards based on the minting time of that NFT. 

## Impact

The vulnerability enables unauthorised individuals to illegitimately accumulate rewards over time, thereby increasing the daily number of claimable tokens.

Example: after 200 days all non-minters can potentially claim 200 tokens each

## Tools Used

Add the following uint test to `AirdropTest.t.sol`:

```js
function test_anyoneCanClaimAirdopRewards() public {
        _mintOneTokenForBothSoulmates();

        uint256 passedDays = 200;
        vm.warp(block.timestamp + passedDays * 1 days + 1 seconds);

        vm.prank(attacker);
        airdropContract.claim();

        assertTrue(loveToken.balanceOf(attacker) == passedDays * 1 ether);
    }
```

## Recommendations
1. make `Soulmate::nextId` start from 1 instead of 0
2. rename the mapping `Soulmate::ownerToId` to `Soulmate::_ownerToId` and make it private
3. add a wrapper public function to the now private mapping `Soulmate::_ownerToId` that throws if the returned id is 0

```js
mapping(address owner => uint256 id) private _ownerToId;

function ownerToId(address _owner) public view returns (uint256) {
        uint256 id = _ownerToId[_owner];
        if (id != 0) return id;
        else revert Soulmate_invalidId();
    }
```

By renaming you don't have to change all the other contracts that requires `Soulmate::ownerToId()` to work.
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. `Soulmate::mintSoulmateToken()` allows anyone to soulmate with himself            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Soulmate.sol#L74

## Summary

Anyone can soulmate with himself.

## Vulnerability Details

Due to the missing check when the second person calls `Soulmate::mintSoulmateToken()`, the first soulmate can call the function twice and soulmate with himself.

## Impact

Even if this doesn't break the other functionalities in major ways, it goes against the protocol's main goal (soulmate with another random individual).

## Tools Used

Add the following uint test to `SoulmateTest.t.sol`:

```js
function test_canSoulmateWithMySelf() public {
    vm.startPrank(soulmate1);
    soulmateContract.mintSoulmateToken();
    soulmateContract.mintSoulmateToken();
    assertTrue(soulmateContract.soulmateOf(soulmate1) == soulmate1);
    vm.stopPrank();
}
```

## Recommendations

Add the missing check in the else if block of the function:

```js
else if (soulmate2 == address(0)) {
	require(soulmate1 != msg.sender, "Can't soulmate with yourself");
    idToOwners[nextID][1] = msg.sender;
	
	// ...
	
	_mint(msg.sender, nextID++);
}
```

Alternatively you can also add this line after the memory variable `soulmate1` is initialised to save some gas:

```js
address soulmate1 = idToOwners[nextID][0];
require(soulmate1 != msg.sender, "Can't soulmate with yourself");
```

# Low Risk Findings

## <a id='L-01'></a>L-01. Informational and Gas findings            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/Soulmate.sol#L82

https://github.com/Cyfrin/2024-02-soulmate/blob/b3f9227942ffd5c443ce6bccaa980fea0304c38f/src/LoveToken.sol#L20

## [INFO-1] Wrong event emission in `Soulmate::mintSoulmanteToken()`

### Summary
The event `SoulmateAreReunited` is always emitted with address(0) as the value of the `soulmate2` parameter because a non-updated memory variable is used in the emission.

### Recommendations
replace `soulmate2` with msg.sender

```diff
- emit SoulmateAreReunited(soulmate1, soulmate2, nextID);
+ emit SoulmateAreReunited(soulmate1, msg.sender, nextID);
```

## [GAS-1] Unused state variable in `LoveToken:soulmateContract` that can be removed

### Summary
```js
ISoulmate public immutable soulmateContract;
```



