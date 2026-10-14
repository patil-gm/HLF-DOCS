# Hyperledger Fabric transaction flow with success and rejection branches

In Hyperledger Fabric, a transaction moves through multiple hops before it is finally committed. The main stages are: proposal creation, endorsement, ordering, validation, ledger commit, and world state update. A transaction can fail at several of these stages, and the outcome depends on where the failure happens.

A very important point is this:

- If a transaction fails before it is ordered into a block, it is never written to the ledger.
- If a transaction fails during validation after ordering, it is still written into the block ledger, but it is marked invalid and its write set is not applied to the world state.

## 1. Transaction proposal creation by client

The process starts when the client application or SDK creates a transaction proposal. This proposal contains the channel name, chaincode name, function name, arguments, client identity, nonce, and signature.

At this stage, the transaction can fail even before reaching the endorsing peer. This can happen if the proposal is malformed, the client identity is invalid, the signature is incorrect, or the wrong channel or chaincode name is used.

If the transaction fails here, it does not move forward. It is not endorsed, not ordered, not written to the ledger, and not reflected in the world state.

## 2. Proposal sent to endorsing peer

Next, the client sends the proposal to one or more endorsing peers. Each endorsing peer first performs admission checks before simulating the chaincode.

The endorsing peer verifies whether:
- the client's identity is valid,
- the signature is correct,
- the client is authorized to invoke the chaincode,
- the channel exists and the peer has joined it,
- the chaincode is installed and defined correctly.

At this stage, the transaction can be rejected if the client is not authorized, the certificate is invalid, the channel is wrong, the chaincode is missing, or the proposal format is incorrect.

If rejected here, the peer does not return a valid endorsement. If the client cannot collect enough valid endorsements from other peers, the transaction stops here. It never reaches ordering, so it is not written into the ledger or world state.

## 3. Chaincode simulation on endorsing peer

If admission checks succeed, the endorsing peer runs the chaincode in simulation mode. The chaincode reads the current world state and produces a read set and a write set. This simulation does not actually update the world state. It only calculates what would be changed if the transaction is later committed.

At this stage, the transaction can fail because of chaincode-level reasons such as:
- insufficient balance,
- asset not found,
- duplicate asset creation,
- invalid business input,
- runtime exception or panic,
- timeout or chaincode container/service failure.

If simulation fails, the peer returns an error instead of a successful endorsement. In that case, the transaction never proceeds to ordering, so it is not stored in the ledger and does not affect world state.

## 4. Endorsement collection by the client

After receiving responses from endorsing peers, the client checks whether the endorsements satisfy the endorsement policy. The client also verifies that all successful endorsers returned the same read-write set and response payload.

A transaction can fail at this stage if:
- not enough endorsements are collected,
- the required organizations did not endorse,
- endorsement signatures are invalid,
- endorsers returned different results,
- the chaincode is non-deterministic.

Non-deterministic chaincode is a common reason for failure here. For example, if chaincode uses current time, random numbers, external API calls, or unstable iteration order, two endorsers may produce different results for the same proposal.

If the client cannot assemble a valid endorsed transaction, the transaction is not submitted to the orderer. Therefore, it is never included in a block, never written to the ledger, and never written to world state.

## 5. Submission to the ordering service

If the endorsement policy is satisfied, the client sends the endorsed transaction to the ordering service.

At this stage, the transaction can fail because of:
- orderer authentication failure,
- wrong channel,
- malformed transaction envelope,
- TLS or network failure,
- message size limits,
- unavailable orderer node.

If submission fails, the transaction is not ordered into a block. As a result, it is not written into the ledger and does not update world state.

## 6. Ordering and block creation

The ordering service does not execute chaincode and does not check endorsement logic the same way committing peers do. Its main role is to establish a total order of transactions and package them into blocks.

At this stage, the transaction may be delayed or temporarily blocked if the orderer is down, there is a consensus issue, or the ordering service is unavailable. In such cases, the transaction is not yet committed anywhere.

If the ordering step succeeds, the transaction becomes part of a block and is delivered to committing peers.

## 7. Validation on committing peers

After peers receive the block, they validate each transaction before applying it. This is a critical point in Fabric.

Validation includes checks such as:
- endorsement policy validation,
- endorser signature validation,
- duplicate transaction ID detection,
- MVCC validation,
- phantom read/range query validation,
- private data/hash consistency checks where applicable.

This is the stage where a transaction may be marked invalid even though it was successfully endorsed and ordered.

### Common validation failure scenarios

#### a. Endorsement policy failure
The peer may detect that the transaction does not actually satisfy the required endorsement policy. This can happen if endorsements are missing, invalid, or not from the required organizations.

#### b. MVCC conflict
This happens when the transaction was simulated using older versions of keys, but before commit another transaction already changed those keys. The read set no longer matches the current committed state, so the transaction is invalidated.

For example, if two transactions both read the same asset version and both try to update it, the one validated first may succeed, while the later one fails because the version changed.

#### c. Phantom read conflict
If a transaction depends on a range query result, and another committed transaction changes the keys in that range before validation, the original result is no longer stable. Fabric may mark the transaction invalid due to a phantom read conflict.

#### d. Duplicate transaction ID
If the same transaction ID is submitted twice, the duplicate can be marked invalid.

#### e. Private data validation issues
For private data transactions, the peer may detect missing or inconsistent private data hashes or policy mismatches, causing the transaction to be invalid.

## 8. Ledger commit behavior

Once validation is complete, the block is written to the ledger, also called the block store. This ledger is immutable and stores the complete history of transactions.

At this point, both valid and invalid transactions are recorded in the ledger. This is extremely important in Fabric:

- valid transactions are stored in the ledger,
- invalid transactions are also stored in the ledger,
- invalid transactions are marked with an invalidation reason or validation code.

So if a transaction fails during validation after ordering, it still becomes part of blockchain history.

## 9. World state update behavior

After the block is written to the ledger, Fabric updates the world state database. The world state is typically stored in LevelDB or CouchDB and contains only the latest values of keys.

Only valid transactions update the world state.

This means:
- if a transaction is valid, its write set is applied to the world state,
- if a transaction is invalid, its write set is skipped,
- the transaction still remains in the ledger history.

So the ledger contains the full historical record, while the world state contains only the latest valid state.

## 10. Success path

A transaction is considered fully successful when:
- the proposal is valid,
- endorsing peers simulate it successfully,
- required endorsements are collected,
- the orderer accepts and includes it in a block,
- committing peers validate it successfully,
- the block is written to the ledger,
- the write set is applied to world state.

In this case:
- the transaction is present in the ledger,
- it is marked valid,
- the world state is updated.

## 11. Rejection path before ordering

If the transaction fails during proposal creation, proposal admission, simulation, endorsement collection, or submission to orderer, then it never enters a block.

In this case:
- it is not in the ledger,
- it is not in world state.

## 12. Invalid path after ordering

If the transaction is ordered into a block but fails during validation at the committing peer, then the result is different.

In this case:
- it is stored in the ledger,
- it is marked invalid,
- it does not update world state.

## 13. Final conclusion

The full Hyperledger Fabric transaction flow is: the client creates a proposal, endorsing peers simulate it and return endorsements, the client collects enough matching endorsements, the transaction is submitted to the ordering service, the orderer places it into a block, peers validate it, the block is committed to the ledger, and finally valid transactions update the world state.

The most important rule is that failures before ordering prevent the transaction from entering the ledger, while failures during post-order validation still allow the transaction to be stored in the ledger as invalid, but prevent any change to world state.