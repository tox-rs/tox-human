This is a description of packets of conference and conference version 2 of Tox.

# Current version

## INVITE_CONFERENCE

Conference can up to 4 members connected us. Each member of conference has unique random 32 bytes id.
Each conference has its conference id of 2 bytes.
It also has a one byte type identifier, the current supported types are:
```
0: text,  1: audio
```

Invite conference packets consist of invite packet and response packet.

#### Invite packet

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x60`
`1`       | `0x00`
`2`       | `conference id`
`1`       | `conference type`(0: text, 1: audio)
`32`      | `unique id`

#### Response packet

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x60`
`1`       | `0x01`
`2`       | `conference id(local)`
`2`       | `conference id to join`
`1`       | `conference type`(0: text, 1: audio)
`32`      | `unique id`

When the user of tox client want to invite a friend to a conference, it uses `Invite packet`.
The receiver who is a friend of us responds with `Response pcket`.
If the receiver of `Invite packet` do not want to join the conference, just no reply will be enough.
The sender of `Invite packet` doesn't receive `Response packet` from the friend, then inviting will be expired.

The procedure of invite is:

- find a conference using conference_id.
- assemble and send `Invite packet` using messenger and net crypto.
    
When a peer receives `Invite packet` and want to join the conference then:

- get conference id using received packet.
- join the conference
    - send `Response packet` to the peer.
    - add friend connection to the conference.
    - send peer query packet.

When a peer receives `Response packet` then:

- find conference using received packet.
- get random peer number which is not redundant, this peer number will identify the sender of this packet.
- add the peer to conference.
- add the friend connection to conference.
- send new peer message to all of the conference members.
        
## Peer online packet

As soon as the connection to the other peer is opened, a `peer online packet` is sent to the peer.
The purpose of this packet is to tell the peer that we want to establish the conference connection.

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x61`
`2`       | `conference id(local)`
`1`       | `conference type`(0: text, 1: audio)
`32`      | `unique id`

When a peer receives this packet then:

- get conference id of us using received packet's `unique id`.
- get the conference id of the sender which is in the packet's `conference id(local)`.
- if the peer is already online then
    - return.
- else
    - if the number of close connected peers are zero or the sender of this packet is the one we are introducing to conference
        - send `peer query packet`.
    - send `peer online packet` using net crypto.
    - set the status of sender to online.
    - if the sender is the one who we are introducing
        - send new peer message to all of the conference members.
    - send ping.

## Peer leave packet

This packet is sent to the peer right before killing a conference connection.

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x62`
`2`       | `conference id(local)`
`1`       | `0x01`

When a peer receives this packet then:

- kill the friend connection.
- set close list to nothing connected.

## Peer query packet

Sent to peer to query the peer list in a conference. A peer receives this packet send response packet with `response packet`.
If there are many peers in a conference, then response packet may be larger than the maximum packet size of friend connection packet(1373 bytes),
then multiple response packets will be sent.

#### Request

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x62`
`2`       | `conference id`
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
`2`       | `conference id`
`1`       | `0x09`
variable  | `peer info list`

An entry of `peer info list` is

Length    | Content
--------- | --------------------
`2`       | `peer number`
`32`      | Real PK
`32`      | Temp PK
`1`       | `length` of nickname
variable  | Nickname(UTF-8 String)

When a peer receives this packet then:

- while all entry of `peer info list` are processed
    - if conference status is `valid` and real PK is same as the net crypto's real PK
        - set status to `connected`.
        - send `invite` request packet.
    - add peer to conference using temp PK.
    - update nickname of the peer.
- loop

## Title packet

This packet is sent right after peer query packet is sent.

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x62`
`2`       | `conference id`
`1`       | `0x0a`
variable  | Title(UTF-8 C String)

When a peer receives this packet then:

- check if the title is same as existing title, if same then
    - return.
- else
    - update title of the conference.
    
## Message packet

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x63`
`2`       | `conference id`
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

Sent every 60 seconds by every peer. Contains no body.

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

- add peer to peer list.

#### Kill peer (0x11)

Serialized form:

Length    | Content
--------- | ------
`2`       | `peer number`

When a peer quit a conference, right before quit, it send this packet.

When a peer receives this packet then:

- delete peer from peer list.

#### Freeze peer (0x12)

Serialized form:

Length    | Content
--------- | ------
`2`       | `peer number`

When a peer quit running, it need to freeze conference rather than remove it.

When a peer receives this packet then:

- send rejoin packet to the conference members.
- save peer info list.
- delete peer.

#### Name packet (0x30)

Serialized form:

Length    | Content
--------- | ------
variable  | Name (UTF-8 C string)

Sent by a peer who wants to change its name or by a joining peer to notify its name to members of conference.

When a peer receives this packet then:

- if the name equal to existing name then
    - return.
- else
    - update name of the peer in the conference.
    
#### Title packet (0x31)
 
Serialized form:

Length    | Content
--------- | ------
variable  | Title (UTF-8 C string)

Sent by anyone who is member of conference.

This packet is used to change the title of conference.

When a peer receives this packet then:

- if the title equal to existing one then
    - return.
- else
    - update title of conference.
    
#### Chat message packet (0x40)

Serialized form:

Length    | Content
--------- | ------
variable  | Message(UTF-8 C string)

Sent to send chat message to all member of conference.

When a peer receives this packet then:

- show the message in the chatting window by calling callback of client.

#### Action packet (0x41)

Serialized form:

Length    | Content
--------- | ------
variable  | Message(UTF-8 C string)

Sent to send action to all member of conference.

When a peer receives this packet then:

- show the message in the chatting window by calling callback of client.


# DHT based conference(version 2)
![Conference version 2](https://github.com/iphydf/c-toxcore/blob/new-group-chats/docs/DHT-Group-Chats.md)

New version of conference is based on DHT.
It introduces many new features which are not provided by current version.
It has 4 types of member which are founder, moderator, user, observer and provides public conference, private conference.

## Lossless packet (0x5b)

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x5b`
`4`       | `hash id`
`32`      | `PK of sender`
`24`      | `nonce`
`1`       | `packet kind`
`8`       | `message id`
`4`       | `sender pk hash`
variable  | Data

`Data` depends on `packet kind`.

`packet kind` is
```
    GP_CUSTOM_PACKET            = 0xf1,
    GP_PEER_ANNOUNCE            = 0xf2,
    GP_BROADCAST                = 0xf3,
    GP_PEER_INFO_REQUEST        = 0xf4,
    GP_PEER_INFO_RESPONSE       = 0xf5,
    GP_INVITE_REQUEST           = 0xf6,
    GP_INVITE_RESPONSE          = 0xf7,
    GP_SYNC_REQUEST             = 0xf8,
    GP_SYNC_RESPONSE            = 0xf9,
    GP_TOPIC                    = 0xfa,
    GP_SHARED_STATE             = 0xfb,
    GP_MOD_LIST                 = 0xfc,
    GP_SANCTIONS_LIST           = 0xfd,
    GP_FRIEND_INVITE            = 0xfe,
    GP_HS_RESPONSE_ACK          = 0xff
```
When a peer receives this packet then:

- get the peer number using serder's PK.
- get the conference connection using peer number.
- unpack packet using conference connection's shared key.
- if `packet kind` is not GP_HS_RESPONSE_ACK and is not GP_INVITE_REQUEST and is not in state of handshaked then
    - return error.
- get sender pk hash from `data`.
- if sender pk hash is not same as conference connection's one then
    - return error.
- if `packet kind` is invite related packet type and `message id` of this packet is 3 and conference connection's message id is 1 then
    - return error because probably we missed handshake packet.
- if message id is lower than current one then 
    - it is duplicated message, so we responds with ack.
- if message id is grater than current one + 1 then
    - we lost one or more messages, so we put this message to array to save.
    - responds with ack.
- if message id equals current one + 1 then
    - we received correct message id, so process it immediately/
    - processing packet deponds on `packet kind`, it is listed below.
- get peer number and conference connection again.
- if the peer still exists and we received correct message id then
    - responds with ack.
    
#### Broadcast packet (0xf3)

Serialized form:

Length    | Content
--------- | ------
`1`       | `type`
`8`       | `timestamp`
variable  | Data

`Data` depends on `type`.

`type` is
```
    GM_STATUS               = 0x00,
    GM_NICK                 = 0x01,
    GM_PLAIN_MESSAGE        = 0x02,
    GM_ACTION_MESSAGE       = 0x03,
    GM_PRIVATE_MESSAGE      = 0x04,
    GM_PEER_EXIT            = 0x05,
    GM_REMOVE_PEER          = 0x06,
    GM_REMOVE_BAN           = 0x07,
    GM_SET_MOD              = 0x08,
    GM_SET_OBSERVER         = 0x09,
```

##### Status

Serialized form:

Length    | Content
--------- | ------
`1`       | `status`

`status` is
```
    GS_NONE         = 0x00,
    GS_AWAY         = 0x01,
    GS_BUSY         = 0x02,
    GS_INVALID      = 0x03,
```

When a peer receives this packet then:

- set status of a peer in conference
    - call callback of client to update status of a peer.
    - update variable of status of a peer in conference.
    
##### Nickname

Serialized form:

Length    | Content
--------- | ------
variable  | nickname(UTF-8 C String)

When a peer receives this packet then:

- set nickname of a peer in conference
    - call callback of client to update nickname of a peer.
    - update variable of nickname of a peer in conference.

##### Plain/Action message

Serialized form:

Length    | Content
--------- | ------
variable  | message(UTF-8 C String)

When a peer receives this packet then:

- show plain/action message in conference text window
    - call callback of client to show message.

##### Private message

Serialized form:

Length    | Content
--------- | ------
variable  | message(UTF-8 C String)

When a peer receives this packet then:

- show private message in conference text window as private mode
    - call callback of client to show message as private mode.

##### Peer exit

Body of packet is not clearly defined now.

When a peer receives this packet then:

- delete peer number from conference
    - if connection status is (`disconnected` or `connecting` or `manually disconnected`) and private conference then
        - return error.
    - if connection is handshaked but not confirmed then
        - copy PK of peer to confirmed peers list.
    - if connection is confirmed then
        - call callback to notify deleting peer to client.
    - delete tcp connection.
    - clean up peer info in conference.
    - decrease number of peers.

##### Remove peer

Serialized form:

Length    | Content
--------- | ------
`1`       | `event`
`32`      | PK of target
variable  | `sanction list`

`event` is
```
    MV_KICK         = 0x00,
    MV_BAN,         = 0x01,
    MV_OBSERVER     = 0x02,
    MV_USER         = 0x03,
    MV_MODERATOR    = 0x04,
    MV_INVALID      = 0x05,
```

An entry of `sanction list` is

Length    | Content
--------- | ------
`1`       | `type`(see below)
`32`      | PK of signature
`8`       | timestamp
variable  | Data

`Data` depends on `type`.

`type` is
```
    SA_BAN_IP_PORT      = 0x00,
    SA_BAN_PUBLIC_KEY   = 0x01,
    SA_BAN_NICK         = 0x02,
    SA_OBSERVER         = 0x03,
    SA_INVALID          = 0x04,
```

- SA_BAN_IP_PORT

Serialized form:

Length      | Content
------------| ------
`4`         | `id`
`1`         | `type`(of ip)
`4` or `16` | IPv4 or IPv6 address
`2`         | port

- SA_BAN_NICK

Serialized form:

Length      | Content
------------| ------
`4`         | `id`
`128`       | nickname

- SA_BAN_PUBLIC_KEY

Serialized form:

Length      | Content
------------| ------
`4`         | `id`
`32`        | PK of target

- SA_OBSERVER

Serialized form:

Length      | Content
------------| ------
`32`        | PK of target

When a peer receives this packet then:

- if `event` is not MV_KICK and is not MV_BAN then
    - return error.
- get target peer number using PK of target PK.
- if target peer number is 0 then
    - call callback of client which update UI.
    - delete all conference member except founder.
- if `event` is MV_BAN then
    - parse `sanction list`.
    - add sanction list.
- call callback of client to update UI.
- delete peer from conference.

##### Remove ban

Serialized form:

Length      | Content
------------| ------
`4`         | `ban_id`
variable    | `sanction list`(see above)

When a peer receives this packet then:

- parse sanction list.
- remove sanctions which is listed in sanction list.

##### Set/unset moderator

Serialized form:

Length      | Content
------------| ------
`1`         | `kind`(0 = set to user, otherwise = set to moderator)
`32`        | PK(`kind` = 0) or Moderator hash(`kind` is not 0)

When a peer receives this packet then:

- if `kind` is not `0` then
    - get peer number using moderator hash.
    - add peer to moderator list.
    - set peer's role to moderator.
- else
    - get peer number using PK.
    - remove peer from moderator list.
    - set peer's role to user.
- call callback of client to update UI.

##### Set observer

Serialized form:

Length      | Content
------------| ------
`1`         | `kind`(0 = remove sanctions, otherwise = set sanctions)
`32`        | PK
variable    | `sanction list`(see above)

When a peer receives this packet then:

- if `kind` is not 0 then
    - add sanction list to conference.
-else
    - remove sanction list from conference.
- call callback of client to update UI.

#### Peer announce (0xf2)

Serialized form:

Length      | Content
------------|-------
`32`        | PK of peer
`1`         | `flag`(of ip port is setted)
`1`         | `count`(of tcp relays)
`1`         | `type`(of ip)
`4` or `16` | IPv4 or IPv6 address(comes only when `flag` is setted)
`2`         | port(comes only when `flag` is setted)
variable    | tcp relay list

An entry of `tcp relay list` is

Length      | Content
------------|-------
`1`         | `type`(of ip)
`4` or `16` | IPv4 or IPv6 address
`2`         | port
`32`        | PK of peer

When a peer receives this packet then:

- if the peer is in sanction list then
    - return error.
- add peer to conference with PK and Ip, Port.
- add and save tcp relays to conference data structures.

#### Peer info request (0xf4)

This packet has no data body.

When a peer receives this packet then:

- send self info to peer.

#### Peer info response (0xf5)

Serialized form:

Length      | Content
------------|-------
`32`        | `password`(comes only when password is setted)
`2`         | `length` of nickname
`128`       | `nickname`(UTF-8 string)
`1`         | `status`
`1`         | `role`

When a peer receives this packet then:

- updatee info of the peer.
- validate role of peer, if it is error then
    - return error.
- call callback of client to update UI.

#### Sync Request (0xf8)

Serialized form:

Length      | Content
------------|-------
`32`        | `password`(comes only when password is setted)

When a peer receives this packet then:

- send shared state.
- send peer moderator list.
- send peer topic.
- send response packet.

#### Sync response (0xf9)

Serialized form:

Length      | Content
------------|-------
`4`         | `number`(of peers)
variable    | `announce list`


An entry of `announce list` is

Length      | Content
------------|-------
`32`        | PK of peer
`1`         | `flag`(of ip port is setted)
`1`         | `count`(of tcp relays)
`1`         | `type`(of ip)
`4` or `16` | IPv4 or IPv6 address(comes only when `flag` is setted)
`2`         | port(comes only when `flag` is setted)
variable    | `tcp relay list`

An entry of `tcp relay list` is

Length      | Content
------------|-------
`1`         | `type`(of ip)
`4` or `16` | IPv4 or IPv6 address
`2`         | port
`32`        | PK of peer

When a peer receives this packet then:

- parse announce list.
- add peers to conference.
- add and save tcp relays to conference.
- if `flag` of ip port is setted and tcp relays count > 0 then
    - send handshake packet to peer.
- set myself to `connected` status.
- call callback of client to update UI.

#### Invite request (0xf6)

Serialized form:

Length      | Content
------------|-------
`2`         | `length`(of nickname)
variable    | `nickname` of length `length`(UTF-8 string)
`32`        | `password`(comes only when password is setted)

When a peer receives this packet then:

- get peer number using nickname, if can't get
    - return error.
- if the peer is banned then
    - return error.
- if `password` is setted in conference then
    - check the password is same, if not
        - return error.
- send response packet.

#### Invite response (0xf7)

No message body.

When a peer receives this packet then:

- send sync request packet.

#### Topic (0xfa)

Serialized form:

Length      | Content
------------|-------
`64`        | `signature`
`2`         | `length`(of topic)
variable    | `topic` of length `length`(UTF-8 string)
`32`        | `PK of signature`
`4`         | `version`

When a peer receives this packet then:

- veryfy signature, if error then
    - return error.
- if the `topic` is same with current one then 
    - return.
- else
    - call callback of client to update UI.

#### Shared state (0xfb)

Serialized form:

Length      | Content
------------|-------
`64`        | `signature`
`32`        | `PK of founder`
`4`         | `max peers`
`2`         | `length`(of conference name)
`48`        | `conference name`(UTF-8 String)
`1`         | `privacy state`
`2`         | `password length`
`32`        | `password`
`32`        | `moderation hash`
`4`         | `version`

When a peer receives this packet then:

- validate shared state.
- update shared state.

#### Mod list (0xfc)

Serialized form:

Length      | Content
------------|-------
`2`         | `number`(of moderators)
variable    | `mod list`


An entry of `mod list` is

Length      | Content
------------|-------
`32`        | `PK of signature`

When a peer receives this packet then:

- parse packet.
- update mod list.
- check if mod hash is same with packet's one
    - if not, return error.
- check role of us, if it fails then
    - set us to user.

#### Sanctions list (0xfd)

Serialized form:

Length      | Content
------------|-------
`4`         | `number`(of sanctions)
variable    | `sanction list`

An entry of `sanction list` is

Length    | Content
--------- | ------
`1`       | `type`(see below)
`32`      | PK of signature
`8`       | timestamp
variable  | Data

`Data` depends on `type`.

`type` is
```
    SA_BAN_IP_PORT      = 0x00,
    SA_BAN_PUBLIC_KEY   = 0x01,
    SA_BAN_NICK         = 0x02,
    SA_OBSERVER         = 0x03,
    SA_INVALID          = 0x04,
```

- SA_BAN_IP_PORT

Serialized form:

Length      | Content
------------| ------
`4`         | `id`
`1`         | `type`(of ip)
`4` or `16` | IPv4 or IPv6 address
`2`         | port

- SA_BAN_NICK

Serialized form:

Length      | Content
------------| ------
`4`         | `id`
`128`       | nickname

- SA_BAN_PUBLIC_KEY

Serialized form:

Length      | Content
------------| ------
`4`         | `id`
`32`        | PK of target

- SA_OBSERVER

Serialized form:

Length      | Content
------------| ------
`32`        | PK of target

When a peer receives this packet then:

- validate sanctions, if error then
    - return error.
- adopt sanctions to peers.

#### Handshake response ack (0xff)

No message body.

When a peer receives this packet then:

- send invite request packet to peer.

#### Custom packet (0xf1)

Serialized form:

Length      | Content
------------| ------
variable    | `user data`(defined by user)

When a peer receives this packet then:

- call callback of client to pass `user data`.

## Lossy packet (0x5c)
   
Serialized form:

Length    | Content
--------- | ------
`1`       | `0x5c`
`4`       | `hash id`
`32`      | `PK of sender`
`24`      | `nonce`
`1`       | `packet kind`
`4`       | `sender pk hash`
variable  | Data

`Data` depends on `packet kind`

`packet kind` is
```
    GP_PING                     = 0x01,
    GP_MESSAGE_ACK              = 0x02,
    GP_INVITE_RESPONSE_REJECT   = 0x03,
    GP_TCP_RELAYS               = 0x04,
    GP_CUSTOM_PACKET            = 0xf1,
```

#### Message ack (0x02)

Serialized form:

Length    | Content
--------- | ------
`8`       | `read id`
`8`       | `request id`

When a peer receives this packet then:

- re-send requested packet.

#### Ping (0x01)

Serialized form:

Length    | Content
--------- | ------
`4`       | `num peers`
`4`       | `state version`
`4`       | `screds version`
`4`       | `topic version`

- send sync request packet.

#### Invite response reject (0x03)

Serialized form:

Length    | Content
--------- | ------
`1`       | `type`

When a peer receives this packet then:

- call callback of client to update UI.

#### Tcp relays (0x04)

Serialized form:

Length      | Content
----------- | ------
`variable`  | `tcp relays`

An entry of `tcp relays` is

Length      | Content
------------|-------
`1`         | `type`(of ip)
`4` or `16` | IPv4 or IPv6 address
`2`         | port
`32`        | PK of peer

When a peer receives this packet then:

- add tcp relays to conference connection.

#### Custom packet (0xf1)

Serialized form:

Length      | Content
------------| ------
variable    | `user data`(defined by user)

When a peer receives this packet then:

- call callback of client to pass `user data`.

## Handshake packet (0x5a)

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x5a`
`4`       | `hash id`
`32`      | `PK of sender`
`24`      | `nonce`
`1`       | `packet kind`
`4`       | `sender pk hash`
variable  | Data

`Data` depends on `packet kind`

`packet kind` is
```
    GH_REQUEST      = 0x00,
    GH_RESPONSE     = 0x01,
```

#### Handshake Request (0x00)

Serialized form:

Length    | Content
--------- | ------
`32`      | `Enc PK`
`32`      | `Sig PK`
`1`       | `request type`
`1`       | `join type`
`4`       | `state version`
variable  | `nodes`

An entry of `nodes` is

Length      | Content
------------|-------
`1`         | `type`(of ip)
`4` or `16` | IPv4 or IPv6 address
`2`         | port
`32`        | PK of peer

#### Handshake response (0x01)

Serialized form:

Length    | Content
--------- | ------
`32`      | `Enc PK` 
`32`      | `Sig PK`
`1`       | `request type`
variable  | Data

Data depends on `request type`

`request type` is
```
    HS_INVITE_REQUEST       = 0x00,
    HS_PEER_INFO_EXCHANGE   = 0x01,
```

##### Handshake invite request (0x00)

Length    | Content
--------- | ------
`1`       | padding 
`4`       | `state version`

When a peer receives this packet then:

- if `state version` is older than current version or (`state version` is same and PK differs) then
    - return.
- send conference invite request packet.

##### Peer info exchange (0x01)

No message body

When a peer receives this packet then:

- send peer info response packet.
- send peer info request packet.
