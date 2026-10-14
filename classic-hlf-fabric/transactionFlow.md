# Hyperledger Fabric transaction flow with success and rejection branches

## End-to-end sequence

```text
Client App / SDK
   |
   | 1. Create transaction proposal
   |    - channel, chaincode, function, args, identity, signature
   |
   |----> If malformed / bad identity / bad signature
   |        => REJECTED at client or first receiving node
   |        Reason:
   |        - invalid certificate
   |        - bad signature
   |        - wrong channel/chaincode name
   |        Result:
   |        - not ordered
   |        - not in ledger
   |        - not in world state
   |
   v
Endorsing Peer
   |
   | 2. Proposal admission checks
   |    - authentication
   |    - authorization / ACL
   |    - channel existence
   |    - chaincode availability / definition
   |
   |----> If admission checks fail
   |        => REJECTED by endorsing peer
   |        Reason:
   |        - client not authorized
   |        - peer not joined to channel
   |        - chaincode not found / not approved / not committed
   |        - invalid proposal format
   |        Result:
   |        - no endorsement from this peer
   |        - transaction cannot proceed unless enough endorsements are still collected
   |        - not in ledger
   |        - not in world state
   |
   v
Chaincode Simulation on Endorsing Peer
   |
   | 3. Simulate transaction against current world state
   |    - read set generated
   |    - write set generated
   |    - response payload generated
   |
   |----> If simulation fails
   |        => REJECTED during endorsement/simulation
   |        Reason:
   |        - business rule failure
   |        - insufficient balance
   |        - asset not found
   |        - duplicate create
   |        - chaincode panic / exception / timeout
   |        Result:
   |        - endorsement not returned successfully
   |        - not ordered
   |        - not in ledger
   |        - not in world state
   |
   v
Client App / SDK
   |
   | 4. Collect endorsements from required org peers
   |    - verify signatures
   |    - verify same RW set and response
   |    - check endorsement policy can be satisfied
   |
   |----> If endorsement collection fails
   |        => REJECTED at client side before ordering
   |        Reason:
   |        - insufficient endorsements
   |        - endorsement policy not satisfied
   |        - RW set mismatch
   |        - non-deterministic chaincode
   |        - invalid endorsement signature
   |        Result:
   |        - transaction not submitted to orderer
   |        - not in ledger
   |        - not in world state
   |
   v
Ordering Service
   |
   | 5. Submit endorsed transaction to orderer
   |
   |----> If orderer submission fails
   |        => REJECTED by ordering service / network
   |        Reason:
   |        - broadcast auth failure
   |        - wrong channel
   |        - malformed envelope
   |        - TLS/network timeout
   |        - message too large
   |        Result:
   |        - not included in block
   |        - not in ledger
   |        - not in world state
   |
   v
Ordering Service
   |
   | 6. Order transactions and create block
   |    - establish total order on channel
   |    - batch transactions into block
   |
   |----> If orderer unavailable / consensus issue
   |        => DELAY / TEMPORARY FAILURE
   |        Reason:
   |        - orderer down
   |        - leader/consensus issue
   |        - channel service problem
   |        Result:
   |        - transaction not yet committed
   |        - not yet in ledger
   |        - not yet in world state
   |
   v
Committing Peer
   |
   | 7. Receive block from orderer
   | 8. Validate each transaction in the block
   |    - endorsement policy validation
   |    - signature checks
   |    - MVCC validation
   |    - phantom read / range query validation
   |    - duplicate TxID checks
   |
   |----> If validation fails
   |        => MARKED INVALID at commit time
   |        Reason:
   |        - endorsement policy failure
   |        - missing/invalid endorsements
   |        - MVCC read conflict
   |        - phantom read conflict
   |        - duplicate transaction ID
   |        - private data/hash validation failure
   |        Result:
   |        - transaction IS written in block ledger
   |        - transaction IS marked invalid
   |        - transaction DOES NOT update world state
   |
   v
Block Store / Ledger
   |
   | 9. Commit block to ledger
   |    - immutable block history stored
   |    - valid and invalid transactions both recorded
   |
   v
State DB (World State: LevelDB or CouchDB)
   |
   | 10. Apply write sets of VALID transactions only
   |     - update keys
   |     - update versions
   |     - invalid transactions skipped
   |
   +----> SUCCESS PATH
            Result:
            - transaction is in ledger
            - validation code = VALID
            - world state updated

```

---

## Short branch summary

### Branch A: Rejected before ordering
Happens at:
- proposal creation
- endorsement admission
- simulation
- endorsement collection
- submit to orderer

Result:
- transaction never enters a block
- not stored in ledger
- no world state update

---

### Branch B: Ordered but invalid at commit time
Happens at:
- peer validation after block delivery

Reasons:
- endorsement policy failure
- MVCC conflict
- phantom read conflict
- duplicate TxID
- private data/hash mismatch

Result:
- transaction is stored in ledger/block
- transaction is marked invalid
- world state is not updated

---

### Branch C: Successful commit
Happens when:
- proposal succeeds
- endorsements satisfy policy
- orderer includes transaction in block
- peer validation passes

Result:
- transaction is stored in ledger/block
- transaction is marked valid
- write set is applied to world state

---

## One-line conclusion

In Hyperledger Fabric, a transaction is first proposed and simulated, then endorsed, ordered into a block, validated by peers, written to the ledger, and only if it is valid are its writes applied to the world state database.