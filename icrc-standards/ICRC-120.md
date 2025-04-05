
|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|120|Canister Wasm Orchestration Service Specification|Austin Fatheree (@skilesare)|https://github.com/dfinity/ICRC/issues/120|Draft|Standards Track||2025-03-18|

# ICRC-120: Canister Wasm Orchestration Service

## 1. Scope

ICRC-120 establishes a standard interface for orchestrating WebAssembly (wasm) module installs, upgrades and rollbacks on the Internet Computer. This specification details the functions and protocols that ensure smooth transitions during canister upgrades, consistent state management. With clear procedures, the orchestration outlined here may be supported by many ecosystem tools to provide stability and consistency for managed upgrades.

The ICRC-120 specification covers:

- **Wasm Install and Upgrade Initiation:**  
  Defining how upgrades are initiated, including pre-upgrade validations and state transfers.

- **Upgrade Finalization:**  
  Procedures to validate and conclude the upgrade, ensuring that the upgraded canister is operating correctly.

- **Rollback Mechanisms:**  
  Providing a safe fallback through snapshot-based rollbacks when upgrades fail or deviate from expected behavior.

- **Canister Configurations:**  
  Providing a method for configuring canisters and their settings.

- **Canister Life Cycle:**
  Methods for starting and stopping canisters.

### Out of Scope

ICRC-3 blocks for the Canister wasm Orchestration Service are defined in an extension to the ICRC in ICRC-121

---

## 2. Purpose and Reasoning

Many tools currently exist to manage canisters on the internet computer. These range from elaborate workflows in the SNS Suite of canisters to the `dfx` command line.  This standard exists to support the proliferation of those toolsets by providing a standard interface through which the various tools can interact with the canister lifecycle of managed canisters. Canister management may be required by DAOs, decentralized utilities, or simple users programming scripts to deploy their projects.

The specific use cases considered when developing this standard are described below. Use cases may exist beyond these and this ICRC standard may be extended as those use cases become clear.

- **DAO Governed Canisters** - Non-SNS DAOs currently have to roll their own canister management solution. By deploying a pre-rolled canister management service a DAO gets instant-on canister management as long as that DAO can call function on the internet computer.
- **Package Managers and Wasm registries** - Package manager dapps can implement the canister management service and deploy 'packages' as canisters, giving users the ability to rapidly deploy experimental, test, and production canisters.
- **Wallet Canisters** - Wallet canisters implementing the canister management service can easily deploy personalized canisters across a wide array of application on the internet computer.
- **SNS Governance Extension** - While the SNS provides pathways for both "official" canister upgrades and custom dapp canister upgrades, the only method of initiating a canister upgrade is via governance of the entire DAO under the standard governance process.  By deploying a Canister Wasm Orchestration Service as an SNS application canister, an SNS DAO can manage versions and DAO canisters with governance **and** code.  The extensible service might deploy new canisters on a schedule or by approval of a subset of elected executives.
- **Cycle and Payment Abstraction** - Canister management services are able to abstract away the management of cycles for users and provide different economics and incentivization mechanisms for supporting deployed canisters.

---

## 3. Terminology and Definitions

- **Wasm Module:**  
  A binary executable module deployed as part of a canister.

- **Canister Upgrade:**  
  The process of replacing a canister's executable code with a new version while preserving state.

- **Snapshot:**  
  A stored state of a canister taken prior to a code upgrade, used for reverting in case of upgrade failures.

- **Orchestration Function:**  
  A method used to manage and coordinate the lifecycle of a canister upgrade.

---

## 4. Service Function Specifications

### 4.1. icrc120_upgrade_to

**Purpose:**  
Initiate the canister upgrade process and provide necessary parameters.

**Function Signature**
  
  ```candid
  type UpgradeToRequest = record {
      canister_id: principal;
      hash: blob;
      args: blob;
      stop: bool;
      snapshot: bool;
      timeout: nat;
      parameters: opt vec record{text; ICRC16};
  };

  type UpgradeToRequestId: nat;

  type UpgradeToResult = variant{
    Ok: UpgradeToRequestId; // transaction record and/or identifier of the upgrade request
    Err: variant {
      Unauthorized;
      Generic: text;
      WasmUnavailable;
      InvalidPayment;
    }
  };

  icrc120_upgrade_to : vec UpgradeToRequest -> (vec UpgradeToResult);
  ```

**Parameters:**
- **UpgradeToRequest.canister_id**: principal - Principal identifier of the canister to be upgraded.
- **UpgradeToRequest.hash**: blob - SHA-256 hash of the target Wasm module.
- **UpgradeToRequest.args**: blob - Serialized arguments required for the upgrade.
- **UpgradeToRequest.stop**: opt bool - Boolean flag indicating whether to stop the canister before upgrading.
- **UpgradeToRequest.snapshot**: Boolean flag to create a snapshot before initiating the upgrade.
- **UpgradeToRequest.timeout**: Duration (in nanoseconds) to wait before declaring the upgrade as failed.
- **UpgradeToRequest.parameters**: optional vec record{text; ICRC16} - an optional vector of hash, parameter pairs to pass to the canister at each step of the upgrade chain.
  - The following are known config parameters that may be present, but due to the difference between instances and the evolving nature of the Internet Computer, these items may change, be removed, or be added overtime. See the Annotations section below. Other items that may be included include items for canister or cycle payment and/or other service integrations. Additional items should be prefixed with a namespace, typically an ICRC standard.
  - **sys:controllers**: Optional - Array of Principal
    - The list of controllers of the canister. Principals are represented as Blobs of the raw principal
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

**Operation:**
1. **Authorize**: Confirm the rights of the caller.
2 **Lookup**: Confirm the wasm has corresponding to `hash`.
3. **Payment Confirmation**: optional - If the instance requires payment, confirm the payment.

4. **Schedule**:  Schedule the upgrade.
5. **Logging**: optional - Record the request in the ICRC-3 ledger (block type `"121upgrade_to"`).
6. **Return**: return the id of the request
7. **Stop Canister**: optional - Stop the canister if requested.
8. **Snapshot**: optional - Snapshot the canister if requested.
9. **Logging**: optional - Record the snapshot in the ICRC-3 ledger (block type `"121snapshot_finished"`).
10. **Attempt**: Attempt the install/upgrade
11. **Check Status**: Poll the icrc121_upgrade_finished endpoint for upgrade completion.
12. **Record Result**: Record the result for the user
13. **Revert If Necessary**: optional - if the user requests a snapshot and the upgrade fails, the system should attempt to revert to the snapshot.
14. **Logging**: optional - Record the request in the ICRC-3 ledger (block type `"121upgrade_finished"`).
15. **Clean Up**: optional - Clean up the snapshot by calling `icrc121_clean_snapshot`.
16. **Logging**: optional - Record the clean up in the ICRC-3 ledger (block type `"121clean_snapshot"`).

---

### 4.2. icrc120_create_snapshot

**Purpose:**  
Provides a mechanism to create a snapshot for a canister if a back up of the canister is needed. This may be called by a canister that has performed the installation or a user with rights to manage a canister on a particular wasm orchestration service.

```candid
  type CreateSnapshotRequest = record {
    canister_id: text;
    restart: bool;
  };

  type CreateSnapshotResult = variant {
    Ok: nat;
    Error: variant {
      Unauthorized;
      NotFound;
      Generic: text;
    };
  };

  icrc120_create_snapshot: (vec CreateSnapshotRequest) -> (vec CreateSnapshotResult);
```

**Parameters:**
- **CreateSnapshotRequest.canister_id**: principal - Principal identifier of the canister to create a snapshot for.
- **CreateSnapshotRequest.restart**: bool - indicate if the canister should be restarted after the snapshot.

**Operation:**
1. **Authorize**: Confirm the rights of the caller.
1. **Lookup**: Confirm the canister corresponding to `canister_id`.
3. **Stop**: Stop the canister(required for snapshots at this time)
2. **Snapshot**: Create the snapshot.
5. **Start**: optional Restart the canister
3. **Logging**: optional - Record the snap event in the ICRC-3 ledger (block type `"121finished_snapshot"`).

---

### 4.3. icrc120_clean_snapshot

**Purpose:**  
Provides a mechanism to clean a requested snapshot from the replica once the installer is satisfied that it is no longer needed. This may be called by a canister that has performed the installation or a user with rights to manage a canister on a particular wasm orchestration service.

```candid
  type CleanSnapshotRequest = record {
    canister_id: text;
    snapshot_id: nat; 
  };

  type CleanSnapshotResult = variant {
    Ok: nat;       // Transaction record.
    Error: variant {
      Unauthorized;
      NotFound;
      Generic: text;
    };
  };

  icrc120_clean_snapshot: (vec CleanSnapshotRequest) -> (vec CleanSnapshotResult);
```

**Parameters:**
- **CleanSnapshotRequest.canister_id**: principal - Principal identifier of the canister to be upgraded.
- **CleanSnapshotRequest.snapshot_id**: nat - id of the snapshot.


**Operation:**
1. **Authorize**: Confirm the rights of the caller.
1. **Lookup**: Confirm the snapshot corresponding to `snapshot_id`.
2. **Deletion**: Delete the stored snapshot.
3. **Logging**: optional - Record the rollback event in the ICRC-3 ledger (block type `"121clean_snapshot"`).

---

### 4.4. icrc120_revert_snapshot

**Purpose:**  
Provides a mechanism to revert to a snapshot from the replica.


```candid
  type RevertSnapshotRequest = record {
    canister_id: text;
    snapshot_id: nat; 
    restart: bool;
  };

  type RevertSnapshotResult = variant {
    Ok: nat;       // Transaction record.
    Error: variant {
      Unauthorized;
      NotFound;
      Generic: text;
    };
  };

  icrc120_revert_snapshot: (vec RevertSnapshotRequest) -> (vec RevertSnapshotResult);
```

**Parameters:**
- **RevertSnapshotRequest.canister_id**: principal - Principal identifier of the canister to be reverted.
- **RevertSnapshotRequest.snapshot_id**: nat - id of the snapshot to revert to.
- **RevertSnapshotRequest.restart: bool - indicates if the process should start the canister after the revert if successful.


**Operation:**
1. **Authorize**: Confirm the rights of the caller.
2. **Lookup**: Confirm the snapshot corresponding to `snapshot_id`.
3. **Logging**: optional - Record the revert request event in the ICRC-3 ledger (block type `"121revert_snapshot"`).
4. **Return**: Return the record id of the revert request
5. **Stop**: Stop a canister if not already stopped.
6. **Revert**: Revert to the stored snapshot.
7. **Restart**: optional - Restart if the ser requested it.
3. **Logging**: optional - Record the rollback event in the ICRC-3 ledger (block type `"121revert_result"`).

---

### 4.4. icrc120_start_canister

**Purpose:**  
Provides a mechanism to start canister.


```candid

  type StartCanisterRequest = record{
    canister_id: principal;
    timeout: nat;
  };

  type StartCanisterResult = variant {
    Ok: nat;       // Transaction record.
    Error: variant {
      Unauthorized;
      NotFound;
      Generic: text;
    };
  };

  icrc120_start_canister: (vec principal) -> (vec StartCanisterResult);
```

**Parameters:**
- **startCanisterRequest.canister_id**: principal - Principal identifier of the canister to be started.
- **StartCanisterRequest.timeout**: nat - Number of nano seconds to wait for the canister to start before timing out.


**Operation:**
1. **Authorize**: Confirm the rights of the caller.
2. **Lookup**: Confirm the canister corresponding to `canister_id`.
3. **Start**: Attempt to start the canister
3. **Logging**: optional - Record the revert request event in the ICRC-3 ledger (block type `"121start_canister"`).
4. **Return**: Return the record id of the start request

---


### 4.5. icrc120_stop_canister

**Purpose:**  
Provides a mechanism to stop a canister.

```candid
  type StopCanisterRequest = record{
    canister_id: principal;
    timeout: nat;
  };

  type StopCanisterResult = variant {
    Ok: nat;       // Transaction record.
    Error: variant {
      Unauthorized;
      NotFound;
      Generic: text;
    };
  };

  icrc120_stop_canister: (vec StopCanisterRequest) -> (vec StopCanisterResult);
```

**Parameters:**
- **StopCanisterRequest.canister_id**: principal - Principal identifier of the canister to be stopped.
- **StopCanisterRequest.timeout**: nat - Number of nanoseconds to wait for the canister to stop before timing out.

**Operation:**
1. **Authorize**: Confirm the rights of the caller.
2. **Lookup**: Confirm the canister corresponding to `canister_id`.
3. **Stop**: Attempt to stop the canister.
4. **Logging**: optional - Record the stop request event in the ICRC-3 ledger (block type `"121stop_canister"`).
5. **Return**: Return the record id of the stop request.

---

### 4.6. icrc120_config_canister

**Purpose:**  
Allows a user to define new configurations for a canister.

**Function Signature**

```candid
type ConfigCanisterRequest = record {
  canister_id: principal;
  configs: vec record { text; ICRC16 };
};

type ConfigCanisterResult = variant {
  Ok: nat; // Transaction record and/or identifier of the config request
  Err: variant {
    Unauthorized;
    InvalidConfig: text;
    Generic: text;
  }
};

icrc120_config_canister: (vec ConfigCanisterRequest) -> (vec ConfigCanisterResult);
```

**Parameters:**
- **ConfigCanisterRequest.canister_id**: principal - Principal identifier of the canister to be configured.
- **ConfigCanisterRequest.configs**: vec record { text; ICRC16 } - A vector of configuration key-value pairs.

**Operation:**
1. **Authorize**: Confirm the rights of the caller.
2. **Validate**: Validate the provided configurations.
3. **Apply**: Apply the configurations to the canister.
4. **Logging**: optional - Record the configuration request in the ICRC-3 ledger (block type `"121config_canister"`).
5. **Return**: Return the id of the configuration request.

---

### 4.7 icrc120_get_events - query

**Purpose:**  
Provides a mechanism to query the history of a canister.

**Note:**
As of now, a robust generic index canister for ICRC-3 logs does not exist. As a result we are providing this function, but would suggest that it be deprecated once one does. Currently this would only be able to return on-canister data and would disregard archives and history. It also requires the user to maintain a number of indexes that could complicated code.

```candid
type OrchestrationEventType = variant {
  #upgrade_initiated;
  #upgrade_finished;
  #snapshot_created;
  #snapshot_cleaned;
  #snapshot_reverted;
  #canister_started;
  #canister_stopped;
  #configuration_changed;
};

type OrchestrationEvent = record {
  event_type: OrchestrationEventType;
  canister_id: principal;
  details: ICRC16;     // Block containing data.
};

type GetEventsFilter = record {
  canister: opt principal;
  event_types: opt vec OrchestrationEventType;
  start_time: opt nat;   // Only events after this timestamp
  end_time: opt nat;     // Only events before this timestamp
};

icrc120_get_events: (record {
  filter: opt GetEventsFilter;
  prev: opt blob;
  take: opt nat;
}) -> (vec OrchestrationEvent) query;
```

### 4.8 icrc120_metadata - query

See 5.2

---


## 5. Target and General Functions

The Wasm Orchestration service queries its target canisters after an upgrade to inquire if an upgrade has been completed successfully or not.  Canister authors should provide the following endpoints so that the service can determine if the upgrade has completed yet.


### 5.1 - icrc120_upgrade_finished

**Purpose:**  
Mark the successful (or unsuccessful) completion of the upgrade process and finalize canister state.

```candid


type UpgradeFinishedResult = variant {
  #InProgress: nat;
  #Failed: text;
  #Success: nat;
};

icrc120_upgrade_finished: query () -> query UpgradeFinishedResult;
```

**Return:**
- `InProgress` - The upgrade is underway. The nat should the timestamp the upgrade started
- `Failed` - The upgrade failed and the text explains why.
- `Success` - The upgrade is succeeded and the timestamp provides the time it finished.


### 5.2 - icrc120_metadata

`icrc120_metadata` *Returns information about the instance.*

The following metadata fields are defined, but may be expanded by the instance buy using ICRC standards for metadata naming.

- `icrc120:canister_type` - Text or Array - "orchestrator", "target". Indicates what role this canister plays. If only one key is present then the expected functions should be present as defined: orchestrator - Section 4, target - Section 5. It is possible for a canister to be both if an orchestrator is itself managed by another orchestrator.

---

## 6. References

### ICRC-10

An implementation of ICRC-118 Registry MUST also support the method `icrc10_supported_standards` (per [ICRC-10](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-10/ICRC-10.md)), and include the following entries in its response:

```candid
  record { name = "ICRC-10"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-10"; }
  record { name = "ICRC-129"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-120"; }
```

### ICRC-16

This standard utilizes generic data structures defined in [ICRC-16](https://github.com/dfinity/ICRC/issues/16).

### ICRC-105 - Wasm History Management and Transaction Blocks

It is recommended that systems implementing ICRC-120 limit wasms support to wasms and applications that support [ICRC-105](https://github.com/dfinity/ICRC/issues/105) blocks. These blocks help provide assurance that wasm upgrade pathways have been followed and the chain of trust offered by the Internet Computer technology has not been violated.

### ICRC-121 - Wasm Orchestration Block Specification

It is recommended that systems implementing ICRC-120 support [ICRC-121](https://github.com/dfinity/ICRC/issues/121) blocks. These blocks help provide assurance that the wasm registry has integrity and that the upgrades canisters controlled by the orchestration service have not been manipulated by third parties. To take advantage of this, orchestration services SHOULD be the only controller of the canister.

### ICRC-126 - Wasm verification

It is recommended that systems implementing ICRC-120 utilize [ICRC-126](https://github.com/dfinity/ICRC/issues/126) to provide a mechanism for validators to endorse the build and hash of wasm versions that are installed.

## 7. Security Considerations

It is strongly recommended that implementations review the Security Considerations put forward in [ICRC-7 - Security Considerations](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-7/ICRC-7.md#security-considerations) and apply the same care to the implementation of ICRC-120.


## 8. Annotations

Reserved for future updates to configuration parameter changes.
