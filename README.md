---
erc: XXXX
title: Smart Contract Provenance
description: A standard method for smart contracts to declare their source code location and version
author: Martin Monperrus @monperrus
discussions-to: <https://ethereum-magicians.org/TODO>
status: Draft
type: Standards Track
category: ERC
created: 2025-10-31
---

## Abstract

This ERC proposes a standard interface for smart contracts to declare their source code provenance by providing a repository URL and commit hash. This enables transparent verification of deployed contract code against its source, improving transparency, trust and auditability in the Ethereum ecosystem.

## Motivation

Currently, there is no standardized way for smart contracts to reference their canonical source code. 
The existing approaches have serious limitations:

**Solidity Metadata and IPFS** The Solidity compiler embeds metadata at the end of deployed bytecode, which includes an IPFS hash of the contract metadata JSON. However, this approach has several limitations:

- **Not Human-Readable**: The IPFS hash is embedded in bytecode and not easily discoverable without specialized tools
- **No Git Integration**: IPFS metadata doesn't reference specific commits in version control systems, making it difficult to track changes over time
- **Limited Adoption**: Many contracts are deployed without uploading metadata to IPFS
- **No Repository Context**: Even when available, IPFS metadata doesn't indicate the canonical repository, organizational ownership, or development history
- **Compiler-Specific**: Only works with Solidity and only when metadata is enabled during compilation

**Manual Verification Services** Services like Etherscan or Sourcify allow developers to manually submit source code for verification:

- **Off-Chain Process**: Verification happens outside the contract and depends on external platforms
- **Platform Dependency**: Users must trust and have access to specific verification services
- **Delayed or Missing**: Many contracts are never verified, or verification happens long after deployment


A standardized on-chain provenance method would:

- Enable automated verification of deployed contracts against their source repositories
- Provide discoverable, on-chain references to canonical source code
- Improve transparency by making source code locations queryable via standard interfaces
- Support reproducible builds by pinning to specific commit hashes
- Facilitate security audits by providing immutable references to exact source versions
- Allow contracts to self-document their origin in a verifiable way
- Integrate seamlessly with existing development workflows such as Git, without the need for external IPFS servers

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Interface

Compliant contracts MUST implement the following interface:

```solidity
interface IERC_Provenance {
    /**
     * @notice Returns the provenance information for this contract
     * @return repositoryUrl The URL of the source code repository that can be passed to `git clone`
     * @return commitHash The commit hash of the exact version deployed
     */
    function provenance() external view returns (
        string memory repositoryType,
        string memory repositoryUrl,
        string memory commitHash
    );
}
```

### Method Behavior

#### `repositoryType()`

- MUST return a string indicating the type of version control system used by the repository (e.g., "git" for Git repositories)
- The string MUST be lowercase and consist of alphanumeric characters only
- For Git repositories, it MUST be "git"
- For other systems, it SHOULD use a common identifier (e.g., "svn" for Subversion, "hg" for Mercurial)
- The function MUST be marked as `view` or `pure`
- The function MUST NOT revert under normal circumstances

#### `provenance()`

- MUST return a `repositoryUrl` that is a valid URL pointing to a source code repository, that can be directly used with the VCS system (eg `git clone`)
- MUST return a `commitHash` that is a valid commit identifier in the repository
- The `repositoryUrl` SHOULD use HTTPS protocol
- The `commitHash` MUST reference the exact version of the source code that was compiled and deployed
- The function MUST be marked as `view` or `pure`
- The function MUST NOT revert under normal circumstances

### Repository URL Format

The `repositoryUrl`:
- MUST be a complete URL including protocol (e.g., `https://github.com/org/repo`)
- SHOULD point to a publicly accessible repository
- MAY point to a specific branch or directory if needed, but this is NOT RECOMMENDED as the commit hash provides version specificity

### Commit Hash Format

The `commitHash`:
- MUST be a valid commit identifier for the version control system used by the repository
- For Git repositories, this MUST be the full 40-character SHA-1 hash or SHA-256 SHA
- MUST NOT use shortened commit hashes
- MUST NOT use branch names, tags, or other mutable references

## Rationale

### Single Method Design

A single method returning both URL and commit hash is chosen for simplicity and gas efficiency. This design:
- Minimizes the contract interface surface area
- Reduces the number of external calls needed for verification
- Ensures atomicity of the provenance information

### Immutable References

Requiring full commit hashes rather than branches or tags ensures that:
- The reference remains valid even if branches are deleted or tags are moved
- Verification is deterministic and cannot be altered after deployment
- Security audits reference an immutable code version

### String Returns

Using string types for both return values provides:
- Maximum flexibility for different repository systems
- Human readability when querying contracts
- Compatibility with various URL and commit hash formats

### View Function

The `provenance()` function is specified as `view` to:
- Ensure it can be called without gas costs
- Guarantee it doesn't modify contract state
- Enable easy integration with static analysis tools

## Backwards Compatibility

This ERC introduces a new interface and does not affect existing contracts. Contracts implementing this standard can coexist with contracts that do not, as the interface is purely additive.

Existing contracts MAY be upgraded to implement this interface if their upgrade mechanisms permit it.

## Reference Implementation

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.0;

contract ProvenanceExample {

    string private constant REPOSITORY_URL = "https://github.com/example/contract";
    string private constant COMMIT_HASH = "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0";
    
    function provenance() external pure returns (
        string memory repositoryType,
        string memory repositoryUrl,
        string memory commitHash
    ) {
        return ("git", REPOSITORY_URL, COMMIT_HASH);
    }
}
```

## Security Considerations

### Trust Assumptions

Implementing this standard does not guarantee:
- That the provided repository URL is accurate
- That the source code at the commit hash matches the deployed bytecode
- That the repository is trustworthy or secure

A malicious contract could provide:
- A URL to a malicious or phishing site
- A commit hash that doesn't correspond to the deployed code
- Misleading information to create false trust

Users and verification tools MUST NOT blindly trust provenance information without independent verification.
They MUST independently verify:
- The repository URL points to a legitimate source
- The source code at the specified commit compiles to the deployed bytecode
- The repository and commit history have not been tampered with

### Immutability

If the contract is upgradeable, implementers MUST ensure that the provenance information:
- References the correct commit for each deployed version
- Is updated appropriately when the contract is upgraded
- Maintains a history of provenance for all versions if possible

### Privacy Considerations

Projects requiring privacy SHOULD consider:
- Using private repositories with controlled access
- Providing provenance only to authorized parties
- Balancing transparency needs against privacy requirements

## Copyright

Copyright and related rights waived via [CC0](../LICENSE).
