# Signaling Errors at Bridges

Sometimes bridges just silently swallow messages and other events. This proposal
enables bridges to communicate that something went wrong and gives clients the
option to give feedback to their users.

## Proposal

Bridges might come into a situation where there is nothing more they can do to
successfully deliver an event to the foreign network they are connected to. Then
they should be able to inform the originating group of the event about this
delivery error.

This document proposes the addition of a new PDU event with type
`m.bridge_error`. It is sent by the bridge and references an event previously
sent in a room, by that marking it as “failed to deliver” for all users of a
bridge. The new event type utilizes reference aggregations ([MSC
1849](https://github.com/matrix-org/matrix-doc/blob/matthew/msc1849/proposals/1849-aggregations.md))
to establish the relation to the event its delivery it is marking as failed.
There is no need for a new endpoint as the existing `/send` endpoint will be
utilized.

Additional information contained in the event are the name of the bridged
network (e.g. “Discord” or “Telegram”) and a regex¹ describing the affected
users (e.g. `@discord_*:example.org`). This regex should be similar to the one
any Application Service uses for marking its reserved user namespace. By
providing this information clients can inform their users who in the room was
affected by the error and for which network the error occurred.

There are some common reasons why an error occurred. These are encoded in the
`reason` attribute and can contain the following types:

* `m.event_not_handled` Generic error type for when an event can not be handled
  by the bridge. It is used as a fallback when there is no other more specific
  reason.

* `m.event_too_old` When the foreign network does not support timestamp
  massaging like Matrix does, a message will – with enough time passed – fall
  out of its original context. In this case the bridge might decide that the
  event is too old and emit this error.

* `m.foreign_network_error` The bridge was doing its job fine, but the foreign
  network permanently refused to handle the event.

* `m.unknown_event` The bridge is not able to handle events of this type.

Nothing prevents multiple bridge error events to relate to the same event. This
should be pretty common as a room can be bridged to more than one network at a
time.

The need for this proposal arises from a gap between the Matrix network and
other foreign networks it bridges to. Matrix with its eventual consistency is
unique in having a message delivery guarantee. Because of this property there is
no need in the Matrix network itself to model the failure of message delivery.
This need only arises for interactions with foreign networks where message
delivery might fail. This proposal extends Matrix to be aware of these error
cases.

Additionally there might be some operational restrictions of bridges which might
make it necessary for them to refrain from handling an event, e.g. when hitting
memory limits. In this case the new event type can be used as well.

### Example

```
{
    "type": "m.room.bridge_error",
    "content": {
        "network: "Discord",
        "affected_users": "@discord_*:example.org",
        "reason": "m.event_too_old",
        "m.relates_to": {
            "rel_type": "m.reference",
            "event_id": "$some:event.id"
        }
    }
}
```

\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\
¹ Or similar – see *Security Considerations*

## Tradeoffs

Without this proposal, bridges could still inform users in a room that a
delivery failed by simply sending a plain message event from a bot account. This
possibility carries the disadvantage of conveying no special semantic meaning
with the consequence of clients not being able to adapt their presentation.

A fixed set of error types might be too restrictive to express every possible
condition. An alternative would be a free-form text for an error message. This
brings the problems of less semantic meaning and a requirement for
internationalization with it. In this proposal a generic error type is provided
for error cases not considered in this MSC.

## Potential issues

When the foreign network is not the cause of the error signaled but the bridge
itself (maybe under load), there might be an argument that responding to failed
messages increases the pressure.

## Security considerations

Sending a custom regex with an event might open the doors for attacking a
homeserver and/or a client by exposing a direct pathway to the complex code of a
regex parser. Additionally sending arbitrary complex regexes might make Matrix
more vulnerable to DoS attacks. To mitigate these risks it might be sensible to
only allow a more restricted subset of regular expressions by e.g. requiring a
maximal length or falling back to simple globbing.

## Conclusion

In this document a new permanent event is proposed which a bridge can emit to
signal an error on its side. The event informs the affected room about which
message errored for which reason; it gives information about the affected users
and the bridged network. By implementing the proposal Matrix users will get more
insight into the state of their (un)delivered messages and thus they will become
less frustrated.