# First Flight #12: Kitty Connect - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Lack of access control in `KittyBridge:_ccipReceive()` allows anyone to mint a bridged cat without having to own it on the other chain](#H-01)

- ## Low Risk Findings
    - ### [L-01. The same shop partner can be added twice in `KittyConnect:addShop()`](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #12: Kitty Connect

### Dates: Mar 28th, 2024 - Apr 4th, 2024

[See more contest details here](https://www.codehawks.com/contests/clu7ddcsa000fcc387vjv6rpt)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
   - Medium: 0
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Lack of access control in `KittyBridge:_ccipReceive()` allows anyone to mint a bridged cat without having to own it on the other chain            



## Severity

**Impact**: Medium, since the owner can still bridge his NFT

**Likelihood**: High, since it can be performed by almost anyone

## Description

`_ccipReceive()` is a critical function in the CCIP system which is invoked on the receiver contract by the CCIP router.<br>
The issue is found in the passing of `msg.sender` as the `_sender` parameter of the `onlyAllowlisted` modifier from `KittyBridgeBase`.<br>

However, in order for the function not to revert, `msg.sender` **can only be the address of the CCIP router** because of the following public function exposed by `CCIPReceiver` (that KittyBridge inherits) that has the `onlyRouter` modifier:
```js
/// @inheritdoc IAny2EVMMessageReceiver
function ccipReceive(Client.Any2EVMMessage calldata message) external virtual override onlyRouter {
    _ccipReceive(message);
}
```

This makes possible for anyone to mint a cat NFT by **directly calling the router without passing from `KittyConnect:bridgeNftToAnotherChain()`**, the only function that is responsible for checking that the caller actually owns the NFT.

## POC

Refer to the following diagram: https://drive.google.com/file/d/1JolPeIiFjYYS1Um8TkUWhMSTg0oa2IWj/view?usp=sharing

## Recommendations
To always make sure that the NFT is owned and burn by the catOwner on the source chain, we must add proper access control to the functions listed below:
1. protect `KittyBridge:bridgeNftWithData()` to only allow the KittyConnect contract to call it
2. Change the second parameter of the `onlyAllowListed` as follows to pass the actual sender of the message (which will be the `KittyBridge` on the source chain)

```diff
onlyAllowlisted(
    any2EvmMessage.sourceChainSelector,
+   abi.decode(any2EvmMessage.sender, (address))
)
```

Of course, the `KittyBridge` on the destination chain, must allow-list the address of the `KittyBridge` on the source chain in his `allowlistedSenders` mapping.

The `any2EvmMessage.sender` is safely set by the CCIP router on the source chain:

```js
function ccipSend(uint64 destinationChainSelector, EVM2AnyMessage message)
{
    //..
    IEVM2AnyOnRamp(onRamp).forwardFromRouter(destinationChainSelector, message, feeTokenAmount, msg.sender);
}
```

Also refer to the following ChainLink CCIP official example:
https://docs.chain.link/ccip/tutorials/programmable-token-transfers-defensive#tutorial
		


# Low Risk Findings

## <a id='L-01'></a>L-01. The same shop partner can be added twice in `KittyConnect:addShop()`            



## Severity
**Impact**: Medium, since a "bad" shop partner can be allowed twice

**Likelihood**: Low, since only authorized user can perform such activity

## Description
`KittyConnect:addShop()` allows an admin to register a new shop partner in the KittyConnect protocol.
However, this function, doesn't check if a shop partner is already allowed before pushing it in the array of allowed partners.

```js
function addShop(address shopAddress) external onlyKittyConnectOwner {
        s_isKittyShop[shopAddress] = true;
        s_kittyShops.push(shopAddress);
        emit ShopPartnerAdded(shopAddress);
    }
```

## Recommendations
Change the function like this:

```diff
function addShop(address shopAddress) external onlyKittyConnectOwner {
+       if (s_isKittyShop[shopAddress])
+           revert KittyConnect__ShopAlreadyRegistered();
        s_isKittyShop[shopAddress] = true;
        s_kittyShops.push(shopAddress);
        emit ShopPartnerAdded(shopAddress);
    }
```


