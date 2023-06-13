---
title: EIP-1155 Permit Approvals
description: Permit approvals for ERC-1155 tokens
author: calvbore (@calvbore)
discussions-to: <URL>
status: Draft
type: Standards Track
category: ERC
created: <date created on, in ISO 8601 (yyyy-mm-dd) format>
requires: 165, 712, 1155, 5216
---

<!--
  READ EIP-1 (https://eips.ethereum.org/EIPS/eip-1) BEFORE USING THIS TEMPLATE!

  This is the suggested template for new EIPs. After you have filled in the requisite fields, please delete these comments.

  Note that an EIP number will be assigned by an editor. When opening a pull request to submit your EIP, please use an abbreviated title in the filename, `eip-draft_title_abbrev.md`.

  The title should be 44 characters or less. It should not repeat the EIP number in title, irrespective of the category.

  TODO: Remove this comment before submitting
-->

## Abstract

The "permit" approval flow for both ERC-20 and ERC-721 tokens outline in [EIP-4494](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-4494.md) and [EIP-2612](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2612.md) respectively are large improvements for the existing UX of the token underlying each EIP. This EIP extends the "permit" pattern to ERC-1155 tokens, borrowing heavily upon both EIP-4494 and EIP-2612.

The structure of ERC-1155 tokens requires a new EIP to account for the token standard's use of both token IDs and balances (also why this EIP requires [EIP-5216](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-5216.md)). An additional field of arbitrary `bytes` is added for extra data to be added to the signatures in the case that the permit function would be extended in some way.  

## Motivation

<!--
  This section is optional.

  The motivation section should include a description of any nontrivial problems the EIP solves. It should not describe how the EIP solves those problems, unless it is not immediately obvious. It should not describe why the EIP should be made into a standard, unless it is not immediately obvious.

  With a few exceptions, external links are not allowed. If you feel that a particular resource would demonstrate a compelling case for your EIP, then save it as a printer-friendly PDF, put it in the assets folder, and link to that copy.

  TODO: Remove this comment before submitting
-->

The permit structures outlined in both EIP-4494 and EIP-2612 allows a signed message to create an approval, but are only applicable to their respective underlying tokens (ERC-721 and ERC-20).

While the permit flow described in the above EIPs greatly improves UX and contract interactions they do not allow for any extensibility to the permit flow. It may be that implementors would like to verify additional data than what is strictly required by the permit signature.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

Three new functions must be added to ERC-1155 and ERC-5216.

```solidity
interface IERC1155Permit {
	function permit(address owner, address operator, uint256 tokenId, uint256 value, uint256 deadline, bytes memory data, bytes memory sig) external;
	function nonces(address owner, uint256 tokenId) external view returns (uint256);
	function DOMAIN_SEPARATOR() external view returns (bytes32);
}
```

The semantics of which are as follows:

For all addresses `owner`, `spender`, uint256's `tokenId`, `value`, `deadline`, and `nonce`, bytes `data` and `sig`, a call to `permit(owner, spender, tokenId, value, deadline, data, sig)` MUST set `allownace(owner, spender, tokenId)` to `value`, increment `nonces(owner, tokenId)` by 1, and emit a corresponding `ApprovalByAmount` event, if and only if the following conditions are met:
- The current blocktime is less than or equal to `deadline`
- `owner` is not the zero address
- `nonces[owner][tokenId]` (before state update) is equal to `nonce`
- `sig` is a valid `secp256k1`, [EIP-2098](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2098.md), or [EIP-1271](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1271.md) signature from `owner` of the message:
```
keccak256(abi.encodePacked(
   hex"1901",
   DOMAIN_SEPARATOR,
   keccak256(abi.encode(
            keccak256("Permit(address owner,address spender,uint256 tokenId,uint256 value,uint256 nonce,uint256 deadline,bytes data)"),
            owner,
            spender,
            tokenId,
            value,
            nonce,
            deadline,
            data))
));
```

If any of these conditions are not met the `permit` call MUST revert.

Where `DOMAIN_SEPARATOR` MUST be defined according to [EIP-712](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md). The `DOMAIN_SEPARATOR` should be unique to the contract and chain to prevent replay attacks from other domains, and satisfy the requirements of EIP-712, but is otherwise unconstrained. A common choice for `DOMAIN_SEPARATOR` is:
```
DOMAIN_SEPARATOR = keccak256(
    abi.encode(
        keccak256('EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)'),
        keccak256(bytes(name)),
        keccak256(bytes(version)),
        chainid,
        address(this)
));
```

In other words, the message is the following ERC-712 typed structure:
```
{
  "types": {
    "EIP712Domain": [
      {
        "name": "name",
        "type": "string"
      },
      {
        "name": "version",
        "type": "string"
      },
      {
        "name": "chainId",
        "type": "uint256"
      },
      {
        "name": "verifyingContract",
        "type": "address"
      }
    ],
    "Permit": [
	  {
	    "name": "owner".
	    "type": "address"
	  },
      {
        "name": "spender",
        "type": "address"
      },
      {
        "name": "tokenId",
        "type": "uint256"
      },
      {
        "name": "value",
        "type": "uint256"
      },
      {
        "name": "nonce",
        "type": "uint256"
      },
      {
        "name": "deadline",
        "type": "uint256"
      },
      {
        "name": "data",
        "type": "bytes"
      }
    ],
    "primaryType": "Permit",
    "domain": {
      "name": erc1155name,
      "version": version,
      "chainId": chainid,
      "verifyingContract": tokenAddress
  },
  "message": {
    "owner": owner,
    "spender": spender,
    "tokenId": tokenId,
    "value": value,
    "nonce": nonce,
    "deadline": deadline,
    "data": data
  }
}}
```

The `permit` function MUST check that the signer is not the zero address.

Note that nowhere in this definition do we refer to `msg.sender`. The caller of the `permit` function can be any address.

This EIP requires [EIP-165](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-165.md). EIP165 is already required in [ERC-1155](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1155.md), but is further necessary here in order to register the interface of this EIP. Doing so will allow easy verification if an NFT contract has implemented this EIP or not, enabling them to interact accordingly. The EIP-165 interface of this EIP is `0x7409106d`. Contracts implementing this EIP MUST have the `supportsInterface` function return `true` when called with `0x7409106d`.

## Rationale

<!--
  The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work, e.g. how the feature is supported in other languages.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

The `permit` function is sufficient for enabling a `safeTransferFrom` transaction to be made without the need for an additional transaction.

The format avoids any calls to unknown code.

The `nonces` mapping is given for replay protection.

A common use case of permit has a relayer submit a Permit on behalf of the owner. In this scenario, the relaying party is essentially given a free option to submit or withhold the Permit. If this is a cause of concern, the owner can limit the time a Permit is valid for by setting deadline to a value in the near future. The `deadline` argument can be set to `uint(-1)` to create Permits that effectively never expire. Likewise, the `value` argument can be set to `uint(-1)` to create Permits with effectively unlimited allowances.

ERC-712 typed messages are included because of its use in [EIP-4494](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-4494.md) and [EIP-2612](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2612.md), which in turn cites widespread adoption in many wallet providers.

This EIP focuses on both the `value` and `tokenId` being approved, EIP-4494 focuses only on the `tokenId`, while EIP-2612 focuses primarily on the `value`. EIP-1155 does not natively support approvals by amount, thus this EIP requires EIP-5216, otherwise a `permit` would grant approval for an account's entire `tokenId` balance.

Whereas ERC-2612 splits signatures into their `v,r,s` components, this EIP opts to instead take a `bytes` array of variable length in order to support [EIP-2098](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2098) or [EIP-1271](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1271.md) signatures, which may not be easily separated or reconstructed from `r,s,v` components (65 bytes).

An additional parameter `data` is added to the `permit` in order to offer extensibility to implementations of this EIP.

## Backwards Compatibility

<!--

  This section is optional.

  All EIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The EIP must explain how the author proposes to deal with these incompatibilities. EIP submissions without a sufficient backwards compatibility treatise may be rejected outright.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

No backward compatibility issues found.

## Test Cases

<!--
  This section is optional for non-Core EIPs.

  The Test Cases section should include expected input/output pairs, but may include a succinct set of executable tests. It should not include project build files. No new requirements may be be introduced here (meaning an implementation following only the Specification section should pass all tests here.)
  If the test suite is too large to reasonably be included inline, then consider adding it as one or more files in `../assets/eip-####/`. External links will not be allowed

  TODO: Remove this comment before submitting
-->

## Reference Implementation

<!--
  This section is optional.

  The Reference Implementation section should include a minimal implementation that assists in understanding or implementing this specification. It should not include project build files. The reference implementation is not a replacement for the Specification section, and the proposal should still be understandable without it.
  If the reference implementation is too large to reasonably be included inline, then consider adding it as one or more files in `../assets/eip-####/`. External links will not be allowed.

  TODO: Remove this comment before submitting
-->

## Security Considerations

Special attention should be held to any extra `data` being verified.

The below considerations have been copied from EIP-4494.

Extra care should be taken when creating transfer functions in which `permit` and a transfer function can be used in one function to make sure that invalid permits cannot be used in any way. This is especially relevant for automated NFT platforms, in which a careless implementation can result in the compromise of a number of user assets.

The remaining considerations have been copied from [ERC-2612](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2612.md) with minor adaptation, and are equally relevant here:

Though the signer of a `Permit` may have a certain party in mind to submit their transaction, another party can always front run this transaction and call `permit` before the intended party. The end result is the same for the `Permit` signer, however.

Since the ecrecover precompile fails silently and just returns the zero address as `signer` when given malformed messages, it is important to ensure `ownerOf(tokenId) != address(0)` to avoid `permit` from creating an approval to any `tokenId` which does not have an approval set.

Signed `Permit` messages are censorable. The relaying party can always choose to not submit the `Permit` after having received it, withholding the option to submit it. The `deadline` parameter is one mitigation to this. If the signing party holds ETH they can also just submit the `Permit` themselves, which can render previously signed `Permit`s invalid.

The standard [ERC-20 race condition for approvals](https://swcregistry.io/docs/SWC-114) applies to `permit` as well.

If the `DOMAIN_SEPARATOR` contains the `chainId` and is defined at contract deployment instead of reconstructed for every signature, there is a risk of possible replay attacks between chains in the event of a future chain split..

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
