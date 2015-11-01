#Amiko Pay mid-level communication protocol

Note 1: this information is to be moved to other documents, once it is
determined where this information belongs. In the end, most Amiko Pay specific
protocol choices should either be removed, or raised to become a universal
standard for Lightning nodes.

Note 2: Since Amiko Pay is in a prototype stage, its design is expected to
change. The information written here shows the current situation in Amiko Pay,
which is not necessarily the desired final design. The information is subject to
change.


##Message confirmation and re-transmission

Since message delivery must be guaranteed, even when a node crashes / restarts,
the retransmission provided by TCP is insufficient. Because of this, the
communication protocol contains its own message confirmation and retransmission
mechanism.

There are two kinds of data that are transmitted as JSON-serialized messages
over connections: messages and confirmations.

The outer layer of messages is a dictionary, containing the
following elements:
* 'message': instance. the message itself. This is an instance of one of the
  message classes of the high-level protocol.
* 'index': number. The index of the message. This is a 16-bit unsigned integer
  (so it wraps around after 65535); every message has an index that is one
  higher than the index of the previous message.

The outer layer of confirmations is a dictionary, containing a single element:
* 'received': number. The index of a message that is confirmed to be received.
  This confirms reception of all previous messages as well, so a confirmation
  should not be sent unless all previous messages have been received.

The sending side will periodically re-transmit unconfirmed messages. Each time
the receiving side receives messages, it will transmit a corresponding
confirmation, even if the same messages were already received earlier.

To prevent wrap-around issues, the sending side must limit the length of the
queue of unconfirmed messages. A very long queue is an indication of an
unresponsive connection, and it should be treated as such: for instance, no
transaction attempts should be made until the connection is restored.

Conceptually, the 16-bit index and received numbers can be seen as read and
write pointers in a circular buffer. To prevent wrap-around issues, the write
pointer must never cross the read pointer.


##Session initiation

When a node (re)connects to another node, the first message it sends is an
instance of one of the following classes:

###ConnectLink
Used to initiate communication for a link (that may contain one or more
micro-payment channels).

Attributes:
* 'ID': string. The name of the link on the receiving side.
* 'dice': string. See below.
* 'callbackHost': string or null. If not null, the receiving side can use this
  value to re-establish the connection in the opposite direction.
* 'callbackPort': number or null. If not null, the receiving side can use this
  value to re-establish the connection in the opposite direction.
* 'callbackID': string or null. If not null, the receiving side can use this
  value to re-establish the connection in the opposite direction.

Note that ConnectLink messages correspond to URLs of the form
amikolink://host[:port]/ID

###Pay
Used to initiate communication for a single payment (between payee and payer
endpoints).

Attributes:
'ID': string. The payment ID.
'dice': string. See below.

Note that Pay messages correspond to URLs of the form
amikopay://host[:port]/ID

###Index value of initiation message
As an exception to the section 'Message confirmation and re-transmission', the
index value of session initiation messages is None, and its reception is not
confirmed. This is because the act of closing and re-establishing a connection
should be a NOP with respect to transmission of actual messages.

###Dice values
The dice values transmitted in session initiation are used in the case the two
sides of a link try to establish communication simultaneously. Each side sets
the dice value to a random 4-byte string. If a duplicate connection is detected,
both sides keep the connection with the highest dice value, and drop the other
connection. Here, 'highest' is evaluated by interpreting the dice string as a
big endian 32-bit unsigned integer.

Note that, since communication between payer and payee is always initiated by
the payer, the dice value serves no purpose there.

###Future design change: UDP
When the design is changed to be based on UDP instead of TCP, there will be
no such thing anymore as initiating a communication session. To keep track of
which data belongs to which link, each UDP packet will have to contain a
destination link ID. This ID can be included in the same level as the
confirmation and re-transmission, so it will be an extra element next to
'index' and 'message', and next to 'received'.

Dice values will no longer be used. Call-back information can be transmitted in
a regular message, as part of the regular message stream between peers.

