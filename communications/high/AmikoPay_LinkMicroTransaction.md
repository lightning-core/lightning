#Amiko Pay link protocol: performing a micro-transaction

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
deals with performing a micro-transaction over an already existing channel.


##Route establishment and fund reservation
Let's assume that Alice and Bob have a link between each other. Alice wishes to
establish a route for a micro-payment, and tries a route using non-source
routing through Bob. Alice starts by sending a MakeRoute message to Bob:

###MakeRoute:
Attributes:
* 'amount': number. The amount (in Satoshi).
* 'transactionID': string. The transaction ID (the hash of the commit token).
* 'startTime': number. The start of the time range in which the transaction can
  be committed by showing the transaction token.
* 'endTime': number. The end of the time range in which the transaction can
  be committed by showing the transaction token.
* 'meetingPointID': string.
* 'channelIndex': number. The index of the channel.
* 'isPayerSide': boolean. True if this is the route from the payer to the
  meeting point, False if this is the route from the payee to the meeting point.

Alice will (informally, without any microtransaction channel update or other
cryptographic commitment) reserve the funds required for the transaction, to
make sure the channel won't go out of funds before the routing is finished.
On receiving the MakeRoute message, Bob will do the same. Time-out of this
reservation is TBD.

Next, Bob will try to find a route towards the meeting point, using his other
links. Of course, if Bob is the meeting point, this process succeeds in a
trivial way.

If Bob fails to find a route, he sends back a HaveNoRoute message to Alice:

###HaveNoRoute:
Attributes:
* 'transactionID': string. The transaction ID (the hash of the commit token).
* 'isPayerSide': boolean. True if this is the route from the payer to the
  meeting point, False if this is the route from the payee to the meeting point.

In this case, both nodes will unreserve the funds on the channel. The transacion
finishes in a cancelled state on this link; Alice may try to establish a route
by using her other links.

The other case is that Bob finds a route. In that case, Bob sends a HaveRoute
message:

###HaveRoute:
Attributes:
* 'transactionID': string. The transaction ID (the hash of the commit token).
* 'isPayerSide': boolean. True if this is the route from the payer to the
  meeting point, False if this is the route from the payee to the meeting point.

Alice can send a CancelRoute message to Bob:

###CancelRoute:
Attributes:
* 'transactionID': string. The transaction ID (the hash of the commit token).
* 'isPayerSide': boolean. True if this is the route from the payer to the
  meeting point, False if this is the route from the payee to the meeting point.

Since her decision to send this
message is asynchronous from Bob's decision to send a HaveNoRoute / HaveRoute
message, Alice can send a CancelRoute both before and after a HaveNoRoute /
HaveRoute from Bob.

When Alice sends a CancelRoute, both nodes will unreserve the funds on the
channel. The transacion finishes in a cancelled state on this link; Alice may
try to establish a route by using her other links. Note that CancelRoute takes
precedence over HaveRoute. Since it has the same effect as HaveNoRoute,
precedence order with HaveNoRoute is irrelevant.

The rest of this protocol description assumes that a HaveRoute has been sent,
and no CancelRoute has been sent.


##Fund locking

Let's assume that the newly established route represents a payment from Alice
to Bob. This means that, in case isPayerSide equals True, they have the same
roles as in the previous section; in case isPayerSide equals False, their roles
are reversed.

The next step is that the funds are locked in the payment channel. To do this,
Alice sends a Lock message to Bob:

###Lock:
Attributes:
* 'transactionID': string. The transaction ID (the hash of the commit token).
* 'isPayerSide': boolean. True if this is the route from the payer to the
  meeting point, False if this is the route from the payee to the meeting point.
* TBD: a payload attribute, with comments specific to the channel type.

Next, a conversation consisting of zero or more 'ChannelMessage' messages
finishes the locking. The number of messages and their content is specified by
the channel class.

TBD: collision between Lock and route establishment messages.


##Commit settlement requesting

If Bob (the receiving side of the payment) receives the transaction token, he
can request a commit settlement by sending a RequestCommit message to Alice:

###RequestCommit:
Attributes:
* 'token': string. The commit token.
* 'isPayerSide': boolean. True if this is the route from the payer to the
  meeting point, False if this is the route from the payee to the meeting point.

If this is done before the time-out of the transaction, this proves to Alice
that Bob can enforce a commit on the transaction channel, if necessary. Under
these conditions, Alice is incentivized to voluntarily provide a commit
settlement on the transaction channel, to free up the locked funds as soon as
possible for future transactions. Also, having the token allows Alice to send
a RequestCommit message on her payer-side link.


###Settlement:

If Alice (the sending side of the payment) receives the transaction token,
either through a RequestCommit message or in any other way, she can settle for
committing the transaction on the link, by sending a SettleCommit message:

###SettleCommit:
Attributes:
* 'token': string. The commit token.
* 'isPayerSide': boolean. True if this is the route from the payer to the
  meeting point, False if this is the route from the payee to the meeting point.
* TBD: a payload attribute, with comments specific to the channel type.

Next, a conversation consisting of zero or more 'ChannelMessage' messages
finishes the settling. The number of messages and their content is specified
by the channel class.

Settling for commit finishes the transacion in a committed state on
this link; the funds are freed up, and become available to the receiving side
(Bob). Bob can now voluntarily choose to settle for commit on his payee-side
link: this frees up funds as soon as possible for future transactions, and it
is a way of 'acting nice' to his payee-side neighbor.

The opposite settlement is also possible: Bob (the receiving side of the
payment) can voluntarily settle for roll-back with a CommitRollback message:

###SettleRollback:
Attributes:
* 'transactionID': string. The transaction ID (the hash of the commit token).
* 'isPayerSide': boolean. True if this is the route from the payer to the
  meeting point, False if this is the route from the payee to the meeting point.
* TBD: a payload attribute, with comments specific to the channel type.

Next, a conversation consisting of zero or more 'ChannelMessage' messages
finishes the settling. The number of messages and their content is specified
by the channel class.

Bob can safely do this if the payment time-out has been exceeded, if he has
received a roll-back settlement on his payee-side link, or if he has some other
way of being sure he will never be forced to pay on his payee-side link. This
frees up funds as soon as possible for future transactions, and it is a way of
'acting nice' to Alice.

Settling for roll-back finishes the transacion in a cancelled state on
this link; the funds are freed up, and become available to the sending side
(Alice). Alice can now voluntarily choose to settle for roll-back on her
payee-side link: this frees up funds as soon as possible for future
transactions, and it is a way of 'acting nice' to her payer-side neighbor.

Note: Settling for rollback is not yet implemented.


##No Settlement:
If neither party settles, the transaction continues to exist in a non-settled
state in the microtransaction channel until the channel is closed. Presumably,
channel closing will be non-cooperative, since cooperative channel closing will
probably require all microtransactions to be finished.


##Final note:
Routing from payee to meeting point, and from payer to meeting point, happens
independently from each other. It is possible that both establish a route
through the same link, or even the same channel. There are both cases where
this happens in the same direction as in opposite direction. Even though, in
efficient routing schemes, this is expected to be a rare occurrence, all
messages have been designed to be robust against these cases. Typically, the
pair of attributes (TransactionID, isPayerSide) acts as an identifier to a
specific micro-transaction on a link. A node should never attempt to establish
a route with identical (TransactionID, isPayerSide) multiple times over the
same link, and there should never be a need to do so.

