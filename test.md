Sismo is a protocol for the creation of zero-knowledge attestations. More specifically, a proof (zero-knowledge or not) is submitted by a user to an attester. The attester verifies the proof and certifies its validity (on a registry and by minting a soulbound token), so that other parties can trust the user’s claim without needing to formally verify the proof themselves.

As an example, Alice can prove that she voted in a DAO vote, without revealing what she voted for.

Built on four modular concepts: 

1. **Groups of Accounts (Available Data)**: Data source for attestations
2. **Attesters (Smart Contracts)**: Issuers of attestations
    1. *Verify Request*
    2. *Build Attestations*
    3. *Generate Attestations*
3. **Attestations Registry (Smart Contract):** Store of attestations
    1. *Attestations Collection*
    2. *Registry Governance*
4. **Badges (ERC1155):** Non Transferrable Token (aka soulbound) representation of attestations

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/360938fe-d5b3-44f4-b416-7da1053aa225/Untitled.png)

Let us explain each of the concepts in more detail.

# Groups of accounts:

In this context, an account is a pair (account ID, value). Examples of groups:

- ENS DAO voters. Account values = number of votes
- Early Ethereum users. Account values = number of transactions made before 2018

Attesters issue attestations from **claims about group membership** that are made by users. (Example: I made more than 50 transactions on Ethereum before 2018)

How are groups generated?

- Sismo has built an off-chain infrastructure that periodically generates off-chain groups. Groups are stored periodically off-chain as cells in a bi-dimensional data structure (see diagram below: rows represent each different group instance and columns represent the timestamps at which they are captured) These off-chain groups are reusable—they can be used to make available groups for several attesters.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7d3fbe02-b3a3-4eba-8b69-e49b74da7943/Untitled.png)

- To validate these groups, a Merkle tree can be constructed, and only the Merkle root needs to be posted on-chain!
- **A concern: these groups are not generated in a trustless way.** “Anyone can propose a new group by opening a PR to Sismo’s data source repository.” This seems cumbersome and unscalable.
    - More specifically, there is the following censorship concern. Groups are generated off-chain by Sismo, and a Merkle root is generated out of the group data. A Merkle proof can be submitted by an account owner and verified on-chain (the verifier will check the Merkle path AND check that each leaf is indeed a member of the group using on-chain data). However, if Sismo did not include a given user in the corresponding group, their corresponding account won’t belong to the Merkle tree, and so they won’t be able to generate a proof. In other words, we can say Sismo attestations are sufficient, but not necessary, for a claim’s validity.
    - An idea: how could we improve this? Set up an “optimistic” structure, where the Merkle trees built by Sismo off-chain can be challenged and corrected.

# Attesters:

**Attesters** are core smart contracts of the Sismo protocol. They are in charge of **verifying** user requests and **issuing** attestations.

Available groups: A group is said to be **available for an attester** when the attester has access to on-chain data in the specific format that it requires. This enables the attester to validate user claims. 

User request: a user makes a claim about his belonging to a group. Then he requests (by sending his claim and a proof of his claim) an attestation from the attester, to be sent to a certain address (in form of an SBT)

What can proofs be in this context?

- No proof, simply verifying a token is being held.
- Verifying a public Merkle proof (e.g. my account belongs to an on-chain Merkle tree of DAO governance accounts).
- Verifying a zk-proof. Sismo leverages [Hydra](https://eprint.iacr.org/2021/641.pdf), a novel verifiable proving scheme, to generate a zero knowledge delegated proof of ownership. In general, it is used for anonymous Merkle proofs.
    - Example: I want to prove a source account that I own holds over a certain amount of tokens, without revealing the source account or the exact amount of tokens.
    - The soulbound token is sent to a destination account.

Remark: all the proofs implemented so far are Merkle proofs!

Once the proof submitted by the user is verified, the attester will build an attestation in the Attestation Registry.

# Attestation registry:

The **Attestations Registry** is the main smart contract of the Sismo protocol; it's in charge of **recording** all attestations issued by authorized (by Sismo governance) issuers. Attestations are grouped into **Collections.**

- Attestation Registries will be deployed on multiple chains with a focus on Ethereum L2s.

The Attestations Registry is split into **Attestations Collections**. A collection bundles owners that share some historical or reputation characteristics.

- Only one attestation is stored on-chain per user per collection. A user can update an attestation, which will override the previous attestation.

Authorized issuers:

- The **Sismo Governance** (currently a multisig owned by Sismo Genesis Team) is the owner of the Attestation Registry. The Sismo Governance is the only entity allowed to authorize/unauthorize **issuers** to record attestations in the registry.

# Badges

Badges are non-transferable (aka soulbound) ERC1155 tokens, received by a user after an attestation is issued in the Registry.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/92b03fe9-5773-487d-82fc-73336efb1477/Untitled.png)

# Remarks

We looked into Sismo in order to “get off-chain data on chain in a private and verifiable manner”. Analysis of Sismo’s approach:

- Attesters are smart contracts, so once deployed one can check whether they are verifying the correct kind of proofs.
- The attestator registry is also a smart contract—can Sismo kick out attesters from its registry? YES. “Owner of the contract, has the right to authorize/unauthorize attestations issuers”. It does not look like the owner can delete or forge attestations, though.
- For the currently-implemented case of Merkle proofs of Merkle roots stored on-chain, their approach depends on the correct computation of the Merkle root, i.e., on including all of the corresponding leaves.

In conclusion, proofs are trustless, but an honest group member may not be able to generate a proof under Sismo censorship or malfunction.

Another concern to point out is users selling zk-badges to accounts that they do not own.

- Can we think of a way to transfer the private key of the first account (in an encrypted way) to the owner of the second account?
