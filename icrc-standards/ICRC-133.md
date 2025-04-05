|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|133|Generic Input Capture and State Change - Extends ICRC-3|Austin Fatheree (@skilesare)|https://github.com/dfinity/ICRC/issues/105|Draft|Standards Track||2025-03-18|

# ICRC-133 Generic Input Capture and State Change - Extends ICRC-3

## 1. Scope

The ICRC-133 specification extends ICRC-3 by defining the block types and associated fields for recording generic input calls and state changes. It is intended to ensure a deterministic recording of user calls to a smart contract(canister) on the Internet Computer and the effects that call has on state in order to support proper auditability and transparency of a canister's evolution.

## 2. Terminology and Definitions

- **Wasm Module:**  
  A binary executable module deployed as part of a canister.

- **ICRC-3 Block:**  
  A log entry recorded on the ICRC-3 ledger capturing metadata associated with operations (e.g., upgrades, rollbacks).

- **User Input:**  
  An update method call to manage a smart contract, typically to produce some state change.

- **State Change:**  
  Data inside a canister changes such that the behavior of or data reported by the canister changes for future users.

## 3. Block Types

ICRC-133 blocks consist of the following top level fields as defined in ICRC-3

- **btype** - Identifies the block type
- **ts** - Timestamp of the block.
- **tx** - Transaction details. See below for definitions.

### 3-1. 133call

This block records user input. It is intend to be implemented by any state-changing method in a canister. 

**tx Fields:**
- **caller**: Principal
  - The unique identifier of the caller invoking the configuration change. Principals are represented as a Blob of the raw principal.
- **method**: Text
  - The function name that was executed to modify configuration.
- **cycles**: optional Nat
  - The number of cycles provided in the call.
- **argsHash**: optional Blob
  - The representational independent hash of the args provided to the method.  The representational independent hash is defined in ICRC16.
- **args**: optional ICRC16Map
  - The complete set of parameters in the form (argName as string, value as ICRC16).

DECISION POINT: Up for debate: The alternative here is to just use blob and hash of the blob, but then the decision becomes if that should be CBOR encoded or not, an if so, do we need another place for arg names?  The use of ICRC16 is a bit more EXPLICIT and easier to read from a transaction log. Since transparency is the goal here we've opted for ICRC16.  See also args for 105wasm_installed.

**Usage:**
Each method function call should be logged with a complete set of these fields to allow for post-mortem analysis and verification of the canister's usage history.

### 3-2. 133state

This block is used to record generic updates to state. It is the canister developer's responsibility to ensure that persisted state changes are recorded.

**Fields:**
- **caller**: Principal
  - The unique identifier of the caller invoking the installation. Principals are represented as a Blob of the raw principal.
- **canisterId**: Principal
  - The unique identifier of the canister at the time of installation. Principals are represented as a Blob of the raw principal.
- **callId**: Nat
  - A unique call id assigned to this particular call.
- **nonce**: Nat
  - Starts with 0 and increments with each await call encountered during the processing of the function.
- **mode**: text - install or upgrade or migration.
- **changes**: Array [Mode, optional Change]
  - **mode**: Text: `null`, `set`
  - **change**: Array [Text, optional ICRC16]
    - If the mode is `null` the first parameter should be the fully qualified, unique name of the variable set to null. No second parameter is expected.
    - If the mode is `set` first parameter should be the fully qualified, unique name of the variable and the second should be the value the variable is set to.
    - Fully qualified names should be from the root of the canister, ie setting a balance on a variable `state` that has a collection of balances in `balances`: #Array([#Text("set"), #Array(["state.balances[\"706d348819f5b7316d497da5c9b3aae2816ffebe01943827f6e66839fcb64641\"]", #Nat(1000)])])

Note: Should we try to log throws or return values here?  I don't think they are necessary.

**Usage:**

The entry should be created and all state changes are known:
  - before each await call
  - after an update call has completed running

## 5. References

### ICRC-3

An implementation of ICRC-133 Registry MUST also support the methods and standards defined in [ICRC-3](https://github.com/dfinity/ICRC-1/tree/main/standards/ICRC-3)

### ICRC-10

An implementation of ICRC-133 Registry MUST also support the method `icrc10_supported_standards` (per [ICRC-10](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-10/ICRC-10.md)), and include the following entries in its response:

```candid
  record { name = "ICRC-3"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-3"; }
  record { name = "ICRC-10"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-10"; }
  record { name = "ICRC-133"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-133"; }
```
