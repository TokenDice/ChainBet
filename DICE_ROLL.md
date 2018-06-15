# DICE ROLL (draft)

## Summary
Extending the ChainBet protocol to go beyond a simple coin flip can begin with dice.  Games like craps offer many different kinds of bets.  Here we will focus on a simple one-time dice roll.  For example, a standard 6-sided die could be rolled and Bob could choose a number at random, say "5".  Alice could then offer Bob bet odds at a 6:1 payout.  Bob can wager 1 BCH and if a 5 is rolled, he will collect 6 BCH from Alice.  If he loses, Alice will collect his 1 BCH.

## Protocol Extension

We can extend the base protocol by introducing a new bet type designating a dice roll.  It can configured to allow:

* different types ('cardinalities') of dice:  6-sided, 12-sided, 20-sided, etc
* different betting odds
* different roles (giving or taking odds)

The changes are fairly modest:

1. Alice needs advertise the bet differently, specifying the parameters of the bet.
2. The participants need to fund the bet according to the parameters chosen.
3. The main betting script should contain changes to handle the bet odds.

## Phase 1: Bet Offer Announcement

This phase is modified from the base protocol.  Here we use a different bet type (0x02) instead of the coin flip (0x01).  The presense of a different bet type will effect the "various_phase_dependent_data" that follows. In other words, the phase dependent data is not only dependent on the phase, but also on the bet type.

Alice could choose to either give odds (pay a bet multiple to Bob if he successfully guesses the outcome of a multi-sided dice roll), or take odds (the roles would reverse: Alice would win the multiple if Bob loses).

Alice can also choose the number of sides of the virtual die.

Since giving or taking odds is a binary decision, it would be a waste of space to consume an entire byte.  Thus, we shall combine the "role" field (giving or taking odds) with the "number of sides of the die" in the same byte, with the most significant bit signaling the role, and the least significant 7 bits designating the number of sides.

A value of 0 for the "role" indicates Alice is giving odds to Bob (Bob wagers 1 BCH to win 6 BCH) and a value 1 indicates Alice is taking odds.

7 bits for the number of sides allows up to a 128-sided die.

The next 2 fields in the payload designate payout and payIn amounts.  Rather than having a single amount, we need two fields since this is an assymetrical bet.  **Note that the amounts do NOT need to correspond to fair probabilities.** If we always wanted a fair bet, only a single amount field would be needed.  However, by providing the flexibility to give slightly less-than-fair bets, it incentivizes liquidity to enter the system.

Therefore, it is essential for implementations to check and handle fairness parameters in a way that makes sense for their users. 

OP_RETURN OUTPUT:

| Bytes       | Name         | Hex Value | Description  |
| ------------- |-------------| -----|-----------------|
| 1      | Phase | 0x01  | Phase 1 is "Bet Offer Announcement" |
| 1     | Bet Type | 0x02 | Denotes what kind of bet will be contructed. 0x02 for Dice Roll. |
| 1     | Role & Sides | <odds and sides> | Most significant bit designates who is giving odds; the least significant 7 bits designates the number of sides to the die |
| 8     | Payout Amount   | \<payout amount> | Payout amount in Satohis for each participant. | 
| 8     | PayIn Amount   | \<payIn amount> | PayIn amount in Satohis for each participant. | 
| 20 | Target Address | \<target> | Optional.  Restricts offer to a specific bet participant. |


 
