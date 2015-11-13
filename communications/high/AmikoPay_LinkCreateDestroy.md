#Amiko Pay link protocol: creating and destroying links

Note 1: this information is to be moved to other documents, once it is
determined where this information belongs. In the end, most Amiko Pay specific
protocol choices should either be removed, or raised to become a universal
standard for Lightning nodes.

Note 2: Since Amiko Pay is in a prototype stage, its design is expected to
change. The information written here shows the current situation in Amiko Pay,
which is not necessarily the desired final design. The information is subject to
change.


##Introduction
This document describes a part of the protocol used for communication between
neighboring nodes in the Amiko Pay network. The part described in this document
deals with creating and destroying links and channels.

A link can contain zero or more channels. Different channels within the same
link can be of a different type (e.g. Bitcoin micro-transaction channel types
vs. more simple IOU-exchanging types); each has its own balances, expiry dates
and so on.


##Creating a link
Let's assume that Alice and Bob want to create a link between each other.
In the example, Alice has a host, reachable at alice.com; Bob is reachable at
bob.net:12345.

Alice starts by giving the link a name, and creating the link in her node.
The name is an arbitrary-content string, with the restriction that it must only
contain characters that are valid in the path section of an URL.
The name must be unique in the node on which it is used.

Initially, the link in Alice's node is not connected to any remote node.

Alice generates a link URL, and passes it to Bob (for instance, by
e-mail or on a web interface). The link URL has the form:

amikolink://host[:port]/name

So, if Alice calls the link 'linkForBob', then this URL is:

amikolink://alice.com/linkForBob

Next, Bob gives this link a name, and creates the link in his node.
The same rules apply to the name as to the name created by Alice; the name of
the link on Alice's node and on Bob's node do not need to be the same.
When creating the link on his node, Bob also gives the URL to the software, so
his node knows how to connect to Alice's node.

As soon as Bob has created the link, the link in Bob's node connects to
the link in Alice's node. If, in the example, Bob has given the link the name
'linkForAlice', then the ConnectLink message sent by Bob has the following
values:

###ConnectLink
* 'ID': 'linkForBob'
* 'dice': See mid-level communications layer.
* 'callbackHost': 'bob.net'
* 'callbackPort': 12345
* 'callbackID': 'linkForAlice'

Using the callBack\* attributes, Alice knows how to reconnect to Bob, in case
the link becomes disconnected. If one of the two ever re-connects with different
data in the callBack\* attributes, then the other side will update its
call-back data; this allows nodes to move to new network locations.

A node can set the callback* values to None if there is no way for the other
node to re-connect. In that case, re-connection can only be done by one of the
two nodes.

Nodes can always disconnect the link, for instance when shutting down; it is
of course best to wait until all ongoing transactions have finished before
shutting down a node. When a link is disconnected, both nodes will periodically
try to re-connect the link.

###Future design change: authentication
Obviously, this design is very sensitive to link hijacking. An addition is
needed to authenticate links.


##Creating a channel
Let's assume that Alice and Bob have a link between each other. Alice wishes to
create a new channel in the link, where she provides the initial deposit.

Alice initates the creation of the channel by sending a Deposit message to Bob:

###Deposit
Attributes:
* 'ID': string. The name of the link on Bob's node
* 'channelIndex': number. The index of the to-be-created channel (starting
  counting at zero). This must be equal to the number of channels that was
  already present in the link before this new channel is added.
* 'channelClass': string. The name of the channel class. The two peers must
  agree on which channel types are supported on the link, and how these are
  named.

Next, a conversation consisting of zero or more 'ChannelMessage' messages
further initializes the channel:

###ChannelMessage
Attributes:
* 'ID': string. The name of the link on the node receiving this message.
* 'channelIndex': number. The index of the channel.
* 'message': string. An arbitrary-content sequence of bytes. The channel class
  determines the format of this attribute.

The number of messages and their content is specified by the channel class.
Typically, a channel class will exchange things like public keys, transactions,
signatures and the initial deposit amount.


##Destroying a channel
This is not yet implemented.


##Destroying a link
This is not yet implemented. Note that a link should only be destroyed if it
does not contain any channels.

