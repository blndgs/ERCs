---

title: Extend 4337 with post-execution validation
description: This proposal introduces post-bundle execution validation, allowing user operations to be validated against the final state of the network.
author: Ofir Elias (EliasiOfir), Har Preet Singh (singhhp1069), Mario Karagiorgas (blewater).
status: Draft
type: Standards
category: ERC
created: 2024-02-05
requires: ERC-4337
---

## Abstract

This proposal extends [ERC-4337](./erc-4337.md) wallets' IAccount interface with a post-bundle execution validation function. Moreover, it proposes amending the EntryPoint contract's `handleOps()` function to call the proposed IAccount validation function. Thus, this proposal introduces a validation layer that occurs after the execution of the entire bundle, providing a comprehensive view of the final state against which each user operation can be validated.

## Motivation
The motivation behind enhancing ERC-4337 with a post-execution validation mechanism stems from the need to assess and validate the cumulative effects of user operations within a bundle, especially in light of their complex interdependencies and the final state they collectively produce. ERC-4337 cannot validate the state of a user operation after the execution of a bundle, limiting its use in scenarios where operations are interdependent. This limitation hinders assessing the effectiveness of executing dependent operations that rely on the outcome of preceding operations within the same bundle. For instance, in DeFi, a user aiming to perform a sequence of swaps and stakings in a single transaction (bundle) lacks the assurance that the entire operation can be validated post-execution, leading to potential risks. Similarly, decentralized games and social recovery mechanisms cannot enforce conditions based on the final state after multiple operations.

## Specification

IAccount Interface Enhancement
The IAccount interface is extended to include a validatePostExecution method. After executing the entire bundle, `validatePostExecution()` executes custom userOp validation logic.

```Solidity
interface IAccount {
    function validatePostExecution(UserOperation calldata userOp, bytes32 userOpHash) external returns (uint256 validationData);
}
```

SimpleAccount Implementation
The SimpleAccount contract should implement the validatePostExecution method, providing a template for how accounts can perform post-execution validations. This method could, for instance, check whether the account's state after the bundle execution meets certain criteria specified in the userOp.

```Solidity
contract SimpleAccount is BaseAccount, TokenCallbackHandler, UUPSUpgradeable, Initializable {
    ...
    function validatePostExecution(UserOperation calldata userOp, bytes32 userOpHash) external override returns (uint256 validationData) {
        // Implement post-execution validation logic here
        ...
    }
    ...
}
```

EntryPoint Contract Modification
After executing the entire bundle, the EntryPoint contract's `handleOps()` function should be modified to call the validatePostExecution method for each userOp. This ensures that all post-execution validations are performed before compensating the caller's beneficiary address with the collected fees.

```Solidity
contract EntryPoint is IEntryPoint, StakeManager, NonceManager, ReentrancyGuard {
    ...
    function handleOps(UserOperation[] calldata ops, address payable beneficiary) public non-reentrant {
        ...
        for (uint256 i = 0; i < opslen; i++) {
            collected += _executeUserOp(i, ops[i], opInfos[i]);
        }

        // Perform post-bundle execution validation for each userOp
        for (uint256 i = 0; i < opslen; i++) {
            _validatePostExecution(ops[i], opInfos[i]);
        }

        _compensate(beneficiary, collected);
    }
    ...
}
```

## Rationale

This enhancement aims to unlock new capabilities and scenarios in DeFi, DApps, and beyond by enabling post-execution bundle validation. This feature is particularly beneficial in complex transaction sequences where one operation's outcome may affect subsequent ones' within the same transaction bundle. By allowing for validation after the execution of all bundled operations, this proposal accommodates the requirement for a generic, post-bundle execution validation mechanism that can address an array of conditions, including but not limited to the enforcement of transactional invariants and more sophisticated transaction constructs such as Intents fulfillment, and enhances the bundle's security guarantees.

Scenario Examples

DeFi Sequences 
A user performs a token swap followed by liquidity provision and staking in one transaction. Post-execution validation ensures that the entire sequence meets the user's conditions based on the final state, such as minimum received tokens.

Blockchain-based Games
In-game transactions that depend on specific game states can be validated post-execution. Such validation ensures that actions like trades or character upgrades only proceed if prior operations result in the expected state.

Social Recovery
Validates the completion of a social recovery process based on approvals from designated guardians, ensuring the integrity of the recovery operation post-execution.

Alternative design considerations

During the design phase, we considered the existing `postOp` function utilized by Paymasters as an alternative mechanism for post-operation validation. As defined within the ERC-4337 standard, Paymaster's approach facilitates gas payment delegation and handles validation immediately after executing each user operation. While functionally significant, this approach cannot inherently assess the state changes produced by the entirety of a user operation bundle. Moreover, the semantics and primary intent of the Paymaster's post-operation validation are focused on the financial aspect of operations.

## Backwards Compatibility

Since each smart account is inherently bound to a specific EntryPoint contract, the ERC's updates do not impose disruptive changes on existing accounts. Smart accounts that do not implement the new post-execution validation method will continue to operate as before without altering to their transactional behavior or interaction with the EntryPoint. This design choice is pivotal in ensuring that the integration of the new validation mechanism is non-intrusive and preserves the integrity of existing smart account deployments.

## Security Considerations

Security Benefits
Incorporating a post-execution validation mechanism into ERC-4337 brings challenges and significant security benefits by enabling more thorough state validations post-transaction execution. This supplemental validation approach allows contracts to confirm that their post-conditions are met only after the entire transaction bundle has been executed, enhancing security and reliability.

Enhanced State Integrity Verification
This mechanism ensures that complex operations involving multiple interdependent transactions can be validated against the final state. It guards against unforeseen changes in state that might occur during transaction execution, offering a more robust security model.

Mitigation of State Reentrancy Vulnerabilities
Validating userOps post-bundle execution, the mechanism reduces the risk of reentrancy attacks, which are common in scenarios where multiple transactions interact. This is particularly beneficial in complex DeFi interactions, where the final state validation confirms no undesirable state changes have occurred.

Security Risks
A concern is ensuring the system's robustness against both Denial of Service (DoS) and griefing attacks, particularly those exacerbated by the potential for increased gas consumption.

This section incorporates several mitigation strategies inspired by the original ERC-4337 specification, tailored to address the unique challenges posed by post-execution validation:

Increased Gas Consumption
The introduction of post-execution validation inherently increases the computational and gas overhead associated with processing user operation bundles. This heightened consumption poses risks of network congestion and opens avenues for malicious entities to exploit these mechanisms, intentionally crafting operations that fail validation to waste resources or complicate the validation process for legitimate operations within the same bundle.

Addressing Increased Gas Consumption
While the addition of post-execution validation incurs extra gas costs, its implementation can be optimized to mitigate impact:

Stake-Based Throttling and Banning
Analogous to ERC-4337's approach to mitigating DoS attacks through economic disincentives, entities causing the invalidation of multiple userOps due to failed post-execution validation are subject to throttling or temporal banning. This mechanism is enforced through a required stake, making the cost of launching such attacks prohibitively high.

Pure Validation Logic
Similar to the `validateUserOp` before the bundle execution, the `validatePostExecution` method restricts the use of environment-dependent opcodes, ensuring deterministic validation outcomes. To accommodate the need to assess post-execution state conditions, contracts can design their operations and validation logic to indirectly consider the necessary environmental conditions. For example, a contract could record relevant environmental information during operation execution (such as the current block number) in its storage, which could then be referenced during post-execution validation.

Predefined Gas Limits
Strict gas limits for post-execution validation prevent exploitation through excessive gas consumption, ensuring the network remains resilient against potential DoS attacks.

Selective Validation
Implementers can design their validation logic to execute conditionally based on the specific requirements of the user operation. Similar to the `executeUserOp` 4-byte selector within the calldata, the 4-byte selector of the `validatePostExecution` within calldata signals the EntryPoint to call it. This approach ensures that the additional validation gas is only consumed when necessary rather than as a blanket requirement for all operations.

Mitigation of DoS Risks
The potential for increased gas consumption to be exploited for DoS attacks is addressed through:

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
