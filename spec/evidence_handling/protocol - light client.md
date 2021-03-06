# Light Client Attack - protocol (with pseudocode)

## Pseudocode

In this document, we represent pseudocode of the evidence handling subprotocol for the "light client attack" scenario.
<br>First, we present data structures that abstract (1) an evidence of a misbehavior of a validator,  and (2) an evidence that a light client attack has occurred.
<br>Then, we define functions and callbacks needed to ensure that validators of the baby blockchain that have mounted a light client attack are slashed at the parent blockchain.

### Data Structures

The following data structure abstracts an evidence of a misbehavior of a validator:
```golang
type InternalEvidence struct {
  evidence Evidence
  validator Validator
  chain ChainId
}
```
Namely, this data structure specifies that validator *validator* committed a misbehavior which is provable with *evidence*.
Note that we specify the identifier of chain where the misbehavior occurred (this represents a slight difference from the "single-chain" scenario).
We assume that each correct full node could verify this statement and that no verifiable evidence could ever be produced to prove a misbehavior of a correct validator.
See [Light Client Attack Detector](https://github.com/tendermint/spec/blob/master/rust-spec/lightclient/detection/detection_003_reviewed.md).

Next, the data structure below represents an evidence that a light client attack has taken place:
```golang
type LightClientAttackEvidence struct {
  conflictingBlock LightBlock
  commonHeight int64
  chain ChainId
}
```

### Protocol

Let us first describe the protocol that is executed once a light client of the baby blockchain discovers that a light client attack has occurred:
<br>Once a light client discovers that a light client attack on the baby blockchain has taken place, it transfers this information to full nodes of the baby blockchain (we assume that at least one correct full node receives this information).
<br>A correct full node is able to interpret the received information and produce an evidence of misbehavior for some faulty validators.
<br>Then, the full node transfers information about exposed faulty validators (along with evidences) to full nodes of the parent blockchain.
<br>Lastly, once a correct full node of the parent blockchain receives this information, it informs its staking module, which is responsible for ensuring that the evidences end up committed on the parent blockchain.

The following function is invoked once a light client of the baby blockchain discovers that a light client attack has occurred:
```golang
// Submits evidence of a light client attack to a set of full nodes
func submitLightClientAttackEvidence(evidence LightClientAttackEvidence, fullNodeSample []FullNode)
```
- Expected precondition
  - A light client attack has occurred
- Expected postcondition
  - Evidence is submitted to each full node from *fullNodeSample*
-Error condition
  - If the precondition is violated

As we have already mentioned, a (correct) full node of the baby blockchain should discover a set of faulty validators (with corresponding evidences of misbehaviors) whenever a light client attack is observed by the light client.
The following function captures this logic (please, see [Light client Attackers Isolation](https://github.com/tendermint/spec/blob/master/rust-spec/lightclient/attacks/isolate-attackers_002_reviewed.md#LCAI-FUNC-NONVALID1%5D) for more details):
```golang
func isolateMisbehavingProcesses(ev LightClientAttackEvidence) []InternalEvidence {
      bc := blockchain[ev.chain]
      reference := bc[ev.conflictingBlock.Header.Height].Header
      ev_header := ev.conflictingBlock.Header

      ref_commit := bc[ev.conflictingBlock.Header.Height + 1].Header.LastCommit // + 1 !!
      ev_commit := ev.conflictingBlock.Commit

      if violatesTMValidity(reference, ev_header) {
          // lunatic light client attack
          signatories := Signers(ev.ConflictingBlock.Commit)
          bonded_vals := Addresses(bc[ev.CommonHeight].NextValidators)
          return intersection(signatories,bonded_vals)

      }
      // If this point is reached the validator sets in reference and ev_header are identical
      else if RoundOf(ref_commit) == RoundOf(ev_commit) {
          // equivocation light client attack
          return intersection(Signers(ref_commit),
          Signers(ev_commit))
    }
    else {
        // amnesia light client attack
        return IsolateAmnesiaAttacker(ev, bc)
    }
}
```

Moreover, we define a callback triggered at a correct full node of the baby blockchain once it receives an information about a light client attack:
```golang
// Triggered once a full node of the baby blockchain receives LightClientAttackEvidence
func lightClientAttackEvidenceSubmitted(ev LightClientAttackEvidence) {
  evidences := isolateMisbehavingProcesses(ev)
  submitLightClientAttackEvidence(evidences, parentBlockchainFullNodes)
}
```
- Expected precondition
  - `LightClientAttackEvidence` received
- Expected postcondition
  - Array of `Evidence` submitted to each full node in `parentBlockchainFullNodes`
- Error condition
  - If the precondition is violated

Lastly, we define a callback triggered at a correct full node of the parent blockchain once it receives an array of evidences (`Evidence`):
```golang
// Triggered once a full node of the parent blockchain receives an array of Evidence from a full node of the baby blockchain
func evidenceOfmisbehaviorsSubmitted(evidences []Evidence) {
  stakingModule.processEvidences(evidences)
}
```
- Expected precondition
  - Array of `Evidence` received
- Expected postcondition
  - `stakingModule.processEvidences()` invoked
- Error condition
  - If the precondition is violated

We now just present the preconditions and postconditions of `stakingModule.processEvidences()` function:
```golang
func stakingModule.processEvidences(evidences []Evidence)
```
- Expected precondition
  - Array of `Evidence` received by a full node
- Expected postcondition
  - The input array of `Evidence` is eventually committed on the blockchain
- Error condition
  - If the precondition is violated

![image](../images/evidence_handling_1.PNG)

#### Committed Evidence Scenario

We now consider the protocol that is executed once an evidence of a misbehavior is committed on the baby blockchain.
The evidence is simply transferred to the parent blockchain via IBC.
<br> **Remark:** We do not define all the functions and callbacks needed for an IBC communication to be established between two blockchains.
For more details, please see: [Validator Change Protocol](https://github.com/informalsystems/cross-chain-validation/blob/main/spec/valset-update-protocol.md).

The following callback is triggered once there exists an evidence of a misbehavior committed on the baby blockchain:
```golang
// Invoked once there is an evidence of a misbehavior committed on the baby blockchain
func evidenceCommitted(evidence Evidence) {
  // create the CommittedEvidencePacket
  CommittedEvidencePacket packet = CommittedEvidencePacket{evidence}

  // obtain the destination port of the parent blockchain
  destPort = getPort(parentChainId)

  // send the packet
  handler.sendPacket(packet, destPort)
}
```
- Expected precondition
  - Begin-Block method is executed for the block `b`
  - Evidence `evidence` is committed in `b`
- Expected postcondition
  - Packet containing information about `evidence` is created
- Error condition
  - If the precondition is violated

Moreover, the function below is triggered once a CommittedEvidencePacket is received on the parent blockchain:
```golang
// Executed at the parent blockchain to handle a delivery of the IBC packet; in this case it is exclusively a CommittedEvidencePacket
func onRecvPacket(packet: Packet) {
  // the packet is of CommittedEvidencePacket type
  assert(packet.type = CommittedEvidencePacket)

  // inform the staking module of the new evidence
  stakingModule.submitEvidence(evidence)

  // construct the default acknowledgment
  ack = defaultAck(CommittedEvidencePacket)
  return ack
}
```
- Expected precondition
  - The `CommittedEvidencePacket` is sent to the parent blockchain previously
  - The received packet is of the `CommittedEvidencePacket` type
- Expected postcondition
  - The evidence from the received packet is submitted to the staking module
  - The default acknowledgment is created
- Error condition
  - If the precondition is violated

Lastly, we describe the `submitEvidence()` function of the staking module of the parent blockchain that is responsible for "punishing" the misbehaving validator:
```golang
func stakingModule.submitEvidence(evidence Evidence)
```
- Expected precondition
  - `onRecvPacket` function invoked because of the reception of a packet of the `CommittedEvidencePacket` type
- Expected postcondition
  - Validator from the `CommittedEvidencePacket` is slashed
- Error condition
  - If the precondition is violated

Note the difference between `stakingModule.submitEvidence()` and `stakingModule.processEvidences()` functions.
Namely, once the `stakingModule.submitEvidence()` function is invoked, the evidence is already committed on the parent blockchain (because of IBC) and the slashing could take place.
<br>However, `stakingModule.produceEvidences()` is invoked while the evidence(s) is still not committed on the parent blockchain.
Therefore, this function is responsible for ensuring that the evidence(s) are eventually committed and only then the slashing takes place.

![image](../images/evidence_handling_2.PNG)
