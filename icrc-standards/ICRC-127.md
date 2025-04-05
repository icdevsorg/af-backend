|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|127|Generic Bounty System |Austin Fatheree(@skilesare)|[Link to Discussion](https://github.com/dfinity/ICRC/issues/127)|Draft|Standards Track|General|2025-03-18|

---

# ICRC-127: Generic Bounty System

ICRC-127 establishes a standardized framework for creating and managing bounties on the Internet Computer. The system incentivizes a wide range of challenges—from recovering deleted WASM modules to proving data integrity, code correctness, or possession of a digital asset—by securely holding bounty funds in escrow and releasing them only when a claimant’s submission meets the required criteria. A key innovation is the integration of a dedicated validation canister that interprets generic challenge parameters and performs the necessary verifications.

---

## 1. Scope

The Generic Bounty System (ICRC-127) is designed to support decentralized incentive mechanisms where tasks are defined as challenges. Users (bounty creators) deposit tokens into escrow along with a challenge description. Claimants then submit evidence or data in response to the challenge via a standard submission process. A specified validation canister executes a challenge routine—using generic parameters (such as an ICRC-16 map or a raw blob)—and returns a boolean outcome. Upon successful validation, the system releases the escrowed funds to the claimant or applies alternate payout rules as specified by the bounty creator.

ICRC-127 covers the following:

- **Bounty Creation and Escrow:**  
  Users create bounties by depositing tokens and configuring challenge parameters. Funds are held securely until a claim is validated or the bounty times out.

- **Claim Submission and Validation:**  
  Claimants invoke the bounty submission function (e.g., `icrc127_submit_bounty`) with generic parameters. The bounty canister relays these parameters to a designated validation canister (via its `validation_canister_id`), which processes the challenge and returns a verification result.

- **Generic Challenge Parameters:**  
  Challenges are parameterized using flexible types (e.g., an ICRC-16 map or blob) so that diverse use cases (code verification, data integrity, proof-of-possession, etc.) can be supported.

- **Reward Distribution and Escrow Release:**  
  Upon successful validation, escrowed funds are transferred to the claimant or handled per bounty-specific payout rules. The system may also support partial refunds if a bounty is not claimed within a specified timeout.

- **Extensibility and Modular Architecture:**  
  The system is modular, allowing new validation canisters to be integrated, and enables automated as well as community-governed verification processes.

---

## 2. Terminology and Definitions

- **Bounty:**  
  A task defined by a challenge that requires a specific submission to validate—such as the recovery of a deleted wasm, the provision of specific data, or proof of asset possession.

- **Bounty Canister:**  
  A smart contract deployed on the Internet Computer that manages bounty creation, token escrow, claim submissions, and reward distribution.

- **Validation Canister:**  
  A dedicated canister referenced by a bounty that implements a standard verification routine (e.g., running fuzz tests, comparing hashes, executing challenge–response protocols) using generic parameters.

- **Generic Parameters:**  
  Data structures (ICRC-16 - Usually a Map or Blob) that allow bounties to specify challenge criteria in a flexible, extensible manner.

- **Escrow:**  
  A secure holding state for tokens deposited by bounty creators until the challenge is successfully completed or expires.

---

## 3. Bounty System Overview

The architecture of ICRC-127 comprises three principal components:

1. **Bounty Creation and Configuration:**  
   - A bounty creator calls a function (e.g., `icrc127_create_bounty`) to deposit tokens and configure the bounty.  
   - Key parameters include a challenge identifier (which might be a hash or an opaque blob), the token amount, timeout period, and the `validation_canister_id`.

2. **Claim Submission and Validation:**  
   - A claimant initiates the bounty claim via `icrc127_submit_bounty`, providing generic parameters that encapsulate the evidence or solution.
   - The bounty canister logs the claim and produces a claim_id. This is returned to the user and the rest of the process runs asynchronously. The canister may choose to run it synchronously if it is not a long running process.
   - The bounty canister then invokes `icrc127_run_bounty` (or equivalent) by calling the validation canister, passing along the generic parameters.
   - The validation canister runs its process (e.g., running tests, comparing data) and returns a Boolean result indicating whether the claim meets the challenge.

3. **Reward Distribution:**  
   - If the validation canister returns a positive result, the bounty system marks the bounty as claimed and releases escrowed funds to the claimant.
   - In the event of a failure or if the bounty times out, the funds are returned to the bounty creator (subject to any fees or penalty rules).

---

## 4. Bounty Creation and Escrow Management

### 4.1 icrc127_create_bounty - Creating a Bounty

A bounty is created by depositing tokens into the bounty canister. The creation function (e.g., `icrc127_create_bounty`) accepts the following parameters:

- **opt bounty_id (nat):**  
  The bounty Id if this bounty is meant to override values for an existing bounty.

- **token_canister_id (Principal):**  
  The token canister holding the funds.

- **token_amount (Nat):**  
  The number of tokens to be deposited.

- **subaccount (blob):**
  Optional subaccount where the caller's tokens are held.

- **validation_canister_id (Principal):**  
  The canister that will run the challenge verification.

- **validation_call_timeout (Nat):**  
  The number of nanoseconds the call to the validation canister should wait before assuming a failure. Using best effort messaging this allows for calls to untrusted canisters.

- **challenge_params (ICRC16):**  
  Generic parameters that detail the challenge criteria. This field allows for flexible specification of the challenge. Note: use #Blob() variant to submit just binary data.

- **timeout_date (nat):**  
  The timestamp (in nanoseconds UTC) the bounty remains expires at. This is required to prevent permanent fund loss. It is the Bounty canister's responsibility to implement rational limits on the length of a bounty. One year in the future is a suggested maximum.  Funds should be returned after this date passes.

- **start_date (opt nat):**  
  The timestamp (in nanoseconds UTC) the bounty becomes public. Once this date passes, funds should stay locked until the timeout_date.

- **bounty_metadata(vec record{text; ICRC16}):**
  Holds metadata for the intended bounty.

  The following are predefined keys. Additional keys maybe be used on an instance by instance basis. Names should be namespaced. If the key starts with a number it should refer to an ICRC standard number.

  1. **icrc127:bounty_category**  
    - **Description:** Indicates the general category of the bounty (e.g., "WASM Recovery", "Data Integrity", "Proof of Possession").
    - **Example Value:** `variant Text("Generic Verification")`

  2. **icrc127:bounty_name**  
    - **Description:** A name for the bounty.
    - **Example Value:** `variant Text("Provide WASM with hash ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad.")`

  3. **icrc127:challenge_description**  
    - **Description:** A detailed textual description of the challenge.
    - **Example Value:** `variant Text("Submit evidence that meets the challenge criteria as defined by the bounty.")`

  4. **icrc127:reward_token_type**  
    - **Description:** Text or Array - `ICRC-1`, `ICRC-2`, `ICRC-7`, `ICRC-37`, `ICRC-80`, etc. SHOULD be a token type ICRC standard.
    - **Example Value:** `variant Text("ICRC-1")`

  5. **icrc127:reward_canister**  
    - **Description:** Blob - Blob version of the principal for the reward token.
    - **Example Value:** `variant Blob("principal-blob")`

  6. **icrc127:reward_account**  
    - **Description:** Array[Blob] - [Principal, Subaccount] of the account that funds the reward. It is expected that if these are approval type rewards that they will be moved to an escrow account on the Bounty Canister. If they are non-approval types then it is the Bounty canister's responsibility to confirm that the account is owned by the Bounty Canister and the subaccount matches the user's deposit address. Deposit address creation is not covered by this ICRC standard.
    - **Example Value:** `variant Array[Blob]("principal-blob", "subaccount-blob")`

  7. **icrc127:reward_token_id**  
    - **Description:** Nat or Array - token id or array of ids for the reward.
    - **Example Value:** `variant Nat(123)`


Upon creation, the bounty system:
- Pulls the specified token amount into escrow. If the bounty is being overridden the registry should calculate the difference in amount and either refund or transfer_from the difference.
- Registers a bounty record with a unique `bounty_id` containing all relevant parameters.
- Logs the bounty creation via an immutable blockchain block (e.g., an ICRC-3 block).
- Returns a bounty id.

### 4.2 Escrow and Transaction Handling

Escrow management ensures that tokens are securely held until a bounty is claimed. The system is responsible for:
- Keeping track of deposited tokens.
- Enforcing the timeout after which funds are returned.
- Preventing double claims by marking bounties as claimed once a successful submission is validated.

---

## 5. Bounty Claim Submission and Validation

### 5.1 Claim Submission

A claimant initiates a bounty claim by calling `icrc127_submit_bounty` with:
- **Generic Submission Parameters:**  
  A blob or ICRC16Map that encapsulates the evidence or solution.

### 5.2 Validation Process

Upon receiving a claim, the bounty canister:
- Forwards the generic submission parameters to the designated validation canister via an call (e.g., `icrc127_run_bounty`).
- The validation canister executes its standard process (e.g., running tests, comparing hashes, verifying signatures) to determine if the submission meets the challenge.
- Returns
  - a Boolean result:
    - **Valid:** The submission meets the criteria.
    - **Invalid:** The submission fails to meet the criteria.
  - Claim metadata about the results

### 6.3 Reward Distribution

If the validation returns true:
- The bounty’s `claimed` flag is set to true.
- The claimant’s account is credited with the escrowed tokens.
- The bounty record is updated to record the claimant and the outcome.
- A corresponding blockchain log (e.g., an ICRC-3 127bounty_claimed) is recorded.

If the bounty expires:
- The the bounty escrow is returned, subject to fees.
- Appropriate error messages or events are recorded for auditability.
- An ICRC-3 127bounty_expired block should be recorded

---

## 7. Additional Query Functions for the Registry

To enhance data retrieval, the following query functions can be added to the registry canister interface:

### 7.1 **Get a Single Bounty**

`icrc127_get_bounty` *Returns the bounty record for a given bounty_id if it exists.*

Claim metadata MAY hold the following predefined item.  Instances may implement additional keys. They should be namespaced.

  1. **icrc127:claim_token_type**  
    - **Description:** Text or Array - `ICRC-1`, `ICRC-2`, `ICRC-7`, `ICRC-37`, `ICRC-80`, etc. SHOULD be a token type ICRC standard.
    - **Example Value:** `variant Text("ICRC-1")`

  2. **icrc127:claim_canister**  
    - **Description:** Blob - Blob version of the principal for the reward token.
    - **Example Value:** `variant Blob("principal-blob")`

  3. **icrc127:claim_account**  
    - **Description:** Array[Blob] - [Principal, Subaccount] of the account that claim was paid to. 
    - **Example Value:** `variant Array[Blob]("principal-blob", "subaccount-blob")`

  4. **icrc127:claim_amount**  
    - **Description:** Nat - amount of tokens paid for fungible tokens
    - **Example Value:** `variant Nat(10_000)`

  5. **icrc127:claim_token_id**  
    - **Description:** Nat or Array - token id if applicable. 
    - **Example Value:** `variant Nat(123)`

  6. **icrc127:claim_token_trx_id**  
    - **Description:** Nat or Array - transaction id/ids of the transfer on the remote ledger. 
    - **Example Value:** `variant Nat(678)`


---

### 7.2 **List Bounties**

`icrc127_list_bounties` *Enables clients to list bounties with optional filters such as claim status and creation time.*

---

### 7.3 **Metadata**

`icrc127_metadata` *Returns information about the instance.*

The following metadata fields are defined, but may be expanded by the instance buy using ICRC standards for metadata naming.

- `icrc127:canister_type` - Text or Array - "bounty", "verification". Indicates what role this canister plays. If only one key is present then the expected functions should be present as defined in the service specification.

---

## 8. Generic Parameterization and Validation Canisters

The flexibility of ICRC-127 comes from its use of generic parameters:
- **Parameter Type:**  
  Bounties can specify their challenge parameters as a blob or an ICRC-16 object, ensuring that any structured or opaque data may be conveyed.
- **Interchangeable Validators:**  
  The system allows different validation canisters to be registered. Each validator implements a standard interface (e.g., `icrc127_run_bounty`) that accepts generic parameters and returns a Valid/Invalid variant result.
- **Extensible Challenges:**  
  This design supports a wide range of challenges:
  - Code verification (e.g., checking that a WASM matches a specified hash and passes tests).
  - Data provision (e.g., confirming that uploaded data matches an expected fingerprint).
  - Proof-of-possession (e.g., via challenge–response cryptographic protocols).

---

## 9. Candid Function Definitions

### Types

An implementation of ICRC-127 uses the following types:

```candid

type RunBountyRequest = record {
    bounty_id: nat;
    submission_id: nat;
    submission: ICRC16;
    challenge_parameters: ICRC16;
};

type RunBountyResult = record {
    result: variant { Valid; Invalid };
    metadata: ICRC16;
    trxId: opt nat;
};


type ClaimRecord = record {
  claimId: nat;                     // trxId of the claim in the ledger. Used to identify the claim.
  timeSubmitted: nat                // timestamp
  caller: principal                 // claimer
  claim_account: opt account        // account to claim the bounty
  submission: ICRC16;               // The data submitted
  claim_metadadta: vec record {text; ICRC16}; // claim info if the claim was succesful
  result: opt RunBountyResult;      // result of the run
}
  

// Represents a bounty record.
type Bounty = record {
    bounty_id: nat;                   // Unique identifier for the bounty.
    validation_canister_id: principal;// Canister that will validate submissions.
    validation_canister_timeout: nat //nanoseconds for best effort call
    bountyMetadata: ICRC16_Map
    timeout_date: opt nat;                 // Date the bounty will expire (in nanoseconds - utc).
    claimed: opt nat;                 // Claim status. Set to trxId of the claim record.
    claims: vec ClaimRecord;          // List of Claim Attempt Records.
    created: nat;                     // Timestamp of bounty creation.
    claimed_date: opt nat             // Timestamp of claim
    
};

type CreateBountyRequest record {
    bounty_id: opt nat
    bounty_metadata: vec record {text; ICRC16};        // See specification for predefined keys
    challenge_parameters: ICRC16;                      // Parameters passed to the validation canister.
    validation_canister_id: principal;            
    timeout_date: nat;                                 //Timeout date required to prevent locked funds
    start_date: opt nat;                               //Date the bounty becomes public. Null will start immediately.
};

// Result types for bounty operations.
type CreateBountyResult = variant {
    Ok: {
      bountyId: nat;
      trxId: opt nat; //id of the icrc2_transfer_from of the tokens.
    };         // bounty ID.
    Error: variant {
        InsufficientAllowance;
        Generic: text;
    };
};

type BountySubmissionRequest = record {
    bounty_id: nat;
    submission: ICRC16;       // Generic submission parameters
    account: opt ICRC1.Account    // Account to pay rewards to upon success
}

type BountySubmissionResult = variant {
    Ok: {
      claimId: nat;               // TransactionId of claim record.
      result: opt RunBountyResult // Only available if run was done synchronously
    };
    Error: variant {
        NoMatch;
        Generic: text;
    };
};
```

### Registry Canister Methods

```candid

    // Bounty Creation: Deposits tokens into escrow and registers a new bounty.
    icrc127_create_bounty: (CreateBountyRequest) -> (CreateBountyResult);

    // Bounty Claim Submission: Claimants submit their solution.
    icrc127_submit_bounty: (BountySubmissionRequest) -> (BountySubmissionResult);

    // (Optional) Run Bounty: Internal method to invoke the validation canister.
    icrc127_run_bounty: (RunBountyRequest) -> (RunBountyResult);

    icrc127_get_bounty: (record {
      bounty_id: nat;
    }) -> (opt Bounty) query;

    icrc127_list_bounties: (record {
      filter: opt vec variant {
        claimed: bool;         // Filter by claim status.
        claimed_by: account;
        created_after: nat;      // Return bounties created after a given timestamp.
        created_before:nat;     // Return bounties created before a given timestamp.
        validation_canister: principal; // Optionally filter by validation canister.
        metadata: ICRC16Map; //Matches all provided metadata
      };
      prev: opt nat;
      take: opt nat;              // Maximum number of bounties to return.
    }) -> (vec Bounty) query;

        // Query canister information
    icrc127_metadata: () -> (vec record{text; ICRC16}) query;

    // Query supported standards (as per ICRC-10)
    icrc10_supported_standards: () -> (vec record { name: text; url: text; });

```

### Validation Canister Methods


```candid

    // Method to invoke a test of bounty accomplishment.
    icrc127_run_bounty: (RunBountyRequest) -> (RunBountyResult);

    // Query canister information
    icrc127_metadata: () -> (vec record{text; ICRC16}) query;

    // Query supported standards (as per ICRC-10)
    icrc10_supported_standards: () -> (vec record { name: text; url: text; });

```

---

## 10. Example Use Cases

ICRC-127 is designed to be adaptable across a variety of challenges. For example:

- **WASM Recovery:**  
  A bounty can be created for recovering a deleted WASM by specifying the expected WASM hash as the challenge identifier. The validation canister would reassemble uploaded chunks, verify the SHA-256 hash, and return true only if the binary is complete.

- **Data Integrity Verification:**  
  A bounty may require the submission of a dataset that matches a known fingerprint. The validation canister checks the data’s hash and format before releasing the bounty.

- **Proof-of-Possession:**  
  A claimant might prove possession of a digital asset by signing a challenge nonce. The validation canister verifies the signature and confirms that only the true owner can claim the bounty.

---

## 11. ICRC-3 Block Schemas for ICRC-127

Each block is a discrete ICRC-3 record that includes common fields (e.g., block type identifier `btype`, timestamp `ts`) along with a transaction (`tx`) record detailing the specific event.

### 11.1 **127bounty - Create/Update Bounty Block**

**Purpose:**  
Logs the creation or update of a bounty with all its key parameters.

**tx Fields:**
- **caller**: *Blob*  
  The principal blob of the caller.
- **update_bounty_id**: *Optional Nat*  
  Unique bounty identifier of the original record for the bounty if this is an update.
- **validation_canister_id**: *Blob*  
  Canister that will validate submissions.
- **validation_call_timeout**: *Nat*  
  Timeout (in nanoseconds) for calling the validation canister.
- **challenge_params**: *ICRC16*  
  A representation of the challenge parameters. Note: If this is too large the system may need to look the data up in some other manner. Incoming calls on the IC are limited to 2MB.
- **bounty_timeout_date**: *Nat*  
  Timeout tiemstamp for the bounty.
- **bounty_start_date**: *Nat*  
  Optional start timestamp for the bounty.
- **bounty_metadata**: *ICRC16*

**Block Header:**
- **btype**: `"127bounty"`
- **ts**: Timestamp of block creation

---

### 11.2 **127submit_bounty - Bounty Submission Block**

**Purpose:**  
Records when a claimant submits a response to a bounty.

**tx Fields:**
- **bounty_id**: *Nat*  
  Identifier of the bounty.
- **caller**: *Blob*  
  Principal as blob.
- **submission**: *ICRC16*  
  Submission data.
- **submission_account**: *Account*  
  Account from which the submission is made.

**Block Header:**
- **btype**: `"127submit_bounty"`
- **ts**: Timestamp of submission

---

### 11.3 **127bounty_run - Bounty Run Result Block**

**Purpose:**  
Logs a run against a validation canister .

**tx Fields:**
- **bounty_id**: *Nat*  
  Identifier of the bounty.
- **claim_id**: *Nat*  
  Unique identifier for the claim transaction.
- **result**: *Text*  
  Valid or Invalid
- **run_metadata**: *ICRC16*  
  Details of the run.

**Block Header:**
- **btype**: `"127bounty_run"`
- **ts**: Timestamp when the bounty run result was captured

---

### 11.4 **127bounty_expired - Bounty Expiration/Refund Block**

**Purpose:**  
Logs when a bounty expires and funds are returned (subject to any fee deductions).

**tx Fields:**
- **bounty_id**: *Nat*  
  Identifier of the bounty.
- **refund_trx**: *Optional Nat or Array*  
  Transaction id corresponding to the refund.
- **refund_errors**: *Optional Text*
  Any errors that may occur during refund.

**Block Header:**
- **btype**: `"127bounty_expired"`
- **ts**: Timestamp of bounty expiration

---

## 12. Reference

### 12.1 Related Standards

### ICRC-10

An implementation of ICRC-126 Registry MUST also support the method `icrc10_supported_standards` (per [ICRC-10](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-10/ICRC-10.md)), and include the following entries in its response:

```candid
  record { name = "ICRC-3"; url = "https://github.com/dfinity/ICRC-1/tree/main/standards/ICRC-3"; }
  record { name = "ICRC-10"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-10"; }
  record { name = "ICRC-129"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-120"; }
```

### ICRC-16

This standard utilizes generic data structures defined in [ICRC-16](https://github.com/dfinity/ICRC/issues/16).

## 13. Security Considerations

It is strongly recommended that implementations review the Security Considerations put forward in [ICRC-7 - Security Considerations](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-7/ICRC-7.md#security-considerations) and apply the same care to the implementation of ICRC-127.



