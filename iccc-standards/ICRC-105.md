|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|105|Wasm History Management and Transaction Blocks|Austin Fatheree (@skilesare)|https://github.com/dfinity/ICRC/issues/105|Draft|Standards Track||2025-03-18|

# ICRC-105 Installation and Configuration History Block Definitions - Extends ICRC-3

## 1. Scope

The ICRC-105 specification extends ICRC-3 by defining the block types and associated fields for recording critical events in canister installation and configuration history. It is intended to ensure a deterministic recording of configuration changes and deployments, along with proper auditability of recorded events.

### Importance of Deterministic Event Recording

The ICRC-105 specification was designed to address the critical need for deterministic event recording in canister management. By capturing detailed logs of configuration changes and Wasm module installations, it ensures that every significant event in a canister's lifecycle is auditable and secure. This level of transparency not only strengthens trust in the system but also aligns with broader goals of maintaining integrity and accountability in decentralized environments. The structured approach to recording these events provides developers and administrators with the tools necessary to verify and analyze historical changes, fostering a robust and reliable operational framework.

## 2. Terminology and Definitions

- **Wasm Module:**  
  A binary executable module deployed as part of a canister.

- **ICRC-3 Block:**  
  A log entry recorded on the ICRC-3 ledger capturing metadata associated with operations (e.g., upgrades, rollbacks).

- **Configuration:**  
  A method used to manage administrative state of a canister and/or change the behavior of a canister that is not defined in other ICRC standards. These changes should be recorded if they would change the behavior, permissions, execution of standard functions, or general operation of the canister.

## 3. Block Types

ICRC-105 blocks consist of the following top level fields as defined in ICRC-3

- **btype** - Identifies the block type
- **ts** - Timestamp of the block.
- **tx** - Transaction details. See below for definitions.

### 3-1. 105config

This block records the execution of functions that alter the configuration state of a canister. It provides a detailed log entry for each configuration change that occurs.

**tx Fields:**
- **caller**: Principal
  - The unique identifier of the caller invoking the configuration change. Principals are represented as a Blob of the raw principal.
- **method**: Text
  - The function name that was executed to modify configuration.
- **argsHash**: Blob
  - The representational independent hash of the args provided to the method.  The representational independent hash is defined in ICRC16.
- **args**: ICRC16Map
  - The complete set of parameters in the form (argName as Text, value as ICRC16).

DECISION POINT: Up for debate: The alternative here is to just use blob and hash of the blob, but then the decision becomes if that should be CBOR encoded or not, an if so, do we need another place for arg names?  The use of ICRC16 is a bit more EXPLICIT and easier to read from a transaction log. Since transparency is the goal here we've opted for ICRC16.  See also args for 105wasm_installed.

**Usage:**
Each configuration function call should be logged with a complete set of these fields to allow for post-mortem analysis and verification of the configuration history.

### 3-2. 105install

This block is used to record the successful installation of a Wasm module on a canister. It ensures there is an auditable log of each installation event.

**Fields:**
- **caller**: Principal
  - The unique identifier of the caller invoking the installation. Principals are represented as a Blob of the raw principal.
- **canisterId**: Principal
  - The unique identifier of the canister at the time of installation. Principals are represented as a Blob of the raw principal.
- **subnet**: Principal
  - The unique identifier of the subnet at the time of installation. Principals are represented as a Blob of the raw principal.
- **mode**: text - install or upgrade or migration.
- **argsHash**: optional Blob
  - The representational independent hash of the args provided to the method.  The representational independent hash is defined in ICRC16.
- **args**: Optional ICRC16 Map
  - The complete set of parameters in the form (argName as string, value as ICRC16).

**Usage:**
The entry should be created immediately after a Wasm installation function call to guarantee that the event is logged reliably. Subnet and wasm hash installed should be verified with system canister calls.

## 4. Implementation Considerations

- **Event Ordering:** Ensure that entries are recorded in the exact order of operations to maintain a reliable historical ledger.  For new ledgers, and where possible, the first record in the ICRC-3 log SHOULD BE the initial wasm installation.  Other events should not occur until after this block has been recorded with confirmed wasm hashs, canisterId, and subnet information.  For in flight, ledgers it should be appended to the log in the next upgrade.

- **Replica Events:** Because changes to the canister at the replica level do not currently produce a lifecycle event inside the canister, the canister should periodically capture any changes to these items and record them in a timely manner. They should be recorded as a 105config configuration changes with the method name "sys:replica".

The following arg names can be used to record detected config changes.
  - **configs**: Map
    - A map of configuration key-value pairs.
      - **sys:controllers**: Optional - Array of Principal
        - The list of controllers of the canister. Principals are represented as Blobs of the raw principal.
      - **sys:compute_allocation**: Optional - Nat
        - Amount of compute that the canister has allocated.
      - **sys:memory_allocation**: Optional - Nat
        - Amount of memory (storage) that the canister has allocated.
      - **sys:freezing_threshold**: Optional - Nat
        - A safety threshold to prevent the canister from running out of cycles and avoid being deleted.
      - **sys:reserved_cycles_limit**: Optional - Nat
        - A safety threshold to protect against spending too many cycles for resource reservation.
      - **sys:wasm_memory_limit**: Optional - Nat
        - A safety threshold to protect against reaching the 4GiB hard limit of 32-bit Wasm memory.
      - **sys:log_visibility**: Optional - Text
        - `controllers` or `public`. Controls who can read the canister logs.

## 5. References

### ICRC-3

An implementation of ICRC-105 Registry MUST also support the methods and standards defined in [ICRC-3](https://github.com/dfinity/ICRC-1/tree/main/standards/ICRC-3)

### ICRC-10

An implementation of ICRC-105 Registry MUST also support the method `icrc10_supported_standards` (per [ICRC-10](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-10/ICRC-10.md)), and include the following entries in its response:

```candid
  record { name = "ICRC-3"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-3"; }
  record { name = "ICRC-10"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-10"; }
  record { name = "ICRC-105"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-105"; }
```

## 6. Conclusion

The ICRC-105 protocol is a foundation for reliable canister configuration management and history logging. By adhering to the detailed block definitions and recording requirements outlined in this document, developers can ensure that each canister's state is fully auditable and traceable, providing enhanced security and operational transparency.

This document provides the essential definitions for recording configuration changes and historical block events for canister deployments.
