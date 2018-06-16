# MultiPlayer (draft in progress)

# Phase 1: Bet Offer Announcement


NOTE: This unique identifier for this Bet will be the transaction id of the txn containing this phase 1 message, herein referred to as <host_opreturn_txn_id>.

OP_RETURN OUTPUT:

| Bytes       | Name         | Hex Value | Description  |
| ------------- |-------------| -----|-----------------|
| 1      | Phase | 0x01  | Phase 1 is "Bet Offer Announcement" |
| 1     | Bet Type | 0x03 | Denotes what kind of bet will be contructed. 0x03 for Multiplayer bet. |
| 8     | Amount   | \<amount> | Bet amount in Satohis for each participant. |  



# Phase 2: Bet Participant Acceptance
 
NOTE: This transaction ID of the transaction containing this phase 2 message, herein referred to as the <participant_opreturn_txn_id>.

OP_RETURN OUTPUT:

| Bytes       | Name         | Hex Value | Description  |
| ------------- |-------------| -----|-----------------| 
| 1      | Phase | 0x02  | Phase 2 is " Bet Participant Acceptance" |
| 32    | Bet Txn Id |\<host_opreturn_txn_id> | This lets Alice know Bob wants to bet. |
|33    | Bob Multi-sig Pub Key  | \<bobPubKey>| This is the compressed public key that players should use when creating the funding transaction. |

# Phase 3: Player List Announcement


| Bytes       | Name         | Hex Value | Description  |
| ------------- |-------------| -----|-----------------| 
| 1      | Phase | 0x03  | Phase 3 is " Player List Announcement" |
| 32    | Bet Txn Id |\<host_opreturn_txn_id> |This is the bet id that is needed in case Alice or Bob(s) have multiple bets going.| 
| 8-88 | Participant Txn List  | \<participant txn list>| The 8 byte tail (least significant bits) of the participant_opreturn_txn_id of each participant other than Alice will be put into a list, sorted, and concatenated. |


# Phase 4: Sign Main Funding Transaction

| Bytes       | Name         | Hex Value | Description  |
| ------------- |-------------| -----|-----------------| 
| 1      | Phase | 0x03  | Phase 4 is " Sign Main Funding Transaction" |
| 32    | Bet Txn Id |\<host_opreturn_txn_id> |This is the bet id that is needed in case Alice or Bob(s) have multiple bets going.| 
| 32 | Transaction Input Id  | \<txIn>| The coin Bob will spend to participate in the bet. |
| 1  | vOut | \<vOut> | The index of the outpoint |
| 72 | Signature | \<signature> | Signature spending Bob's funds to the main bet script. Sigtype hash ALL \| ANYONECANPAY |


# Phase 5: Sign Escrow Funding Transaction

| Bytes       | Name         | Hex Value | Description  |
| ------------- |-------------| -----|-----------------| 
| 1      | Phase | 0x03  | Phase 4 is " Sign Main Funding Transaction" |
| 32    | Bet Txn Id |\<host_opreturn_txn_id> |This is the bet id that is needed in case Alice or Bob(s) have multiple bets going.|  
| 72 | Signature | \<signature> | Signature spending funds to all escrow addresses. Sigtype hash ALL  |



Phase 6: Sign Escrow Refund Transaction

Phase 7: Reveal Secrets
