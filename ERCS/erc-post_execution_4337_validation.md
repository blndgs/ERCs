---

title: Extend 4337 with post-execution validation
description: This proposal introduces optional post-bundle execution validation, allowing user operations to be validated against the post-bundle execution network state.
author: Ofir Elias (EliasiOfir), Har Preet Singh (singhhp1069), Mario Karagiorgas (blewater).
status: Draft
type: Standards
category: ERC
created: 2024-02-05
requires: ERC-4337
---

## Abstract

This proposal extends the [ERC-4337](./erc-4337.md) standard with a new optional `IAccountPostExecution` interface declaring a post-bundle execution validation function. Thus, this proposal introduces a validation layer that occurs after the execution of the entire bundle, providing a comprehensive view of the final state against which each user operation can be validated. The EntryPoint contract's `handleOps` function will be updated to conditionally invoke validatePostExecution if signaled by a specific selector within the userOp's signature field.

## Motivation

The motivation behind enhancing ERC-4337 with a post-execution validation mechanism stems from the need to assess and validate the cumulative effects of user operations within a bundle, especially in light of their complex interdependencies and the final state they collectively produce. ERC-4337 cannot validate the state of a user operation after the execution of a bundle, limiting its use in scenarios where operations are interdependent. This limitation hinders assessing the effectiveness of executing dependent operations that rely on the outcome of preceding operations within the same bundle. The motivation for the post-execution validation mechanism is significantly broadened by considering user `Intents`, which may involve multiple interdependent userOp conditions.

## Specification

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Definitions

- **Account** - An [ERC-4337](./eip-4337.md) compliant smart contract account.
- **Module** - A smart contract with self-contained ERC-4337 functionality per [ERC-6900](./eip-6900.md), [ERC-7579](./eip-7579.md) standards.

### Validation Plugins

This proposal does not dictate the use of a particular module standard to support custom validation logic.

## Reference Implementation

### Selective Validation
Implementers can design their validation logic to execute conditionally based on the specific requirements of the user operation. Similar to the [`executeUserOp`](../assets/erc-post_execution_4337_validation/entrypoint_0.7.pdf) 4-byte selector of the `validatePostExecution` within signature signals the EntryPoint to call it.

### `IAccountPostExecution` Interface Enhancement
A new optional `validatePostExecution` function in IAccountPostExecution.

```Solidity
interface IAccountPostExecution {
    /** 
     * Validates the state of a user operation after bundle execution.
     * Reverts if it does not pass validation.
     */
    function validatePostExecution(UserOperation calldata userOp, bytes32 userOpHash) external view;
}
```

### Account Implementation
An Account contract may implement the validatePostExecution method to perform post-execution validations.

```Solidity
contract SimpleAccount is BaseAccount, TokenCallbackHandler, UUPSUpgradeable, Initializable, IAccountPostExecution {
    ...
    function validatePostExecution(UserOperation calldata userOp, bytes32 userOpHash) external view {
        // post-execution validation here
        ...
    }
    ...
}
```

### EntryPoint Contract Modification
The `EntryPoint` contract's `handleOps` function is modified to call `_validatePostExecution` for each UserOperation, which executes account post-execution validation. The validation dispatch follows the EntryPoint v0.7.0 pattern of calling conditionally the `execUserOperation`, based on the detection of a specific selector in the userOp's signature, which indicates whether the account has implemented the validatePostExecution function.

```Solidity
contract EntryPoint is IEntryPoint, StakeManager, NonceManager, ReentrancyGuard {
    ...
    function handleOps(UserOperation[] calldata ops, address payable beneficiary) public non-reentrant {

        // Prevalidation...

        // EntryPoint v0.7.0 execution loop
        uint256 collected = 0;
        for (uint256 i = 0; i < ops.length; i++) {
            collected += _executeUserOp(ops[i]);
        }

        // ** Proposed addition: 
        // Invoke post-execution validation for each user operation
        for (uint256 i = 0; i < ops.length; i++) {
            _validatePostExecution(ops[i], opInfos[i].userOpHash);
        }

        // ...
    }
    ...
}

function _validatePostExecution(uint256 opIndex, UserOperation calldata userOp, bytes32 userOpHash) internal {
    bytes4 selector;

    // Extract the first 4 bytes of the signature as the selector
    bytes calldata signature = userOp.Signature;
    assembly {
        // omitting length check for brievity
        selector := mload(add(signature, 32))
    }

    if (selector == IAccount.validatePostExecution.selector) {
        bytes memory callData = abi.encodeCall(
            IAccountPostExecution.validatePostExecution,
            (userOp, userOpHash)
        );

        // Call the validatePostExecution function on the target account
        (bool success, bytes memory returndata) = address(userOp.sender).call(callData);
        if (!success) {
            if (returndata.length > 0) {
                // revert data is UTF-8 error message
                revert(string(returndata));
            }
            revert();
        }
    }
}
```

## Rationale

This enhancement aims to unlock new capabilities and scenarios in Intents fulfillment and DApps, by enabling post-execution bundle validation. This feature is particularly beneficial in complex transaction sequences where one operation's outcome may affect subsequent ones' within the same transaction bundle.

### Scenario Examples
In the context of Intents fulfillment, bundles likely represent the execution of a single user Intent using one or more userOp execution steps. 

#### DeFi Sequences 
A user performs a token swap followed by liquidity provision and staking in one transaction. Post-execution validation ensures that the entire sequence meets the user's conditions based on the final state, such as minimum received tokens.

#### Blockchain-based Games
In-game transactions that depend on specific game states can be validated post-execution. Such validation ensures that actions like trades or character upgrades only proceed if prior operations result in the expected state.

#### Social Recovery
Validates the completion of a social recovery process based on approvals from designated guardians, ensuring the integrity of the recovery operation post-execution.

## Alternative design considerations

During the design phase, we considered the existing `postOp` function utilized by Paymasters as an alternative mechanism for post-operation validation. As defined within the ERC-4337 standard, Paymaster's approach facilitates gas payment delegation and handles validation immediately after executing each user operation. While functionally significant, this approach cannot inherently assess the state changes produced by the entirety of a user operation bundle. Moreover, the semantics and primary intent of the Paymaster's post-operation validation are focused on the financial aspect of operations.

## Backwards Compatibility

Introducing the `validatePostExecution` function and the `IAccountExecution` interface and its optional implementation in the accounts maintains compatibility with existing account deployments.

Selective Execution
Similar to the [v0.7.0 executeUserOp](../assets/erc-post_execution_4337_validation/entrypoint_0.7.pdf) selective execution approach of including a 4-byte selector within the signature to signal operations requiring this additional validation step will incur the associated gas costs.

## Security Considerations

### DOS, Griefing attacks
Heightened gas consumption poses risks of network congestion and opens avenues for malicious entities to exploit these mechanisms, intentionally crafting operations that fail validation to waste resources or undermine the validation process for legitimate operations within the same bundle.

### Mitigation through Stake-Based Throttling and Banning
Analogous to ERC-4337's approach to mitigating DoS attacks through economic disincentives, entities causing the invalidation of multiple userOps due to failed post-execution validation are subject to throttling, temporal banning, or through a required stake.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
