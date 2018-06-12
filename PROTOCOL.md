# ChainBet - (working draft). see readme for version 1

## Abstract

ChainBet is a proposed Bitcoin Cash protocol to enable on-chain betting.  This initial proposal focuses on a simple coin flip bet, but the principles could be extrapolated to enable more elaborate configurations.  The protocol consists of 2 components: a commitment scheme to enable a trustless wager, and an on-chain messaging system to facilitate communication. 

## Motivation

Since paleolithic times, humans have engaged in games of chance, and probably always will.  Blockchain technology can increase the fairness, transparency, and safety of these activities.  The Bitcoin Cash ecosystem can gain more users and more transaction volume by providing a trustless gaming mechanism.

## Summary

To perform a trustless coinflip wager, Alice and Bob should each create secret values.  If the sum of the values is even, Alice wins the bet.  If the sum of the values is odd, Bob wins the bet.  Alice and Bob will use a cryptographic  scheme where both parties can lock in the bet and reveal their secrets fairly.

In addition, there is a messaging component of the protocol so that the parties do not have to send any extraneous communication through the internet.  They can use the blockchain for everything.  Moreover, multiple parties can participate and match themselves into coinflip betting pairs, thus creating a virtual casino that functions nearly entirely on the blockchain.  

An app will still be required on the user end to use the protocol, to send transactions from the users’ addresses.
 
# Commitment Scheme

## Overview

Alice and Bob begin by each creating secret values, and then creating a hash of those values which serve as cryptographic commitments.

The scheme is based on the parties funding a multisignature address and the principle that Bob has the responsibility to claim his winnings after learning Alice’s secret.  If he does not, Alice would be able to claim a win by default based on a time lock.  The default win would be necessary since Bob would be motivated to do nothing (keep his secret a secret) once he discovers he has lost the bet.

But there is a flaw: What compels Alice to reveal the secret, knowing she would be guaranteed to win by “time out” if she doesn’t reveal it?  Note that Alice’s secret can’t be part of the multisignature script because Bob would then know it before he has committed funds.  And there is no apparent way to allow Bob to cancel the script if Alice doesn’t share her secret since he could always claim it wasn’t shared if he sees a loss.

To solve this problem, we add some extra steps prior to the **funding transaction** which moves funds to the primary multiignature address.  Essentially, Alice and Bob jointly create a transaction using both of their inputs, with Alice's input coming from a script that requires revealing her secret.  Bob's signatures should be collected first, ensuring that Alice's secret is not revealed prior to Bob committing his funds.  

In addition, we will provide a means of preventing double spend attacks with the use of escrow addresses.

## Escrow Preparation

To prepare, Alice and Bob will each set up a temporary "escrow" address to be used as a holding place before creating the funding transaction.  Both escrow addreses will require both Alice's and Bob's signatures.  Each escrow address will also have its own emergency timelock option to retrieve the funds if one of the parties stops cooperating.

## Alice Escrow Address

The main purpose of Alice's escrow address is to reveal Alice's **Secret A** when spent.  It will require both Alice and Bob's signature plus the secret.  By requiring the secret, it reveals it to Bob, thus fulfilling that part of the commitment shceme.

Alternatively, Alice can retrieve the funds unilaterally after 8 confirmations in the situation when Bob abandonds the betting process.

**Script:**
 
OP_IF "8 blocks" 

    OP_CHECKSEQUENCEVERIFY <alicePubkey> 
    
OP_ELSE 

    OP_HASH160 <AliceCommitment> OP_EQUALVERIFY 
    
    OP_2 <alicePubkey> <bobPubkey> 
    
    OP_2 OP_CHECKMULTISIG 
    
OP_ENDIF'''



## Bob Escrow Address

The main purpose of Bob's escrow address is to prevent Bob from double spending.  Once the funding transaction is created, Alice's secret will be revealed.  If Bob sees that he has a loss, he could theoretically attempt to double spend his input to the funding transaction, thereby invalidating it.  

By first moving the funds into escrow and requiring Alice's signature in addition to Bob's to spend, Bob cannot on his own attempt a doublespend.  

Of course, it is necessary for the transaction that funds the escrow account to have at least 1 confirmation before the funding transaction is attempted, because otherwise Bob could doublespend that, invalidating both itself and the child transaction (the funding transaction).

Alternatively, Bob can also retrieve his own funds unilaterally after 8 confirmations in the situation when Alice abandonds the betting process.

**Script:**

OP_IF "8 blocks" 

    OP_CHECKSEQUENCEVERIFY <bobPubkey> 
    
OP_ELSE 

    OP_2 <alicePubkey> <bobPubkey> 
    
    OP_2 OP_CHECKMULTISIG 
    
OP_ENDIF

## Why Alice's Escrow Needs Bob's Signature

Bob's escrow requires Alice's signature for the simple reason that it prevents Bob from double spending his input to the funding transaction.  However, it is not immediately obvious why Alice should also have Bob's signature, since she is not the one with the easy double spend opportunity: Once the funding transaction happens, Alice's secret is revealed and if Bob won, he could simply avoid any double spend attacks by waiting for the funding transaction to get 1 confirmation before claiming the win.  

However, this is inefficient because it requires more confirmations.  Since funding Bob's escrow account already requires waiting for a confirmation, it makes sense to use that time to prevent Alice's funds from being double spent as well.  This renders it unnecessary for Bob to wait for an additional confirmation after the funding transaction is sent before claiming a win.   
  
## OP_RETURN Communication Messages

The ChainBet protocol operates through a series of phases.  Some phases require a communication message to be sent across the network.  This is accomplished by broadcasting a BCH trasnsaction that contains an OP_RETURN output.  

Each OP_RETURN transaction will be assumed to have outputs going back to the sender's originating address.  The output amount back to the sender will simply be the entire balance (for standard UTXOs) of the sender minus the txn fee.  This approach is simple and it minimizes the impacts to the UTXO set over time, making each OP_RETURN message like a sweep transaction.

The OP_RETURN payload uses this format:

<protocol_id><version_id><phase_value><various_phase_dependent_data> 

The protocol_id is a [standard Terab 4-byte prefix](https://github.com/Lokad/Terab/blob/master/spec/opreturn-prefix-guideline.md) with a value of **0x00424554** (ASCII equivalent of "BET").

The version_id is a one-byte value that can be used to upgrade the protocol in the future.  Currently, it shall be **0x01**.

**Note that all phases of the protocol use an OP_RETURN message.  Some instead consist of a funding transaction.**

# Protocol Phases

## Phase 1: Bet Offer Announcement

d




Announcement messages indicate the intention to participate in a wager.  A nonce is used for general ordering and is set by checking the last seen nonce and incrementing.  The wager amount is also specified.

## Phase 2: Acceptance

If Alice and Bob have a “matching” set of nonces (for example 100 and 101), then they can each indicate acceptance of the wager by sending a transaction to each other containing the hash of their secret.

## Phase 3: Alice funding

This message allows Alice to tell Bob she is proceeding with the wager, and the P2SH of the address she has created in accordance with the commitment scheme.  Bob should deterministically verify this address.

## Phase 4: Bob signing

This message allows Bob to pass to Alice the signature needed for the funding transaction, in accordance with the commitment scheme.

## Phase 5: Bob resignation

If Bob realizes he lost the bet, he can message Alice to allow her to quickly claim her winnings.  If Bob loses but does not send this message, Alice will eventually claim the winnings using the timeout.

# Message Detail

## Message 1a (Alice Announcement)

OP_RETURN OUTPUT:

| Bytes       | Name          | Description  |
| ------------- |-------------| -----|
| 4     | Protocol prefix identifier | TBD |
| 1     | Version      |   Protocol can be modified in the future. |
| 6 | Nonce      |    Arbitrary sequence number |
| 1 | Modulus      |    Reduces collisions |
| 1 | Phase      |   1 indicates announcement |
| 8 | Amount      |    Bet amount in satoshis |


## Message 1b (Bob Announcement)

OP_RETURN OUTPUT:

| Bytes       | Name          | Description  |
| ------------- |-------------| -----|
| 4     | Protocol prefix identifier | TBD |
| 1     | Version      |   Protocol can be modified in the future. |
| 6 | Nonce      |    Arbitrary sequence number |
| 1 | Modulus      |    Reduces collisions |
| 1 | Phase      |   1 indicates announcement |
| 8 | Amount      |    Bet amount in satoshis |



## Message 2a (Alice Acceptance)
PRIMARY OUTPUT: MINIMAL AMOUNT SENT TO BOB

OP_RETURN OUTPUT:

| Bytes       | Name          | Description  |
| ------------- |-------------| -----|
| 4     | Protocol prefix identifier | TBD |
| 1     | Version      |   Protocol can be modified in the future |
| 6 | Nonce      |    Arbitrary sequence number |
| 1 | Modulus      |    Reduces collisions |
| 1 | Phase      |   2 indicates acceptance |
| 32 | Hash       |   Hash of secret number |
| 8 | Amount      |    Bet amount in satoshis |




## Message 2b (Bob Acceptance)
PRIMARY OUTPUT: MINIMAL AMOUNT SENT TO ALICE

OP_RETURN OUTPUT:

| Bytes       | Name          | Description  |
| ------------- |-------------| -----|
| 4     | Protocol prefix identifier | TBD |
| 1     | Version      |   Protocol can be modified in the future |
| 6 | Nonce      |    Arbitrary sequence number |
| 1 | Modulus      |    Reduces collisions |
| 1 | Phase      |   2 indicates acceptance |
| 32 | Hash       |   Hash of secret number |
| 8 | Amount      |    Bet amount in satoshis |




## Message 3 (Alice Funding)
PRIMARY OUTPUT: MINIMAL AMOUNT SENT TO BOB

OP_RETURN OUTPUT:

| Bytes       | Name          | Description  |
| ------------- |-------------| -----|
| 4     | Protocol prefix identifier | TBD |
| 1     | Version      |   Protocol can be modified in the future |
| 6 | Nonce      |    Arbitrary sequence number |
| 1 | Modulus      |    Reduces collisions |
| 1 | Phase      |   3 indicates funding |
| 32 | Hash       |   Hash of secret number |
| 20 | P2SH addr    |    Built deterministcally |



## Message 4 (Bob Signing)
PRIMARY OUTPUT: MINIMAL AMOUNT SENT TO ALICE

OP_RETURN OUTPUT:

| Bytes       | Name          | Description  |
| ------------- |-------------| -----|
| 4     | Protocol prefix identifier | TBD |
| 1     | Version      |   Protocol can be modified in the future |
| 6 | Nonce      |    Arbitrary sequence number |
| 1 | Modulus      |    Reduces collisions |
| 1 | Phase      |   4 indicates signing |
| 20 | P2SH addr    |    Built deterministcally |
| 73 | Signature | Sighash ALL ANYONECANPAY |


Note: After phase 4, no more special communication messages are required.  At this point, Alice will have received Bob’s signature and she can fully construct and broadcast the funding transaction, revealing her secret.  Bob will then claim the funds if he wins.  If he does not claim a win within the allotted time, Alice will claim the funds by default.  

Optionally (but recommended), Bob can be a gracious loser and send a resignation message revealing his secret to Alice, enhancing the user experience.



## Message 5 (Bob Resignation)
PRIMARY OUTPUT: MINIMAL AMOUNT SENT TO ALICE

OP_RETURN OUTPUT:

| Bytes       | Name          | Description  |
| ------------- |-------------| -----|
| 4     | Protocol prefix identifier | TBD |
| 1     | Version      |   Protocol can be modified in the future. |
| 6 | Nonce      |    Arbitrary sequence number |
| 1 | Modulus      |    Reduces collisions |
| 1 | Phase      |   5 indicates resignation |
| 32| Secret value    |  Actual secret after loss| 

# Use of Modulus to Reduce Collisions

Finding partners to wager is handled with the nonce.  In the case when protocol usage is modest, an Alice can send an announcement message with an even numbered nonce and a Bob can join with a corresponding (incremented) odd numbered nonce.

**In the basic usage, Alice is always assumed to be even, and the Alice nonce must always be 1 less than the Bob nonce.  (10 , 11) is valid but (11, 12) is invalid.**

If usage were to accelerate, this system would break down because of multiple participants frequently using the same nonces.

For this, we can use the Modulus field.  With a value below 3, the optional functionality is ignored.  For values 3 or higher, we use a different scheme:  Given the Modulus field **m** and nonce **n**, n  should be matched with the next highest number **v**  where n mod m = v nod m.

For example, if Modulus field is 3, then nonce 12 Alice should be paired with nonce 15 Bob, 13 pairs with 16, and so on. For Modulus field 4, nonce 12 pairs to 16, 13 pairs to 17, etc.

The general idea is that when participants are being added quickly, they should “space themselves out”.  More thought should be put into the details of how this would be implemented in practice.

# Implementation Considerations

Here is a (probably incomplete) list of thoughts and considerations:

1. Applications implementing the protocol need to look at the mempool to read the announcements.  Mempool alone (rather than going back to prior blocks) should be sufficient to find players once the protocol has some usage.

2. The highest nonce should normally be used but there needs to be logic to handle gaps in the situation when a bad actor makes a large jump.

3. Note that the use of a separate “acceptance” step can eliminate collisions.

4. When Modulus > 3 , the lower number is always assumed to be Alice.  

5. Alice always wins the bet on “even” (regardless of nonce)

6. Direct messages between participants  make blockchain scanning minimal.


7. In the case Bob disappears after Alice creates the funding address A1, she can probably still use the address in the next round.

8. Participants need to check for double spends.

9. Some data passed between participants is redundant (for example the secret hash value is passed in the funding round, but it makes implementation easier and also supports simultaneous bets with the same partners.  Things can be made more efficient in future versions.

10. Messaging address public keys will be assumed for the smart contract.

11. In the case that Alice doesn’t publish the transaction in a timely manner following Bob’s signing, Bob should sweep the funds back soon to reclaim them and  so he doesn’t have to worry about keeping his application online.  

12. Participants need to be online generally.

13. It is important that Bob can deterministically generate that which he needs.  We assume the current blockheight is used in reference to the timelocks.

14. The bet amounts need to match (obviously)

15.  The secret values chosen must be large enough to avoid rainbow table attacks.

16. optional double spend

17. op return output vs regular output

## Author

Jonald Fyookball


 
