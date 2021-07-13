# Protocol

This document provides the specification of the CCV protocol.

## Data Structures

This section defines the data structures used to represent the state of the baby blockchain, as well as the packets exchanged by two blockchains.

### Aplication Data

#### Parent Blockchain

- pendingChanges: Keeps track of unprocessed validator set updates issued by the parent blockchain; emptied on every EndBlock.
- unbondingChanges: Keeps track of updates that are not acknowledged.

#### Baby Blockchain

- pendingChanges: Keeps track of unprocessed validator set updates; emptied on every EndBlock.
- unbondingTime: Keeps track of unbonding times of received packets.

#### Packets

- ValidatorSetUpdatePacket(update[1], ..., update[n]): Packet with a sequence of validator set updates.

## Protocol

### Parent Blockchain

```
// invoked by the parent blockchain exclusively
upon <ChangeValidatorSet, ValidatorSetUpdate update>:
    pendingChanges.append(update)
```
- Expected precondition
    - None
- Expected postcondition
    - The update is appended to the pendingChanges
- Error condition
    - None

```
upon <EndBlock>:
    // if pendingChanges is empty, do nothing
    if pendingChanges.isEmpty():
        return

    // create the packet
    ChangeValidatorSet packet = ValidatorSetUpdate(pendingChanges)

    // set unbonding changes
    unbondingChanges.add(pendingChanges)

    pendingChanges.empty()

    // send the packet
    sendPacket(babyId, packet)
```
- Expected precondition
    - EndBlock method is invoked
- Expected postcondition
    - If pendingChanges is not empty, then the ChangeValidatorSet packet with the updates from pendingChanges is created
    - If pendingChanges is not empty, then pendingChanges are added into unbondingChanges.
    - If pendingChanges is not empty, then pendingChanges is emptied
- Error condition
    - If the precondition is violated

```
upon <OnAcknowledgementPacket, ValidatorSetUpdatePacket packet>:
    // unbond the validators; JOVAN: do we need to discuss this more since it is important in general, but not relevant to our problem definition
    unbondValidators(packet.updates)

    // delete the pending changes
    unbondingChanges.delete(packet.updates)

    // trigger that all updates have matured
    for each update in packet.updates:
        trigger <MatureUpdate, update>
```
- Expected precondition
    - packet is acknowledged
- Expected postcondition
    - unbondValidators(packet.updates) is invoked
    - packet.updates are removed from unbondingChanges
    - for each update in packet.updates, \<MatureUpdate, update\> is triggered
- Error condition
    - If the precondition is violated

### Baby Blockchain

```
upon <OnRecvPacket, ValidatorSetUpdate packet>:
    // store the updates from the packet
    pendingChanges.append(packet.updates)

    // calculate and store the unbonding time for the packet
    unbondingTime = blockTime().Add(UnbondingPeriod)
    unbondingTime.add(packet, unbondingTime)
```
- Expected precondition
    - packet is received
- Expected postcondition
    - packet.updates are appended to pendingChanges
    - (packet, unbondingTime) is added to unbondingTime, where unbondingTime = UnbondingPeriod + blockTime()
- Error condition
    - If the precondition is violated


```
// function used for mature packets
function <UnbondMaturePackets>:
    // take the current time
    currentTime = blockTime()

    for each (packet, unbondingTime) in unbondingTime.sortedByUnbondingTime():
        if currentTime >= unbondingTime:
            acknowledgePacket(packet)
            unbondingTime.delete(packet, unbondingTime)
        else:
            break
```
- Expected precondition
    - <EndBlock> is triggered
- Expected postcondition
    - for each (packet, unbondingTime) where currentTime >= unbondingTime, packet is acknowledged and the tuple is removed from unbondingTime
- Error condition
    - If the precondition is violated

```
upon <EndBlock>:
    // store the pending changes
    changes = pendingChanges

    // empty pending changes
    pendingChanges.empty()

    // unbond mature packets
    UnbondMaturePackets()

    for each update in changes:
        trigger <ValidatorSetUpdate, update>
```
- Expected precondition
    - EndBlock method is invoked
- Expected postcondition
    - for every update in pendingChanges, <ValidatorSetUpdate, update> is triggered
    - pendingChanges is emptied
    - UnbondMaturePackets() function is invoked
- Error condition
    - If the precondition is violated

## Correctness

We now prove the correctness of the protocol.

- **Safety - Parent:** <MatureUpdate, update> is triggered once ValidatorSetUpdate packet is acknowledged, where update in packet.updates.
Hence, packet has been previously sent.
Thus, <ChangeValidatorSet, update> has been previously invoked.

- **Unbonding Safety:** Let <MatureUpdate, update> be triggered at time T.
This implies that a ValidatorSetUpdatePacket packet is acknowledged, where update in packet.updates.
Furthermore, this means that UnbondingPeriod elapsed on baby blockchain since packet is received.
Once first \<EndBlock\> is triggered (after packet is received), <ValidatorSetUpdate, update> is triggered; let this time be T'.
Since T' is also the time at which packet is received by the baby blockchain, we conclude that Unbonding Safety is satisfied. 

- **Order Preservation - Parent:** Let <MatureUpdate, update> be triggered before <MatureUpdate, update'>.
This means that a ValidatorSetUpdatePacket packet is acknowledged, where update in packet.updates.
Moreover, a ValidatorSetUpdate packet' is acknowledged, where update' in packet'.updates.

    We consider two possible cases:
    - packet = packet': This means that update comes before update' in packet.updates. Hence, pendingChanges has update coming before update' (because packet is "built" out of pendingChanges). Thus, <ChangeValidatorSet, update> is invoked before <ChangeValidatorSet, update'>.
    - packet != packet': This means that packet' is sent after packet (since packets are acknowledged in the order they were sent). Hence, <ChangeValidatorSet, update> is invoked before <ChangeValidatorSet, update'> because of the ordered channel.

- **Safety - Baby:** Let <ValidatorSetUpdate, update> be triggered.
This means that update in pendingChanges.
Hence, ValidatorSetUpdate packet with update in packet.updates is received.
Hence, <ChangeValidatorSet, update> has been previously invoked.

- **Order Preservation - Baby:** Let <ValidatorSetUpdate, update> be triggered before <ValidatorSetUpdate, update'>.
This means that a ValdiatorSetUpdate packet is received, where update in packet.updates.
Moreover, a ValidatorSetUpdate packet' is received, where update' in packet'.updates.

    We consider two possible cases:
    - packet = packet': This means that update comes before update' in packet.updates. Thus, <ChangeValidatorSet, update> is invoked before <ChangeValidatorSet, update'>.
    - packet != packet': This means that packet' is sent after packet (since packets are received in the order they were sent). Hence, <ChangeValidatorSet, update> is invoked before <ChangeValidatorSet, update'> because of the ordered channel.

- **Liveness - Parent:** Let <ChangeValidatorSet, update> be invoked.
Since the channel is forever-active, update is eventually received on the baby blockchain.
At that point, the packet is added to unbondingTime.
Because of the fact that the baby blockchain is also forever-active, the UnbondingPeriod eventually elapses and the packet is acknowledged.
Thus, <MatureUpdate, update> is eventually triggered.

- **Liveness - Baby:** Let <ChangeValidatorSet, update> be invoked.
Since the channel and the baby blockchain is forever-active, update is eventually received on the baby blockchain.
Hence, it is added to pendingChanges.
Once \<EndBlock\> is triggered, <ValidatorSetUpdate, update> is triggered.
