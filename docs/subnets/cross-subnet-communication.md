# Cross-Subnet Communication

Avalanche Warp Messaging (AWM) enables native cross-Subnet communication and allows [Virtual Machine
(VM)](/learn/avalanche/subnets-overview.md#virtual-machines) developers to implement arbitrary
communication protocols
between any two Subnets.

## Use Cases 

Use cases for AWM may include but is not limited to:

- Orcale Networks: Connecting a Subnet to an oracle network is a costly process. AWM makes it easy
  for oracle networks to broadcast their data from their origin chain to other Subnets.
- Token transfers between Subnets 
- State Sharding between multiple Subnets

## Elements of Cross-Subnet Communication

The communication consists of the following four steps:

![image showing four steps of cross-Subnet communication: Signing, aggregation, Delivery and Verification](/img/cross-subnet-communication.png)

### Signing Messages on the Origin Subnet

AWM is a low-level messaging protocol. Any type of data encoded in an array of bytes can be included
in the message sent to another Subnet. AWM uses the [BLS signature
scheme](https://crypto.stanford.edu/~dabo/pubs/papers/BLSmultisig.html), which allows message
recipients to verify the authenticity of these messages. Therefore, every validator on the Avalanche
network holds a BLS key pair, consisting of a private key for signing messages and a public key that
others can use to verify the signature.

### Signature Aggregation on the Origin Subnet

If the validator set of a Subnet is very large, this would result in the Subnet's validators sending
many signatures between them. One of the powerful features of BLS is the ability to aggregate many
signatures of different signers in a single multi-signature. Therefore, validators of one Subnet can
now individually sign a message and these signatures are then aggregated into a short
multi-signature that can be quickly verified.

### Delivery of Messages to the Destination Subnet

The messages do not pass through a central protocol or trusted entity, and there is no record of
messages sent between Subnets on the primary network. This avoids a bottleneck in Subnet-to-Subnet
communication, and non-public Subnets can communicate privately.

It is up to the Subnets and their users to determine how they want to transport data from the
validators of the origin Subnet to the validators of the destination Subnet and what guarantees they
want to provide for the transport.

### Verification of Messages in the Destination Subnet

When a Subnet wants to process another Subnet's message, it will look up both BLS Public Keys and
stake of the origin Subnet. The authenticity of the message can be verified using these public keys
and the signature.

The combined weight of the validators that must be part of the BLS multi-signature to be considered
valid can be set according to the individual requirements of each Subnet-to-Subnet communication.
Subnet A may accept messages from Subnet B that are signed by at least 70% of stake. Messages from
Subnet C are only accepted if they have been signed by validators that account for 90% of the stake.

Since all validators' public keys of the validators and their stake weights are recorded on the
primary network's P-chain, they are readily accessible to any virtual machine run by the validators.
Therefore, the Subnets do not need to communicate with each other about changes in their respective
sets of validators, but can simply rely on the latest information on the P-Chain. Therefore, AWM
introduces no additional trust assumption other than that the validators of the origin Subnet are
participating honestly.

## Reference Implementation

We created a Proof-of-Concept VM called [XSVM](https://github.com/ava-labs/xsvm). This VM allows for
simple AWM transfers between any two Subnets running it out-of-the-box.

