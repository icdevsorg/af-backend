|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|126| Wasm Verification|Austin Fatheree (@skilesare)|https://github.com/dfinity/ICRC/issues/126|Draft|Standards Track||2025-03-18|

---

## 1. Scope

ICRC-126 defines the wasm verification life cycle for WebAssembly (Wasm) modules on the Internet Computer. Building on the foundation provided by ICRC-105, this specification extends standard practices by providing a framework for financial verification, attestation, divergence reporting, refund, and dividend distribution into the Wasm deployment lifecycle. This document is intended to serve as a reference for developers and service providers to implement secure and traceable compilation and deployment processes. 

ICRC-126 covers the following key aspects:
- **Wasm Version Details:** Ensures that a method and recording instrument exists for a wasm version that may include language, libraries, payment details, etc.
- **Attestation Generation:** Provides immutable and traceable attestation for a wasm module version.
- **Divergence Reporting:** Allows for the logging of discrepancies between expected and actual compiled outputs.
- **Finalization:** Outlines records for finalization in the affirmative or rejection of a wasm module version.

### Out of Scope

ICRC-126 does not define the *process* for verifying a build in any specific language and only dictates that the output for a verifiable build should be a sha-256 hash of the resulting wasm that will be installed on a canister. Please see the Addendum section for pointers to ICRC definitions for build processes for different languages.

ICRC-126 does not define the *economics* for verifying a build. The endpoints and process are defined but the incentivization, consequences of attesting incorrectly, and payment for a validation is not defined on purpose. It should be the responsibility of the ICRC-126 Implementor to determine how to incentivize honest and quick wasm build validity.

---

## 2. Terminology and Definitions

### 2.1 WebAssembly (Wasm) Module
A binary executable unit intended to be deployed on the Internet Computer, identified by its cryptographic hash.

### 2.2 Attestation Record
A verifiable record that certifies the integrity of a compiled Wasm module, linking it definitively to its compilation process.

### 2.3 Divergence Report
A logged record detailing discrepancies between the expected compile output and the actual generated Wasm module. 

---

## 3. Wasm Verification Service Function Definition

### 3.1 icrc126_verification_request

- **Purpose:** Initiates the compilation and deployment of a new Wasm module ensuring that valid payment information is provided.
- **Inputs:**
  - `caller : principal` – The initiating entity.
  - `wasm_hash : blob` – The binary hash of the compiled Wasm.
  - `repo : text` - The repo that holds the code.
  - `commit_hash: blob` - The commit hash for this version of the wasm.
  - `metadata : ICRC16` - additional data about the build. ie. Payment Info, Compiler Info,etc
- **Expected Behavior:**
  - Record the information in a 126request block
  - Return the transaction id as a unique identifier for the newly submitted module.

### Candid Definition

```candid
type VerificationRequest = record {
  caller: principal;
  wasm_hash: blob;
  repo: text;
  commit_hash: blob;
  metadata: ICRC16;
};

icrc126_verification_request: (VerificationRequest) -> (nat);
```

---

### 3.2 126_file_attestation

- **Purpose:** Generates a secure attestation record that certifies the compilation process for a Wasm module by a caller
- **Inputs:**
  - `wasm_id : Text` – Unique identifier of the compiled Wasm module.
  - `metadata: ICRC16` Data about the attestation.
- **Expected Behavior:**
  - Create a verifiable attestation record that links the caller to attesting to the validity of the Wasm module in a 126attest block
  - Typically the calculated hash will be placed in the metadata in a Map with the key `{"126hash",#Blob({hash})}`
  - If the attestation amends a previous attestation or divergence report, include the key `{"126amends",#Nat({original_trx_id})}`

### Candid Definition

```candid
type AttestationRequest = record {
  wasm_id: text;
  metadata: ICRC16;
};

icrc126_file_attestation: (AttestationRequest) -> (nat);
```

---

### 3.3 icrc126_file_divergence

- **Purpose:** Records and files a divergence report detailing discrepancies observed in the compilation output.
- **Inputs:**
  - `wasm_id : Text` – Identifier of the Wasm module.
  - `divergence_report : text` – Details of the divergence.
  - `metadata : opt ICRC16` - Data annotations
- **Expected Behavior:**
  - Log the divergence report with full details in an 126divergence block
  - Typically the calculated hash, if available, will be placed in the metadata in a Map with the key `{"126found_hash",#Blob({hash})}`
  - If the divergence amends a previous attestation or divergence report, include the key `{"126amends",#Nat({original_trx_id})}`

### Candid Definition

```candid
type DivergenceReportRequest = record {
  wasm_id: text;
  divergence_report: text;
  metadata: opt ICRC16;
};

icrc126_file_divergence: (DivergenceReportRequest) -> (nat);
```
---

## 6. Block Types for ICRC-3 Logging

In addition to function implementations, events in the verifiable compile process must be logged as ICRC-3 blocks. The following block types record the details of the wasm verification lifecycle in a transaction log.  Most blocks have undefined metadata to allow for various implementations to record various data about event. Map keys in the data must be uniquely namespaced. If the key begins with a number it is assumed that the number is the ICRC standard where the data type and use is defined:

### 6.1 126request Block

- **description** - Records a new request for a validation
- **btype:** `"126request"`
- **Required Fields:**
  - `tx.caller: Principal` – The initiating entity.
  - `tx.wasm_hash: Blob` – The binary hash of the compiled Wasm.
  - `tx.repo: Text` – The repository that holds the code.
  - `tx.commit_hash: Blob` – The commit hash for this version of the Wasm.
  - `tx.metadata: ICRC16` – Additional data about the build, e.g., Payment Info, Compiler Info, etc.

### 6.2 126attest Block

- **description** - Records a new or updated attestation
- **btype:** `"126attestation"`
- **Required Fields:**
  - `tx.wasm_id: Text`
  - `tx.metadata: ICRC16`

### 6.3 126divergence Block

- **btype:** `"126divergence"`
- **Required Fields:**
  - `tx.wasm_id: Text` – Identifier of the Wasm module.
  - `tx.divergence_report: Text` – Details of the divergence.
  - `tx.metadata: opt ICRC16` – Data annotations.

### 6.4 126verified Block

- **btype:** `"126verified"`
- **Required Fields:**
  - `tx.wasm_id: Text` – Identifier of the Wasm module.
  - `tx.metadata: ICRC16` - Details of the verification

### 6.5 126rejected Block

- **btype:** `"126rejected"`
- **Required Fields:**
  - `tx.wasm_id: Text` – Identifier of the Wasm module.
  - `tx.metadata: ICRC16` - Details of the verification

---

## 8. References

### ICRC-10

An implementation of ICRC-126 Registry MUST also support the method `icrc10_supported_standards` (per [ICRC-10](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-10/ICRC-10.md)), and include the following entries in its response:

```candid
  record { name = "ICRC-3"; url = "https://github.com/dfinity/ICRC-1/tree/main/standards/ICRC-3"; }
  record { name = "ICRC-10"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-10"; }
  record { name = "ICRC-129"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-120"; }
```

### ICRC-16

This standard utilizes generic data structures defined in [ICRC-16](https://github.com/dfinity/ICRC/issues/16).

## 9. Security Considerations

It is strongly recommended that implementations review the Security Considerations put forward in [ICRC-7 - Security Considerations](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-7/ICRC-7.md#security-considerations) and apply the same care to the implementation of ICRC-126.

## 10. Addendum

Motoko Build Verification - [ICRC-128](https://github.com/dfinity/ICRC/issues/128)
Rust Build Verification - [ICRC-129](https://github.com/dfinity/ICRC/issues/129)
Azel Build Verification - [ICRC-130](https://github.com/dfinity/ICRC/issues/130)
Kybra Build Verification - [ICRC-131](https://github.com/dfinity/ICRC/issues/132)
C++ Build Verification - [ICRC-132](https://github.com/dfinity/ICRC/issues/133)
