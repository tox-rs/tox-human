Title: Onion client (UDP)

This is description of onion layer in Tox.

## Paths

Onion routing can be described using this chart:

<object data="static/path.svg" type="image/svg+xml"></object>

To send a message via an onion path, first we construct the path by choosing three random nodes. Then, we put the message in several layers:

1. `OnionRequest2` contains the address of the message receiver and the message encrypted with the third's node key 
2. `OnionRequest1` contains the address of the second node and `OnionRequest2` encrypted with the second's node key, and
3. `OnionRequest0` contains the address of the first node and encrypted `OnionRequest0`

A path node receives a message, decrypts the payload, attaches `OnionReturn` and sends the result to the next node in the path.

An `OnionReturn` of the n'th node is the pair of node's `IP_Port` and `OnionReturn` of the previous node (if it exists), encrypted with the n'th node's key. It allows the receiver to send a responce to the sender using the same onion path.

## Client announce

<object data="static/announce_start.svg" type="image/svg+xml"></object>

