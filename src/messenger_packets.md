Title: Packets of messenger

This is a description of packets of messenger layer of Tox.

## FILE_SENDREQUEST

This packet is used to initiate transferring sender's data file or avatar file to a friend.
Toxcore doesn't accumulate file chunks, accumulating file chunks is role of client.

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x50`
`1`       | `file_id`
`4`       | `file_type`(0 = data, 1 = avatar image data)
`8`       | `file_size`
`32`      | `file_unique_id`(a random symmetric tox key)
`0.255`   | `file_name` as UTF-8 C string

`file_type` and `file_unique_id` are sent in big endian format.

## FILE_DATA

FileData packet is used to transfer sender's data file to a friend.
It holds `file_id` which is one byte long, means that a tox client can send maximum 256 files concurrently.
Also it means that two friends can send 512 files to each other concurrently.

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x52`
`1`       | `file_id`
`0..1371` | file data piece

One FileData packet can hold maximum 1371 bytes.
To send whole file which is bigger than 1371 bytes, messenger's file sending function need to be called many times.
Multiple calling file sending function of messenger must be done by the user of tox protocol, Tox-rs provides only an api for sending a chunk of file data.
Also, assembling chunks of file data received by tox protocol must be done by the user of tox protocol.
File sending module always checks the length of sent bytes by length of file to send.
When the sent bytes exceed the length of file to send, it must stop sending file data.

## FILE_CONTROL

This packet is used to control transferring sender's file to a friend.
If a peer of connection wants to pause, kill, seek or accept transferring file, it must use this packet.

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x51`
`1`       | Whether it is sending or receiving, 0 = sender, 1 = receiver
`1`       | `file_id`
`1`       | Control type: 0 = accept, 1 = pause, 2 = kill, 3 = seek
`8`       | Seek parameter which is only included when `control type` is seek(3)

When messenger layer is called by user using FILE_CONTROL, it checks:

- Sending direction value is `0` or `1` means 0 = sender, 1 = receiver, others = error
- Retrieves `file transfer object` which holds info of file transferring, if it fails then send FileControl packet to a friend to kill the session.
- `file transfer object` is managed by `file_id` and `friend_id` and `sending direction`.

FILE_CONTROL do following things:

- Accept: Accept the request of file sending from a friend. It does:
    - checks if status is not `Accepted` if it is not `Accepted` then
        - change status to `Transferring`
    - else checks:
        - if status is `Pause by friend` then toggle status of `Pause by friend`
        - else error because friend asked me to resume file transfer that wasn't paused.
    - updates `file transfer object`

- Pause: Pause the transfer. It does:
    - checks if pause status is `Paused by friend` or transfer status is not `Transferring` then
    error because friend asked me to pause transfer that is already paused.
    - toggles pause status
    - updates `file transfer object`
    
- Kill: Kill the transfer session. It does:
    - updates `file transfer object`
    - set transfer status to `None`
    - decreases count of friend's number of sending files
    
- Seek: Seek to the position. It does:
    - checks if the transfer status is `Accepted` and the sender of this packet is receiver of file data
        - else error because Seek must be used to session `Accepted` and Seek packet can only be sent by receiver to seek before resuming broken transfers.
    - checks if the position exceeds the file size, if it exceeds
        - error because seek position exceeds file size.
    - set `requested position` and `transferred position` to seek position.
- else error because invalid file control command.

## TYPING

This packet is used to transmit sender's typing status to a friend.

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x33`
`1`       | Typing status(0 = not typing, 1 = typing)

It checks if the data is one bytes, and update typing status to the value of data.

## ACTION

This packet is used to transmit sender's action message to a friend.
Here, action message is a something like an IRC action.

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x41`
`0..1372` | UTF8 byte string

It checks if the action message is empty, if it is then do nothing else sends the action message to the friend.
It is C string.

## MESSAGE

This packet is used to transmit sender's message to a friend.

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x40`
`0..1372` | UTF8 byte string

There is no zero-length or empty string allowed and if it is then do nothing.
It is C string.

## STATUS_MESSAGE

This packet is used to transmit sender's status message to a friend.
Every time a friend become online or my status message is changed,
this packet is sent to the friend or to all friends of mine.

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x31`
`0..1007` | UTF8 byte string

If packet is received then call registered callback function to change the status message of the friend and
change the status data for the friend.
It is C string.

##  USER_STATUS

This packet is used to transmit sender's status to a friend.
Every time a friend become online or my status is changed,
this packet is sent to the friend or to all friends of mine.

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x32`
`1`       | My status(0 = online, 1 = away, 2 = busy)

When a node receives this packet, it call registered callback function to change the status of the friend.

## MSI

MSI(Media Session Interface) is a protocol to manage audio or video calls to a friend(s).
Payload of a msi packet consists of three kinds of sub-packets: request, error, capabilities.
- Request does:
    - init_call.
    - push: start call or change capabilities or report error.
    - pop: end call or adopt capabilities or process error.
- Error is used to report errors during av-call setup, running, terminating.
- Capabilities hold the ability of node related with audio, video.

- One msi packet must have at least 2 sub-packets: request and capabilities(error sub-packet is optional).
    - Optional means if there is no error then msi packet need not include error sub-packet in its payload.
- When a node receives msi packet, it performs the request sub-packet with capabilities sub-packet.
    - For example, if the request is REQU_INIT and capabilities is full option then node initializes a call with capabilities of sending audio, receiving audio, sending video, receiving video.
    - During REQU_INIT, if the node's capabilities changed to audio only, then the node sends REQU_PUSH with capabilities of sending audio and receiving audio(2 sub-packets).
    - If there is an error for changing capabilities, the msi packet would include error sub-packet, so there are 3 sub-packets in payload of a msi packet(REQU_PUSH, capabilities, error).
- A node holds its capabilities in local variable, if the capabilities of received packet differ from saved value then apply changes and update local variable.
- If error sub-packet is included in received msi packet then process error.

Sub-packet: kind [1 byte], size [1 byte], value [$size bytes] : but actually size is always 1, so a sub-packet is always 3 bytes long

- kind: one of Request, Capabilities, Error
- size: the length in byte of value(always 1)
- value: enum value depending on kind

Payload: |sub_packet| |...{sub-packet}| |0|

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x45`
`0..255`  | payload

Sub-packet serialized form:

Length    | Content
--------- | ------
`1`       | kind(1 = Request, 2 = Error, 3 = Capabilities)
`1`       | size(always 1)
`1`       | value(it depends on kind)

If kind is Request then value is one of these
- REQU_INIT = 0
- REQU_PUSH = 1
- REQU_POP = 2

If kind is Error then value is one of these
- MSI_E_NONE = 0
- MSI_E_INVALID_MESSAGE = 1
- MSI_E_INVALID_PARAM = 2
- MSI_E_INVALID_STATE = 3
- MSI_E_STRAY_MESSAGE = 4
- MSI_E_SYSTEM = 5
- MSI_E_HANDLE = 6
- MSI_E_UNDISCLOSED = 7

If kind is Capabilities then value is bitwise-OR of these
- MSI_CAP_S_AUDIO = 4,  // sending audio
- MSI_CAP_S_VIDEO = 8,  // sending video
- MSI_CAP_R_AUDIO = 16, // receiving audio
- MSI_CAP_R_VIDEO = 32, // receiving video

Examples of payload are

- Request(REQU_INIT) + capabilities(audio & video) = 6 bytes
- Request(REQU_INIT) + capabilities(audio) + error = 9 bytes
- Request(REQU_PUSH) + capabilities(video) + Request(REQU_POP) + error = 12 bytes
- Request(REQU_PUSH) + Request(REQU_POP) + Request(REQU_INIT) + capabilities(audio) + error(MSI_E_SYSTEM) + error(MSI_E_HANDLE) = 18 bytes
    
This packet structure can permit for a node to send multiple sub-packets in one packet.
For example a node can send packet to a friend to start call with capabilities and errors.
The node receiving this packet also can do multiple action on sub-packets in a packet.
Because `payload` size is 255 bytes long, we can send maximum 255 / 3 = 85 sub-packets in a packet.

If there are same kind of sub-packets in a packet, the last sub-packet will replace the previous sub-packet. so the result of parsing above examples are

- Request(REQU_INIT) + capabilities(audio & video)
- Request(REQU_INIT) + capabilities + error
- Request(REQU_POP) + capabilities + error
- Request(REQU_INIT) + capabilities + error(MSI_E_HANDLE)

After parsing payload, there are only last sub-packets of each kind.

##### CallState

A node maintains internal call_state variable, call_state is one of these
- MSI_CALL_INACTIVE = 0      // Default
- MSI_CALL_ACTIVE = 1
- MSI_CALL_REQUESTING = 2    // when sending call invite
- MSI_CALL_REQUESTED = 3     // when getting call invite

On receiving this packet, a node does:

- parse packet: parse it to store sub-packets to data structures.
    - Data structure which holds the result of parsing has 3 fields
        - `Request` holds request sub-packet
        - `Error` holds error during processing sub-packet.
        - `Capabilities` holds the abilities related with audio or video of a friend.
    - If there are same kind of sub-packets in a packet, the last sub-packet will replace the previous one.
- Process the `Request` sub-packet in the packet.
    - Init: init a msi session.
        - Check if `Capaboilites` is empty, if it is then return with error(`MSI_E_INVALID_MESSAGE`).
        - If call_state is `MSI_CALL_INACTIVE` then requests a call to a friend.
        - If call_state is `MSI_CALL_ACTIVE` then sends packet containing capabilities of us. This is the situation of a friend re-call us
        but we are not terminated with previous call.
        - If call_state is `MSI_CALL_REQUESTING` or `MSI_CALL_REQUESTED` then return with error(`MSI_E_INVALID_STATE`).
    - Push: Starts a new call to a friend or change capabilities.
        - Check if `Capaboilites` is empty, if it is then return with error(`MSI_E_INVALID_MESSAGE`).
        - If call_state is `MSI_CALL_ACTIVE` then check if capabilities are changed from previous value, if it is then
        change current call's capabilities.
        - If call_state is `MSI_CALL_REQUESTING` then  starts a new call
        - If call_state is `MSI_CALL_INACTIVE` or `MSI_CALL_REQUESTED` then ignore it.
    - Pop: Ends current call
        - If there is a error command in the packet then terminates current call.
        - If call_state is `MSI_CALL_INACTIVE` then it is a impossible case. So, terminates process.
        - If call_state is `MSI_CALL_ACTIVE` then it is a hang-up of a friend. So, ends call.
        - If call_state is `MSI_CALL_REQUESTING` then it is a rejection of call by a friend. So, ends call.
        - If call_state is `MSI_CALL_REQUESTED` then it is a cancelling of a call request by a friend. So, ends call.
    
When we need to send a packet to a friend, a node does:

- creates a new object which is one of these:
    - Init
    - Push
    - Pop
- adds error sub-packet if there is an error
- adds capabilities sub-packet of us
- creates a msi packet using above object and sends it using `NetCrypto`

## ONLINE

Sent to a friend when a connection is established.

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x18`

## OFFLINE

Sent to a friend when deleting the friend. It the friend is a member of a groupchat, we show the node as `offline`.

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x19`

## NICKNAME

Used to send the nickname of the peer to others. Whenever a friend comes online, this packet should be sent or whenever my nickname is changed, this packet should be sent.

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x30`
`0..128`  | UTF-8 C string
