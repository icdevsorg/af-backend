|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|119|Wasm Registry Block Specification - ICRC3 Extension|Austin Fatheree(@skilesare)|https://github.com/dfinity/ICRC/issues/119|Draft|Standards Track||2025-03-18|

---

## 1. Overview

ICRC-119 defines a set of block types that extend the ICRC-3 ledger to provide an auditable, immutable log of operations performed on a Wasm Registry as defined in ICRC-118. The specification covers the creation of canister types, detailed Wasm updates, chunked upload events, management of controller lists, and deprecation of outdated Wasm versions.

These blocks enable:
- **Deterministic logging** of Wasm registry events.
- **Audit trail compliance** via ICRC-3 blocks.
- **Standardized operations** for Wasm registry management.

---

## 2. Scope

ICRC-119 is designed to record and track the following key operations of a wasm registry:

- **Canister Type Registration:**  
  Recording the creation of new canister types with associated controllers and forking history.

- **Wasm Update Details:**  
  Logging updates or modifications to Wasm module information (e.g., version, description, metadata).

- **Chunked Wasm Uploads:**  
  Capturing the upload of individual chunks of a Wasm module to accommodate large binaries.

- **Controller Management:**  
  Documenting additions or removals of controllers for canister types.

- **Wasm Deprecation:**  
  Marking specific Wasm module versions as deprecated.


---

## 3. ICRC-119 Block Schemas

Each block is a discrete ICRC-3 record that includes common fields (e.g., block type identifier `btype`, timestamp `ts`) along with a transaction (`tx`) record detailing the specific event.

### 3.1. 119canister_type

**Purpose:**  
Records the registration or creation of a new canister type in the Wasm Registry.

**tx Fields:**
- **caller**: *Blob*  
  The identity of the principal initiating the creation.
- **canister_type_namespace**: *Text*  
  A unique identifier for the canister type.
- **canister_type_name**: *Text*  
  A display name for the canister type.
- **controllers**: *Array of Blob*  
  A list of principals granted administrative control.
- **forked_from**: *Optional Record*  
  If applicable, details of the parent canister type
    - `canister_type_namespace`: text
    - `version`: Array[Nat] - using semver notation[Major, Minor, patch]
- **description**: *Optional Text*  
  A brief description of the canister type and its purpose.
- **repo**: *Optional Text*  
  A pointer to a repo URL for the type.
- **metadata**: *Optional ICRC16*  
  Additional Metadata.

**Block Header:**
- **btype**: `"119canister_type"`
- **ts**: Timestamp of record creation

---

### 3.2. 119wasm_detail

**Purpose:**  
Records detailed information when a Wasm module is registered or updated in the registry.

**tx Fields:**
- **caller**: *Blob*  
  The principal initiating the Wasm update.
- **canister_type**: *Text*  
  Identifier of the associated canister type.
- **version_number**: *Record { nat; nat; nat; }*  
  The version of the Wasm using semver (Major, Minor, Patch).
- **description**: *Text*  
  Summary of the update or change in functionality.
- **repo**: *Optional Text*  
  URL pointing to the repository with the source or further details.
- **metadata**: *Map*  
  Key-value pairs (as defined by ICRC-16) providing additional context.
- **previous_hash**: *Optional Blob*  
  Hash of the previous version (if this is an upgrade).
- **expected_hash**: *Blob*  
  The calculated hash of the updated Wasm binary.
- **expected_chunks**: *Nat*  
  Number of chunks expected for the complete Wasm.

**Block Header:**
- **btype**: `"119wasm_detail"`
- **ts**: Timestamp of update

---

### 3.3. 119wasm_chunk

**Purpose:**  
Captures the upload of a specific chunk of a Wasm module.

**tx Fields:**
- **caller**: *Blob*  
  The principal responsible for uploading the chunk.
- **wasm_hash**: *Blob*  
  Unique identifier (hash) of the Wasm module.
- **chunk_id**: *Nat*  
  Sequential identifier of the current chunk.
- **chunk_hash**: *Blob*  
  Hash of the chunk.
- **total_chunks**: *Nat*  
  Total number of chunks that make up the Wasm module.
- **calculated_hash_match**: Optional *Nat*  
  Should only appear on the final chunk and be set to `1` if the hash matches and `0` if it does not.

**Block Header:**
- **btype**: `"119wasm_chunk"`
- **ts**: Timestamp of the chunk upload

---

### 3.4. 119manage_controller

**Purpose:**  
Logs events related to modifications of the controller list for a given canister type.

**tx Fields:**
- **caller**: *Blob*  
  The principal performing the controller management operation.
- **canister_type**: *Text*  
  Identifier of the affected canister type.
- **controller**: *Blob*  
  The principal being added or removed.
- **operation**: *Text* (values: `"Add"` or `"Remove"`)  
  Specifies whether the controller is being added or removed.

**Block Header:**
- **btype**: `"119manage_controller"`
- **ts**: Timestamp of the operation


---

### 3.5. 119deprecate_wasm

**Purpose:**  
Marks a specific Wasm module version as deprecated.

**tx Fields:**
- **caller**: *Blob*  
  The principal requesting the deprecation.
- **wasm_hash**: *Blob*  
  The unique identifier (hash) of the Wasm to be deprecated.
- **deprecation_flag**: *Bool*  
  Indicates whether the Wasm is being marked as deprecated (true) or reinstated (false).
- **reason**: *Optional Text*  
  Explanation or rationale for the deprecation.

**Block Header:**
- **btype**: `"119deprecate_wasm"`
- **ts**: Timestamp of the deprecation request

---

## 4. Integration & Compliance

ICRC-119 is intended to complement and extend the functionalities provided by ICRC-118 by introducing a set of fine-grained events specifically focused on Wasm registry operations. It complies with the ICRC-3 block standards for immutability and auditability. Integrators are encouraged to:
- Ensure events are recorded in strict operational order.
- Validate that each transaction adheres to the defined structure.
- Handle unknown or future fields gracefully, preserving forward compatibility.

---

## 5. References

### ICRC-3

An implementation of ICRC-119 Registry MUST also support the methods and standards defined in [ICRC-3](https://github.com/dfinity/ICRC-1/tree/main/standards/ICRC-3)

### ICRC-10

An implementation of ICRC-119 Registry MUST also support the method `icrc10_supported_standards` (per [ICRC-10](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-10/ICRC-10.md)), and include the following entries in its response:

```candid
  record { name = "ICRC-3"; url = "https://github.com/dfinity/ICRC-1/tree/main/standards/ICRC-3"; }
  record { name = "ICRC-10"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-10"; }
  record { name = "ICRC-119"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-119"; }
```
