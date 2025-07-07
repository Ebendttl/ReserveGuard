# ReserveGuard

A transparent on-chain proof-of-reserves system for stablecoins and wrapped assets, enabling real-time verification of asset backing and reserve ratios.

-----

## Table of Contents

  - Introduction
  - Features
  - Error Codes
  - Constants
  - Data Structures
  - Functions
      - Private Functions
      - Public Functions
      - Read-Only Functions
  - Deployment
  - Usage Examples
  - Contributing
  - License
  - Related Projects

-----

## Introduction

The `ReserveGuard` contract provides a robust and transparent framework for managing and verifying the reserves of stablecoins and wrapped assets on the blockchain. It aims to enhance trust and accountability by allowing anyone to verify the backing of an asset in real-time. This system is designed to prevent under-collateralization and ensure that issued assets are always adequately backed by their declared reserves.

-----

## Features

  * **Asset Registration**: Allows the contract owner to register new stablecoins or wrapped assets for reserve tracking.
  * **Reserve Deposits**: Enables designated depositors to add reserves to the system, increasing the backing for registered assets.
  * **Token Minting Control**: Integrates reserve ratio checks into the token minting process, preventing new tokens from being minted if it would lead to insufficient backing.
  * **Auditor Authorization**: Provides a mechanism for the contract owner to authorize trusted auditors to perform verifications.
  * **Comprehensive Audits**: Supports advanced audit functionality, allowing authorized auditors to submit detailed reserve proofs (e.g., using Merkle proofs) and update the on-chain reserve status.
  * **Real-time Reserve Ratio Verification**: Publicly exposes functions to check an asset's current reserve ratio and its compliance with the minimum required ratio.
  * **Audit Trail**: Maintains an immutable record of all comprehensive audits performed, including the auditor, reported reserves, supply, and calculated ratio.
  * **Emergency Pause (Planned)**: Includes a global emergency pause switch, allowing the contract owner to temporarily halt critical operations in case of unforeseen issues (though not fully implemented in the provided snippet for all functions).

-----

## Error Codes

The contract defines the following error codes to provide informative feedback on failed operations:

  * `u100`: `err-owner-only` - Operation can only be performed by the contract owner.
  * `u101`: `err-invalid-asset` - The specified asset ID is not valid or the asset is inactive.
  * `u102`: `err-insufficient-reserves` - The operation would lead to the asset being under-collateralized.
  * `u103`: `err-invalid-amount` - The specified amount is zero or otherwise invalid.
  * `u104`: `err-unauthorized-auditor` - The transaction sender is not an authorized auditor.
  * `u105`: `err-reserve-locked` - Operations are paused due to an emergency.
  * `u106`: `err-invalid-proof` - The provided Merkle proof is invalid (not explicitly used in the provided `perform-comprehensive-audit` beyond a conceptual check, but available).

-----

## Constants

  * `contract-owner`: The principal (address) that deployed the contract.
  * `min-reserve-ratio`: `u15000` (150% in basis points). Represents the minimum required reserve ratio for assets.
  * `basis-points`: `u10000` (100%). Used for percentage calculations, where 10,000 basis points equals 100%.

-----

## Data Structures

### Maps

  * `registered-assets`: Stores metadata for each registered asset.
      * Key: `{ asset-id: uint }`
      * Value: `{ asset-symbol: (string-ascii 10), backing-asset: (string-ascii 20), total-supply: uint, reserve-amount: uint, last-audit-block: uint, is-active: bool }`
  * `reserve-deposits`: Records individual reserve deposits.
      * Key: `{ asset-id: uint, depositor: principal }`
      * Value: `{ amount: uint, block-height: uint }`
  * `authorized-auditors`: Tracks authorized auditors and their audit activity.
      * Key: `{ auditor: principal }`
      * Value: `{ is-authorized: bool, audit-count: uint }`
  * `audit-records`: Stores a historical trail of comprehensive audits.
      * Key: `{ asset-id: uint, audit-id: uint }`
      * Value: `{ auditor: principal, reserve-amount: uint, supply-amount: uint, reserve-ratio: uint, block-height: uint, is-verified: bool }`

### Variables

  * `next-asset-id`: `(define-data-var next-asset-id uint u1)` - Counter for assigning new asset IDs.
  * `next-audit-id`: `(define-data-var next-audit-id uint u1)` - Counter for assigning new audit IDs.
  * `emergency-pause`: `(define-data-var emergency-pause bool false)` - A boolean flag to pause critical contract functions.

-----

## Functions

### Private Functions

  * `(calculate-reserve-ratio (reserves uint) (supply uint))`:
      * Calculates the reserve ratio in basis points.
      * Returns `u0` if `supply` is `u0`.
  * `(meets-reserve-requirements (asset-id uint))`:
      * Checks if a given asset currently meets the `min-reserve-ratio`.
      * Returns `true` if requirements are met, `false` otherwise.
  * `(update-reserve-amount (asset-id uint) (new-amount uint))`:
      * Updates the `reserve-amount` for a specified asset in the `registered-assets` map.
      * Returns `(ok true)` on success or `err-invalid-asset` if the asset is not found.
  * `(verify-merkle-step (proof-hash (buff 32)) (current-hash (buff 32)))`:
      * A helper function used in `perform-comprehensive-audit` for Merkle proof verification.
      * Computes the SHA256 hash of the concatenation of `current-hash` and `proof-hash`.

### Public Functions

  * `(register-asset (asset-symbol (string-ascii 10)) (backing-asset (string-ascii 20)) (initial-supply uint))`:
      * Registers a new asset for reserve tracking.
      * Callable only by `contract-owner`.
      * Requires `initial-supply` to be greater than `u0`.
      * Asserts `emergency-pause` is `false`.
      * Returns `(ok asset-id)` on success or an error code.
  * `(deposit-reserves (asset-id uint) (amount uint))`:
      * Records a reserve deposit for an asset.
      * Requires `amount` to be greater than `u0`.
      * Asserts `emergency-pause` is `false`.
      * Updates the `reserve-deposits` map and the `reserve-amount` in `registered-assets`.
      * Returns `(ok new-reserves)` on success or an error code.
  * `(mint-tokens (asset-id uint) (amount uint))`:
      * Increases the `total-supply` of an asset.
      * Callable only by `contract-owner`.
      * Requires `amount` to be greater than `u0`.
      * Asserts `emergency-pause` is `false`.
      * Crucially, it verifies that the asset still `meets-reserve-requirements` *after* the supply increase.
      * Returns `(ok new-supply)` on success or `err-insufficient-reserves` if the ratio falls below the minimum.
  * `(authorize-auditor (auditor principal))`:
      * Authorizes a `principal` to perform comprehensive audits.
      * Callable only by `contract-owner`.
      * Returns `(ok true)` on success.
  * `(perform-comprehensive-audit (asset-id uint) (reported-reserves uint) (merkle-root (buff 32)) (proof-hashes (list 10 (buff 32))))`:
      * Allows an authorized auditor to submit an external audit proof.
      * Verifies the `tx-sender` is an authorized auditor.
      * Checks if the asset is active.
      * *Conceptual*: It includes a `fold` operation using `verify-merkle-step` to process `proof-hashes` against a `merkle-root`, simulating a Merkle proof verification. **Note**: The actual Merkle proof validation logic (i.e., comparing `calculated-root` to `merkle-root` and using it for an `is-verified` status related to proof validity) is not fully implemented in the provided snippet for assertion, but the structure is present.
      * Records the audit in `audit-records`, including the calculated `reserve-ratio` and if it is compliant (`is-verified`).
      * Updates the `reserve-amount` and `last-audit-block` for the asset based on the `reported-reserves`.
      * Increments the `audit-count` for the auditor.
      * Returns a detailed audit result `(ok { ... })` on success or an error.

### Read-Only Functions

  * `(get-asset-info (asset-id uint))`:
      * Retrieves the full metadata for a registered asset.
      * Returns `(some { ... })` if found, `(none)` otherwise.
  * `(get-reserve-ratio (asset-id uint))`:
      * Calculates and returns the current reserve ratio for an asset.
      * Returns `(ok ratio)` or `err-invalid-asset`.
  * `(is-fully-backed (asset-id uint))`:
      * Checks if an asset currently meets its minimum reserve requirements.
      * Returns `(ok true)` or `(ok false)`.

-----

## Deployment

To deploy this contract, you'll need a Clarity-compatible blockchain environment (e.g., Stacks blockchain).

1.  **Compile**: Ensure your development environment can compile Clarity contracts.
2.  **Deploy**: Deploy the contract to your chosen network. The `contract-owner` will automatically be set to the address that initiates the deployment transaction.

-----

## Usage Examples

Here are some hypothetical examples of how to interact with the `ReserveGuard` contract:

1.  **Registering a new asset (by owner):**
    ```clarity
    (as-contract call-private 'reserve-guard.register-asset "USDCT" "USD" u1000000)
    ```
2.  **Depositing reserves:**
    ```clarity
    (as-contract call-private 'reserve-guard.deposit-reserves u1 u500000)
    ```
3.  **Minting new tokens (by owner):**
    ```clarity
    (as-contract call-private 'reserve-guard.mint-tokens u1 u200000)
    ```
4.  **Authorizing an auditor (by owner):**
    ```clarity
    (as-contract call-private 'reserve-guard.authorize-auditor 'ST1PQHQKV0RJXZFY1DGX8MNSNYVE3FG9QZ5J1S6PM.auditor-contract)
    ```
5.  **Performing a comprehensive audit (by authorized auditor):**
    ```clarlarity
    ;; Assuming reported-reserves is 1500000, and example Merkle root/proof hashes
    (as-contract call-private 'reserve-guard.perform-comprehensive-audit 
      u1 
      u1500000 
      0x1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b 
      (list 0x112233445566778899aabbccddeeff00112233445566778899aabbccddeeff00 
            0xaabbccddeeff00112233445566778899aabbccddeeff00112233445566778899)
    )
    ```
6.  **Getting asset information:**
    ```clarity
    (read-only 'reserve-guard.get-asset-info u1)
    ```
7.  **Checking reserve ratio:**
    ```clarity
    (read-only 'reserve-guard.get-reserve-ratio u1)
    ```
8.  **Checking if fully backed:**
    ```clarity
    (read-only 'reserve-guard.is-fully-backed u1)
    ```

-----

## Contributing

Contributions are welcome\! If you have suggestions for improvements or find any issues, please feel free to:

1.  Fork the repository.
2.  Create a new branch for your feature or bug fix.
3.  Implement your changes following Clarity best practices.
4.  Write comprehensive tests for your changes.
5.  Submit a pull request.

-----

## License

This project is licensed under the MIT License - see the LICENSE file for details.

-----

## Related Projects

  * Stacks.co: The blockchain on which this contract is designed to run.
  * Clarity-Lang: Official documentation for the Clarity smart contract language.

-----
