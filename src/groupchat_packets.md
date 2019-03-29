Title: Packets of group chat and group chat v2.

This is a description of packets of group chat and version 2 of Tox.

# Current version

## INVITE_GROUPCHAT

Group chat can up to 4 members connected us. Each member of group chat has unique random 32 bytes id.
Each group chat has its group number of 2 bytes.
It also has a one byte type identifier, the current types are:
```
0: text,  1: audio
```

Invite group chat packets are consisted by invite packet and response packet.

#### Invite packet

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x60`
`1`       | `0x00`
`2`       | `group number`
`1`       | `group type`(0: text, 1: audio)
`32`      | `unique id`

#### Response packet

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x60`
`1`       | `0x01`
`2`       | `group number(local)`
`2`       | `group number to join`
`1`       | `group type`(0: text, 1: audio)
`32`      | `unique id`

When the user of tox client want to invite a friend to a conference, it uses `Invite packet`.
The receiver who is a friend of us responds with `Response pcket`.

The procedure of invite is:

- find a group using group_number.
- if the group doesn't exist
    - return err
- else if the group status is not `connected`
    - return err
- else
    - assemble and send `Invite packet` using messenger and net crypto
    
When a peer receives `Invite packet` then:

- Check if the packet length is correct, if it is not correct
    - return err
- else
    - get group number using received packet
    - join the group
        - send `Response packet` to the peer
        - add friend connection to the group
        - send peer query packet

When a peer receives `Response packet` then:

- Check if the packet size is correct, if it is not correct
    - return err
- else
    - find group using received packet
    - if can't find group
        - return err
    - else if group type is not equal to group of us
        - return err
    - else if the group id is not correct
        - return err
    - else
        - get random peer number which is not redundant
        - add the peer to group chat
        - add the friend connection to group chat
        - send new peer message to all of the group members
        

## Peer online packet

As soon as the connection to the other peer is opened, a `peer online packet` is sent to the peer.
The purpose of this packet is to tell the peer that we want to establish the groupchat connection.

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x61`
`2`       | `group number(local)`
`1`       | `group type`(0: text, 1: audio)
`32`      | `unique id`

When a peer receives this packet then:

- get group number of us using received packet's `unique id`
- get the group number of the peer us which is in the packet's `group number(local)`
- if the peer is already online then return else:
    - if the number of close connected peers are zero or the peer is the one we are introducing to conference
        - send `peer query packet`
    - send `peer online packet` using net crypto
    - set the status of peer to online
    - if the peer is the one who we are introducing
        - send new peer message to all of the group members
    - send ping

## Peer leave packet

This packet is sent to the peer right before killing a group connection.

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x62`
`2`       | `group number(local)`
`1`       | `0x01`

When a peer receives this packet then:

- kill the friend connection
- set close list to nothing connected

## Peer query packet

Sent to peer to query the peer list in a group chat. A peer receives this packet send response packet with `response packet`.
If there are many peers in a group chat, then response packet may larger than the maximum packet size of friend connection packet(1373 bytes),
then multiple response packets will be sent.

#### Request

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x62`
`2`       | `group number`
`1`       | `0x08`

When a peer receives this packet then:

- assembles `response` packet and if the size of response is larger than maximum size then
sends multiple `response` packets.
- assembles `title` request packet and sends packet to peer.

#### Response

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x62`
`2`       | `group number`
`1`       | `0x09`
variable  | Peer info.s

Peer info. is

Length    | Content
--------- | --------------------
`2`       | `peer number`
`32`      | Real PublicKey
`32`      | Temp PublicKey
`1`       | Length of nickname
variable  | Nickname

When a peer receives this packet then:

- while all peer info.s are processed
    - if group status is `valid` and real PK is same as the net crypto's real PK
        - set status to `connected`
        - send `invite` request packet
    - add peer to group using temp PK
    - update nickname of the peer
- loop

## Title packet

This packet is sent right after peer query packet is sent.

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x62`
`2`       | `group number`
`1`       | `0x0a`
variable  | Title

When a peer receives this packet then:

- check if the title is same as existing title, if same then
    - return
- else
    - update title of the group chat
    
## Message packet

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x63`
`2`       | `group number`
`2`       | `peer number`
`4`       | `message number`
`1`       | `message kind`
variable  | Data

`message kind` is
```
    PING            = 0x00,
    NEW_PEER        = 0x10,
    KILL_PEER       = 0x11,
    FREEZE_PEER     = 0x12,
    NAME            = 0x30,
    TITLE           = 0x31,
    CHAT MESSAGE    = 0x40,
    ACTION          = 0x41,
```

`Data` varies depending on `message kind`.

#### Ping (0x00)

Sent every 60 seconds by every peer. Contains no data

#### New peer (0x10)

Tell everyone about a new peer in the chat.
The peer who invited joining peer sends this packet to warn everyone that there is a new peer.

Serialized form:

Length    | Content
--------- | ------
`2`       | `peer number`
`32`      | Long term PK
`32`      | DHT PK


When a peer receives this packet then:

- add peer to peer list

#### Kill peer (0x11)

Serialized form:

Length    | Content
--------- | ------
`2`       | `peer number`

When a peer quit a group chat, before quit, it send this packet.

When a peer receives this packet then:

- delete peer from peer list

#### Freeze peer (0x12)

Serialized form:

Length    | Content
--------- | ------
`2`       | `peer number`

When a peer quit running, it need to freeze group chat rather than remove it.

When a peer receives this packet then:

- send rejoin packet to the group members
- save peer info.
- delete peer

#### Name packet (0x30)

Serialized form:

Length    | Content
--------- | ------
variable  | Name (UTF-8 C string)

Sent by a peer who wants to change its name or by a joining peer to notify its name to members of group chat.

When a peer receives this packet then:

- If the name equal to existing name then
    - return
- else
    - update name of the peer in the group chat
    
 #### Title packet (0x31)
 
Serialized form:

Length    | Content
--------- | ------
variable  | Title (UTF-8 C string)

Sent by anyone who is member of group.

This packet is used to change the title of group chat.

When a peer receives this packet then:

- If then title equal to existing title then
    - return
- else
    - update title of group chat
    
#### Chat message packet (0x40)

Serialized form:

Length    | Content
--------- | ------
variable  | Message (UTF-8 C string)

Sent to send chat message to all member of group chat.

When a peer receives this packet then:

- Show the message in the chatting window.

#### Action packet (0x41)

Serialized form:

Length    | Content
--------- | ------
variable  | Message (UTF-8 C string)

Sent to send action to all member of group chat.

When a peer receives this packet then:

- Show the message in the chatting window.


# Version 2

