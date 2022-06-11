# Nano Protocol Overview
#### When a Node first boots up it starts calculating Representative Weights and bootstrapping process. It also reconnects with all peers it has connected to before and peering.nano.org

# Connection Process
## Connecting
#### Server Entry = TCP Server (default port: 7075)
#### When a Node connects to a Server Entry of a Node, it sends a NodeIDHandshake (Message Type 10) [extension: 0b01] with a randomized Cookie [Client Cookie] (32 bytes).
#### The Server Responds with NodeIDHandshake (Message Type 10) [extension: 0b11] which haas a randomzied Cookie [Server Cookie] (32 bytes), a Node ID [Server NodeID] (Public Key) and a Signature (of The [Client Cookie] signed with the Private Key of [Server NodeID]).
#### The Client Responds with NodeIDHandshake (Message Type 10) [extension: 0b10] which contains a Node ID [Client NodeID] (Public Key) and a Signature (of the [Server Cookie] signed with the Private Key of [Client NodeID])

| Sender | Message Type | Body | Extensions |
|   --   |    --      |  --  |     --     |
| Client | NodeIDHandshake (10) | { Cookie } | 1
| Server | NodeIDHandshake (10) | { Cookie, { Node ID, Signature } } | 3
| Client | NodeIDHandshake (10) | { { Node ID, Signature } } | 2

# What happens after a connection?
#### After a connection Node sends each other a KeepAlive (list of peers) and a Confirmation Request (to identify represnetaive)
#### KeepAlive contains 8 peers. Message Size = (18 * 8)
#### Confirmation Request will be of a random confirmed block from ledger. The Root is the Previous Block of the Random Block, or the public key of the Account from the Random Block (for open blocks).
| Message Type | Body | Extensions |
|   --      |  --  |     --     |
| KeepAlive (2) | { { IPv6 Address (16 bytes), Port (UInt16LE) } (8) } | 0
| ConfirmReq (4) | { { Block Hash, Root } } | 24832 (0x1100)

# Proccesing KeepAlive
#### The node loops through the peer list
#### If peer is [::]:0 (represented as [IPv6]:port) then skip and continue loop.
#### If node already is connected or is connecting to peer then skip and continue loop.
#### If peer isn't connected then start Connection Process for that peer

# What happens when a node recieves a Confirmation Request
#### Block Count = (extensions & 0xf000) >> 12
#### The node loops through the block list
#### In the loop, it evalutes if the block requested is confirmed, if it is it will send a Confirmation Acknowledgment (Vote) of that block, if there is another winning fork it will send a Confirmation Acknowledgment (Vote) of that winning fork.
#### Timestamp = 18446744073709551615 (Final Vote) or timestamp when vote was generated.
#### Vote Hash = Blake2b("vote " + message body + timestamp)
#### Signature = Signature of Vote Hash signed with Private Key of Representative.
#### Block Hash = Block Hash of the block which was voted on.
| Message Type | Body | Extensions |
|    --      |  --  |     --     |
| ConfirmAck (5) | { { Representative, Signature, Timestamp }, { { Block Hash } } } | 5632 (0x1600)

# What happens when a node recieves a Confirmation Acknowledgment / Vote
#### Block Count = (extensions & 0xf000) >> 12
#### The Node verifies the signature with the Public Key sent in the Vote.
#### If this is response to the first Confirmation Request the connection is associated with the Representative
#### The node loops through the block list
#### If Block does not have an active Election the node checks if Representative has over 0.1% of trended weight (Princple Representative Threshold). If it does its handed over to Inactive Vote Proccesor
#### If it does have an active election its handed to Active Vote Proccesor

# Inactive Vote Proccesor (Vote Hinting)
#### If a Vote Entry does not exist for Block Hash it is initalized.
#### If representative has voted on this block then don't process.
#### It adds the representative to the voted list and adds Online Weight of the representative to the block entry.
#### If the voted list is equal to or great than 10 and online weight is equal to 10% of trended weight (Vote Hint Threshold) a election is started on the block in the state of Passive.

# Active Vote Proccesor
#### If representative has voted on this block then don't process.
#### It adds the representative to the voted list and adds Online Weight of the representative to the block entry.
#### If the online weight is equal to 67% of trended weight (Confirmation Quorum) the block is added to the Confirmation Solliciter.

# Election States

|     State    |                    Condition                    |
|     :--:     |                       ---                       |
|    Passive   | Default State                                   |
|    Active    | Election started for >5s ago                    |
| Broadcasting | More than 2 requests has been made for election |
|    Expired   | Election started for >300s ago                  |

# Election Loop
#### Every 0.5 seconds the Node iterates through active elections
#### If election is passive or expired then skip
#### Send a Confirmation Request to top 50 Associated Representatives which hasn't voted on a succesor of root.
#### If election is in Broadcasting state then publish the block to the top 50 Associated Representatives.

# Block Proccesor Result

|     Result    |                    Condition                    |
|     :--:     |                       ---                       |
|    Progress   | All checks was succesful                                   |
|    Gap Previous    | Previous Block doesn't Exist.                    |
| Gap Source | Link Block Doesn't Exist. |
| Gap Epoch Open Pending | Unknown |
|    Old   | Block already exists.                  |
|    Bad Signature   | Signature isn't valid.                  |
|    Negative Spend   | Unknown                  |
|    Unreceivable   | Block has already been recieved, reciever isn't equal to claimer or block can only be clamed through state blocks |
|    Fork   | Account already exists (open blocks) or Block Hash is not equal to Account's Head |
|    Opened Burn Account   | Block belongs to burn account (nano_1111111111111111111111111111111111111111111111111111hifc8npp) |
|    Balance Mismatch   | Block's Amount does not match Block Link's Amount |
|    Representative Mismatch   | Block's Representative does not match Previous Block's Representative (epoch blocks) |

#
