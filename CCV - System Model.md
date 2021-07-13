# System Model

## Blockchains

A *blockchain* is a tamper-proof distributed sequence of *blocks* (see [Light Client Specification](https://github.com/tendermint/spec/blob/master/spec/light-client/verification/verification_002_draft.md#part-i---tendermint-blockchain)).
A block is a sequence of transactions.
Each block of a blockchain is *verified* by a set of pre-determined *full nodes*.
Moreover, each block defines a set of full nodes that have the responsibility to verify the next block in the sequence.

If a blockchain keeps appending blocks to its sequence forever, we say that the blockchain is *forever-active*.

We assume that a blockchain is associated with *UnbondingPeriod* parameter.
This parameter denotes a time period.

A validator is a tuple (id, power), where id represents a unique identifier of a full node and power denotes the voting power of the full node.
A *validator set* is a set of validators such that no two validators have the same id.
Morevoer, a *validator set update* is a validator set and it is used to describe changes to a validator set.

## Setup

We consider two blockchains: blockchain *parent* and blockchain *baby*.
These blockchains are able to communicate over an ordered IBC channel.
We say that the ordered IBC channel is in one of two states:
- open: The packets sent by the parent to the baby blockchain could eventually be received.
All the packets received by the baby blockchain are received in the same order in which they were sent.
Moreover, the received packets could be acknowledged by the baby blockchain (the packets are acknowledged in the order they were sent).
If the channel is open, then it can be in one of the two substates:
    - open-active: The channel is operating properly, i.e., sent and acknowledged packets are eventually received by the other blockchain. 
    - open-inactive: The channel is open, but it does not provide the transit of packets between two blockchains.
<!-- - active: The channel is operating properly, i.e., the packets sent by the parent to the baby blockchain are eventually received and they are received in the order in which they were sent. Moreover, the received packets could be acknowledged by the baby blockchain (the packets are acknowledged in the order they were sent). -->
- closed: The channel can transit from open to closed state at any point in time (the opposite is not true).
Once the channel is in the closed state, it does not provide transit of packets between two blockchains.
Moreover, once the channel fail, it never restores to the active state.

If the channel transits to open-active state infinitely many times, we say that the channel is *forever-active*.

### Discussion About Channel Abstraction

We have previously mentioned that parent and baby blockchains are able to communicate among an IBC channel.
As it can be seen in the next section, liveness properties we ensure rely on the fact that the channel is forever-active.
Indeed, if the channel is not forever-active, liveness properties cannot be ensured.

Importantnly, IBC relies on timeouts to signal that the sent packet is not going to be received on the other blockchain.
Once an ordered IBC channel timeouts, the channel is closed.
In the CCV protocol, the IBC channel used between parent and baby blockchains **cannot** ever timeout.
This implies that the channel never transits to the closed state.

Another important point is that the state of the IBC channel could be "enforced" to become open-active at any time.
Namely, whether the channels transfers packets from one blockchain to another depends on the **relayer**.
If the relayer works properly, the packets are successfully transferred.
Importantly, any validator could play the role of the relayer, which ensures that the channel can transit into open-active state at any time (if there exists a validator that aims to successfully relay packets). 

# Problem Definition

The CCV module exposes the following interface:
- Request \<ChangeValidatorSet, ValidatorSetUpdate update\>: demands that the validator set of the baby blockchain reflects update; invoked only by the registry of the parent blockchain.
- Indication \<MatureUpdate, ValidatorSetUpdate update\>: indicates that update has matured; triggered only on the parent blockchain.
- Indication \<ValidatorSetUpdate, ValidatorSetUpdate update\>: indicates that update should be applied to the validator set; triggered only on the baby blockchain.

We assume that no two identical <ChangeValidatorSet> operations are invoked.

## Properties

Now, we define the properties of the CCV module:

- **Safety - Parent:** <MatureUpdate, update> is not triggered unless <ChangeValidatorSet, update> has been previously invoked.

- **Unbonding Safety:** If <ValidatorSetUpdate, update> is triggered at time T, then <MatureUpdate, update> is not triggered before T + UnbondingPeriod.

- **Order Preservation - Parent:** If <MatureUpdate, update> is triggered before <MatureUpdate, update'>, then <ChangeValidatorSet, update> is invoked before <ChangedValidatorSet, update'>.

- **Safety - Baby:** <ValidatorSetUpdate, update> is not triggered unless <ChangeValidatorSet, update> has been previously invoked.

- **Order Preservation - Baby:** If <ValidatorSetUpdate, update> is triggered before <ValidatorSetUpdate, update'>, then <ChangeValidatorSet, update> is invoked before <ChangeValidatorSet, update'>.

- **Liveness - Parent:** Let <ChangeValidatorSet, update> be invoked.
If the channel and both blockchains are forever-active, then eventually <MatureUpdate, update> is triggered.

> "forever-active" means that no packet times out. That is there is an active relayer. If a validator wants liveness, then it should run a relayer.

- **Liveness - Baby:** Let <ChangeValidatorSet, update> be invoked.
If the channel and both blockchains are forever-active, then eventually <ValidatorSetUpdate, update> is triggerred.
