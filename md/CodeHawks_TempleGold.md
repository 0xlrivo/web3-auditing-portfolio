# TempleGold - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Smart contract wallet won't be able to transfer TGLD to other chains](#H-01)

- ## Low Risk Findings
    - ### [L-01. TempleTeleporter:quote(.., _to, ...) does not estimate exact nativeFee due to inconsistent usage of abi.encodePacked()](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: TempleDAO

### Dates: Jul 4th, 2024 - Jul 11th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-templegold)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 0
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Smart contract wallet won't be able to transfer TGLD to other chains            



## Summary

Smart contract wallet won't be able to transfer TGLD to other chains.

## Vulnerability Details

The documentation states that users can only transfer TGLD cross-chain to self, which is properly enforced in `TempleGold.sol:send()`:

```Solidity
/// @dev user can cross-chain transfer to self
if (msg.sender != _to) { revert ITempleGold.NonTransferrable(msg.sender, _to); }
```

However, since smart contract wallets like Gnosis Safes (which the team uses), do have different addresses on different chains making impossible for those wallets to transfer their TGLD cross-chain.

## Impact

Smart contract wallets, like Gnosis Safes, won´t be able to transfer their TGLD cross-chain.

## Tools Used

Manual Review

## Recommendations

Remove the limitation of only transferring TGLD cross-chain to self to also allow smart contracts wallet to use this functionality.

    


# Low Risk Findings

## <a id='L-01'></a>L-01. TempleTeleporter:quote(.., _to, ...) does not estimate exact nativeFee due to inconsistent usage of abi.encodePacked()            



## Summary

`TempleTeleporter:quote()` does not estimate the exact `nativeFee` due to the inconsistent usage of `abi.encodePacked()`.

## Vulnerability Details

In `teleport()` the message payload is packed-encoded after converting `_to` to `bytes32` in order to correctly pad the result so that it can be decoded on destination.

```Solidity
// this is equivalent to abi.encode(to, amount)
bytes memory _payload = abi.encodePacked(to.addressToBytes32(), amount);
```

However, the `quote()`function with the `_to` parameter,  incorrectly uses `encodePacked` without converting the parameter to a `bytes32` first:

```Solidity
function quote(
    uint32 _dstEid,
    address _to,
    uint256 _amount,
    bytes memory _options
) external view returns (MessagingFee memory fee) {
    // @audit-issue _to should be extended to a bytes32
    return _quote(_dstEid, abi.encodePacked(_to, _amount), _options, false);
}
```

Meaning that the function will quote a lighter payload since `encodePacked` will trim the 24 leading 0-bytes of the address, leading to a lower `nativeFee` amount than the actual payload that will be sent by `teleport()`.

For example:

```Solidity
abi.encode(address, uint): // actual payload
0x0000000000000000000000007ad846ad333469284b981c5d780580c1337677d20000000000000000000000000000000000000000000000000000000000000016

abi.encodePacked(address, uint): // quote(.., _to, ...) payload
0x7ad846ad333469284b981c5d780580c1337677d20000000000000000000000000000000000000000000000000000000000000016
```

## Impact

If the `quote()` above is used before sending a message it will estimate a lower gas amount than expected and potentially lead to "out of gas" failures on the destination chain.

I belive this finding to be of low severity since:

* HIGH impact -> since there is a direct loss of funds for users&#x20;
* VERY LOW likelihood -> since the actual gas difference is very low thus unlikely to actually cause a revert

## POC

Add the following test to `TempleTeleporter.t.sol`:

```Solidity
function test_quoteTo() public {
        bytes memory options = OptionsBuilder.newOptions().addExecutorLzReceiveOption(200000, 0);

        // quote() implementation without the _to parameter - payload actually sent by teleport()
        MessagingFee memory quote = aTT.quote(bEid, abi.encode(userB, 10 ether), options);

        // quote() implementation with the _to parameter
        MessagingFee memory quoteTo = aTT.quote(bEid, userB, 10 ether, options);

        // 210504 < 210516
        assert(quoteTo.nativeFee < quote.nativeFee);
    }
```

## Tools Used

Manual Review

## Recommendations

Modify `quote(.., _to, ...)` in order to be consistent with how `teleport()` constructs the payload:

```diff
function quote(
    uint32 _dstEid,
    address _to,
    uint256 _amount,
    bytes memory _options
) external view returns (MessagingFee memory fee) {
    // @audit-issue _to should be extended to a bytes32
-   return _quote(_dstEid, abi.encodePacked(_to, _amount), _options, false);
+   return _quote(_dstEid, abi.encodePacked(_to.addressToBytes32(), _amount), _options, false);  
}
```



