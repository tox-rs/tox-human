Title: Packets of messenger

This is a description of packets of messenger layer of Tox.

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
`0..1007`  | UTF8 byte string

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
Payload of msi packet consists of three kind of commands: request, error, capabilities.
- Request does init_call, push call(start call) or pop call(end call).
- Error is used to handle errors during processing requests.
- Capabilities hold the ability of node related with audio, video.


##### MSIRequest

```
enum MSIRequest {
    REQU_INIT,
    REQU_PUSH,
    REQU_POP,
}
```

##### MSIError

```
enum MSIError {
    MSI_E_NONE,
    MSI_E_INVALID_MESSAGE,
    MSI_E_INVALID_PARAM,
    MSI_E_INVALID_STATE,
    MSI_E_STRAY_MESSAGE,
    MSI_E_SYSTEM,
    MSI_E_HANDLE,
    MSI_E_UNDISCLOSED,
}
```

##### MSICapabilities

```
enum MSICapabilities {
    MSI_CAP_S_AUDIO = 4,  // sending audio
    MSI_CAP_S_VIDEO = 8,  // sending video
    MSI_CAP_R_AUDIO = 16, // receiving audio
    MSI_CAP_R_VIDEO = 32, // receiving video
}
```

##### MSICallState

```
enum MSICallState {
    MSI_CALL_INACTIVE,      // Default
    MSI_CALL_ACTIVE,
    MSI_CALL_REQUESTING,    // when sending call invite
    MSI_CALL_REQUESTED,     // when getting call invite
} MSICallState;
```

Protocol: |kind [1 byte]| |size [1 byte]| |value [$size bytes]| |...{repeat}| |0 {end byte}|

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x45`
`0..255`  | payload

Though protocol defines `size` field, actually every commands's size of `value` is 1 byte.
So, `size` is for the future extension.

This packet structure can permit for a node to send multiple commands in one packet.
For example a node can send packet to a friend to start call with capabilities and errors.
The node receiving this packet also can do multiple action on commands in a packet.
Because `payload` size is 255 bytes long, we can send maximum 255 / 3 = 85 commands in a packet.
But if there are same kind of commands in a packet, the last command will replace the previous commands.

On receiving this packet, a node does:

- parse packet: parse it to store commands to data structures.
    - Data structure which holds the result of parsing has 3 fields
        - `Request` holds request command
        - `Error` holds error during processing commands.
        - `Capabilities` holds the abilities related with audio or video of a friend.
    - If there are same kind of commands in a packet, the last command will replace the previous one.
- Process the `Request` command in the packet.
    - Init: init a msi sesstion.
        - Check if `Capaboilites` is empty, if it is then return with error(`MSI_E_INVALID_MESSAGE`).
        - If call_state is `MSI_CALL_INACTIVE` then request a call to a friend.
        - If call_state is `MSI_CALL_ACTIVE` then sends packet containing capabilities of us. This is the situation of a friend re-call us,
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

- creates a new message object which is one of these:
    - Init
    - Push
    - Pop
- adds error if there is an error
- adds capabilites of us
- makes a packet and sends it using `NetCrypto`
