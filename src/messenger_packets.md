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
Multiple calling file sending function of messenger must be done by the user of tox-rs, Tox-rs provides only an api for sending a chunk of file data.
Also, assembling chunks of file data received by tox-rs must be done by the user of tox-rs.
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
    - checks if the position is u64, means position is 8 bytes long
        - else error because expected 8 bytes
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

It checks if the action message is null, if null then do nothing else sends the action message to the friend.

## MESSAGE

This packet is used to transmit sender's message to a friend.

Serialized form:

Length    | Content
--------- | ------
`1`       | `0x40`
`0..1372` | UTF8 byte string

It checks if the message is null, if null then do nothing else sends the message to the friend.

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
