|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|118| Wasm Registry Definition for Version Control and History|Austin Fatheree (@skilesare)|https://github.com/dfinity/ICRC/issues/118|Draft|Standards Track||2025-03-18|

# ICRC-118: Wasm Registry Definition for Version Control and History

---

## 1. Scope

ICRC-118 defines a **Wasm Registry** standard for tracking WebAssembly (Wasm) modules on the Internet Computer (IC). 

The registry ensures:
- A **structured** approach to Wasm versioning and history tracking.
- Verifiable and immutable **audit trail** compliance via **ICRC-3 blocks**.
- Standardized search and query mechanisms for **Wasm deployments**.

ICRC-118 provides the following functionalities:

- **Wasm Registration & Management**  
  Adds, updates, retrieves, and deprecates Wasm modules based on their lifecycle.

- **Canister Type Search & History**  
  Fetches canisters by their type, controller, and upgrade path.

- **Hosts wasms**  
  The standard support the storing and retrieval of wasms. This is limited by canister and economic capacity of the implementation.

---

## 2. Function Specifications

Please review [ICRC7 - Generally-Applicable Specification](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-7/ICRC-7.md#generally-applicable-specification) for general details on batch update methods and batch query methods.

### 2.1 Core Wasm Functions

#### `icrc118_create_canister_type`

- **Purpose:** Create a new canister type with specified controllers.
- **Inputs:** 
  - `controllers`: Optional list of principals who will control the canister.
  - `canister_type_namespace`: Text identifier for the canister type. Namespace creators SHOULD keep in mind the global namespace of the IC and use unique names that they can/could provably control(see ICRC-86)
  - `canister_type_name` : Display name for the canister type.
  - `forked_from`: Optional record indicating the canister type and version it is forked from.
  - `canister_type_namespace`: 
- **Outputs:** 
  - `Ok`: Transaction record ID if successful.
  - `Error`: Variant indicating the type of error (Unauthorized, Generic).

```candid
type CreateCanisterType = record {
    controllers: opt vec principal;
    canister_type_namespace: text;
    canister_type_name;
    description: text;
    repo: text;
    metadata: ICRC16;
    forked_from: opt {
        canister_type_namespace: text;
        version: record {nat; nat; nat};
    }
}

// Result types for various operations.
type CreateCanisterTypeResult = variant {
  Ok: nat;       // Transaction record.
  Error: variant {
    Unauthorized;
    Generic: text;
  };
};

icrc118_create_canister_type: (vec CreateCanisterType) -> (vec CreateCanisterTypeResult); // Returns success message or error.
```

#### `icrc118_manage_controller`
- **Purpose**  Add or remove controllers on a canister type.
- **Inputs** Details of the canister you would like to update controllers for.
- **Outputs** Details of the results.

```candid

type ManageControllerRequest =record {
    canister_type: text;
    controller: Principal;
    op: variant {Add; Remove;};
};

type ManageControllerResult = variant {
  Ok: nat;       // Transaction record.
  Error: variant {
    Unauthorized;
    NotFound;
    Generic: text;
  };
};

icrc118_manage_controller: (vec ManageControllerRequest) -> (vec ManageControllerResult);
```

#### `icrc118_get_wasms`
- **Purpose:** Retrieve a list of registered Wasm modules.
- **Input:** Optional filters (e.g., hash, canister type, version range).
  - Multiple hash filters should be -or-ed- together
  - Multiple canister_type filters should be -or-ed- together
  - Multiple controller filters should be -or-ed- together
  - Multiple previous_has filters should be -or-ed- together
  - Multiple canister filters should be -or-ed- together
  - Dissimilar filters shold be -and-ed- together
- **Output:** Array of Wasm records with details (hash, description, timestamp, metadata).

```candid
type ICRC16Map = ICRC16Map     // Defined in ICRC-16

type Wasm = record {
  
  calculated_hash: blob;                // SHA-256 or similar hash of the wasm binary. Defaults to all 0 bytes
  hash: blob;                           // Should be used as the key
  description: text;                    // Human-readable description of the wasm.
  repo: text;                           // Optional URL to the public repository.
  metadata: ICRC16Map;                  // Additional metadata as key-value pairs.
  version_number: record {nat; nat; nat;}; // Sequential version identifier - semver notation
  previous_hash: opt blob;              // Hash of version this item should be upgraded from with this wasm.
  deprecated: bool;                     // Indicates if the wasm version is deprecated.
  canister_type_namespace: text;                  // Type of the canister associated with this wasm.
  chunkCount: nat;                      // Number of chunks the wasm is divided into.
  created: nat;                         // Timestamp of creation.
};

icrc118_get_wasms: (record {
  filter:  opt vec variant {
    #hash: blob;
    #canister_type: text
    #version_min: record {nat; opt nat; opt nat;};
    #version_max: record {nat; opt nat; opt nat;};
    #controllers: vec principal;
    #previous_hash: blob; //will find descendants of this hash
    #canister: principal; //will find all versions previously installed to this canister
  };
  prev: opt blob;
  take: opt nat;
}) -> (vec Wasm) query;
```

#### `icrc118_update_wasm`
- **Purpose:** Registers or updates a Wasm version in the registry.
- **Input:**
  - `canister_type`: Identifier.
  - `version_number`: (Major, Minor, Patch).
  - `description`: Text summary.
  - `repo`: Repository URL (optional).
  - `metadata`: Key-value pairs (ICRC-16).
  - `previous_hash`: (Optional) Hash of the prior version.
  - `expected_hash`: Expected file hash.
  - `chunk_count`: Number of chunks.

- **Output:** Confirmation with a unique transaction record.

```candid
type UpdateWasmRequest = record {
    canister_type: text;
    version_number: record {nat; nat; nat;};
    description: text;
    repo: text;
    metadata: ICRC16Map;
    previous_hash: opt blob;
    expected_hash: blob;
    expected_chunks: nat;
};

type UpdateWasmResult = variant {
    Ok: nat;       // Transaction record.
    Error: variant {
        Unauthorized;
        NonDeprecatedWasmFound: blob; // Returned if a non-deprecated version with a previous/version match is found.
        Generic: text;
    };
};

icrc118_update_wasm: (UpdateWasmRequest) -> (UpdateWasmResult);
```
#### `icrc118_upload_wasm_chunk`
- **Purpose:** Upload a chunk of a Wasm module to the registry.
- **Input:**
  - `hash`: Unique identifier of the Wasm.
  - `chunk_id`: Sequential identifier for the chunk.
  - `wasm_chunk`: Raw data bytes of the chunk.
- **Output:**
  - `chunk_id`: Identifier for the uploaded chunk.
  - `total_chunks`: Total number of chunks for the Wasm.

```candid
type UploadRequest = record {
  hash: blob;
  chunk_id: nat;
  wasm_chunk: vec nat8;
}

type UploadResponse = record {
  chunk_id: nat;           // Identifier for the uploaded chunk.
  total_chunks: nat;       // Total number of chunks for the WASM.
};

icrc118_upload_wasm_chunk: (UploadRequest) -> (UploadResponse);
```

#### `icrc118_deprecate`
- **Purpose:** Marks a Wasm version as deprecated.
- **Input:** 
  - `wasm_hash`: Unique identifier of the Wasm.
  - `deprecation_flag`: Boolean (defaults to `true` if omitted).
- **Output:** Success status and updated record.

```candid
DeprecateRequest = record {
    hash: blob;
    deprecation_flag: opt bool;
}

type DeprecateResult = variant {
    Ok: nat;       // Transaction record.
    Error: variant {
        Unauthorized;
        NotFound;
        Generic: text;
    };
};

icrc118_deprecate: (DeprecateRequest) -> (DeprecateResult);
```

---

### 2.2 Registry Queries

#### `icrc118_get_canister_types`
- **Purpose:** Searches for canister types by namespace and controller.
- **Input:** Namespace filtering or controller ID.
- **Output:** List of matching canister types.

```candid
type GetCanisterTypesRequest = record {
  filter: vec variant{
    namespace: text;
    controller: principal;
  };
  prev: opt text; //namespace
  take: opt nat;
};

type CanisterVersion = record { 
  version: record {nat; nat; nat };
  hash: blob;
  calculated_hash: blob;
};

type CanisterType = record {
  namespace: text;
  versions: vec CanisterVersion;
  description: text;
  repo: text;
  metadata: ICRC16
  controllers: vec principal;
};

icrc118_get_canister_types: (GetCanisterTypesRequest) -> (vec CanisterType) query;
```

#### `icrc118_get_upgrade_path`
- **Purpose:** Computes **upgrade steps** needed to reach a target Wasm version.
- **Input:**
  - `canister_type_namespace`  
  - `current_version`
  - `target_version`
- **Output:** Ordered list of version upgrades.

```candid
type GetUpgradePathRequest = record {
  canister_type_namespace: text;
  current_version: blob;
  target_version: blob;
};

icrc118_get_canister_types: (GetUpgradePathRequest) -> (vec CanisterVersion) query;
```

#### `icrc118_get_wasm_chunk`
- **Purpose:** Retrieve a specific chunk of a Wasm module from the registry.
- **Input:**
  - `hash`: Unique identifier of the Wasm.
  - `chunk_id`: Sequential identifier for the chunk.
- **Output:**
  - `chunk_id`: Identifier for the retrieved chunk.
  - `wasm_chunk`: Raw data bytes of the chunk.

```candid
type GetWasmChunkRequest = record {
  hash: blob;
  chunk_id: nat;
};

type GetWasmChunkResponse = record {
  chunk_id: nat;           // Identifier for the retrieved chunk.
  wasm_chunk: vec nat8;    // Raw data bytes of the chunk.
};

icrc118_get_wasm_chunk: (GetWasmChunkRequest) -> (GetWasmChunkResponse) query;
```

## 3. Integration & Compliance

### ICRC-10

An implementation of ICRC-118 Registry MUST also support the method `icrc10_supported_standards` (per [ICRC-10](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-10/ICRC-10.md)), and include the following entries in its response:

```candid
  record { name = "ICRC-10"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-10"; }
  record { name = "ICRC-118"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-118"; }
```

### ICRC-16

This standard utilizes generic data structures defined in [ICRC-16](https://github.com/dfinity/ICRC/issues/16).


### ICRC-86 - Domain Claim Standard

It is recommended that for canister type namespaces that systems implementing ICRC-118 SHOULD consider implementing [ICRC-86](https://github.com/dfinity/ICRC/issues/86) to establish clear ownership of domains. This also gives assurance to controllers of canister types that others cannot spoof or impersonate namespaces that could confuse users.

### ICRC-105 - Wasm History Management and Transaction Blocks

It is recommended that systems implementing ICRC-118 while also allowing the system to deploy wasms on behalf of the user to limit wasms support to wasms and applications that support [ICRC-105](https://github.com/dfinity/ICRC/issues/105) blocks. These blocks help provide assurance that wasm upgrade pathways have been followed and the chain of trust offered by the Internet Computer technology has not been violated.

### ICRC-119 - Wasm Registry Block Specification

It is recommended that systems implementing ICRC-118 support [ICRC-119](https://github.com/dfinity/ICRC/issues/119) blocks. These blocks help provide assurance that the wasm registry has integrity and that the ownership of canister types and wasm history have not been manipulated by third parties.

### ICRC-126 - Wasm verification

It is recommended that systems implementing ICRC-118 utilize [ICRC-126](https://github.com/dfinity/ICRC/issues/126) to provide a mechanism for validators to endorse the build and hash of wasm versions.

### ICRC-127 - Generic Bounty System

It is recommended that systems implementing ICRC-118 utilize [ICRC-127](https://github.com/dfinity/ICRC/issues/127) to enable their system to recover wasm files that have become stale and have been removed from the canister.

## 4. Security Considerations

It is strongly recommended that implementations review the Security Considerations put forward in [ICRC-7 - Security Considerations](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-7/ICRC-7.md#security-considerations) and apply the same care to the implementation of ICRC-118.


