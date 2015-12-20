#Amiko Pay link protocol: IOU Channel communication

Note 1: this information is to be moved to other documents, once it is
determined where this information belongs. In the end, most Amiko Pay specific
protocol choices should either be removed, or raised to become a universal
standard for Lightning nodes.

Note 2: Since Amiko Pay is in a prototype stage, its design is expected to
change. The information written here shows the current situation in Amiko Pay,
which is not necessarily the desired final design. The information is subject to
change.


##Introduction
This document describes the part of the communication protocol specific to the
"IOU" Channel type in Amiko Pay.

An IOU channel is not a "real" micro-payment channel. Instead of using Bitcoin
smart contracts to set up a trust-free payment channel between two peers, an IOU
channel does not set up anything, and requires trust between the two peers
during its operation. On closing the channel, a single regular Bitcoin
transaction is performed to settle the balance.

An IOU channel always trades IOUs issued by the peer that created the channel
by "depositing" into it. The "deposited" amount has no meaning, except that it
is an upper limit to the amount of IOU that can be issued in this channel.
The balance of the non-issuing peer is the amount of IOU issued. Initially,
this is zero; on channel closing, this is the amount that will be transferred
from the issuing to the non-issuing peer. The balance of the issuing peer is
the remaining amount that can still be issued as IOUs. Initially, this is the
"deposited" amount; on channel closing, this amount is ignored.

Note that, while this channel type allows transactions in both directions,
trust is unidirectional: one party needs to trust IOUs from the other party,
but not vice versa. This is a feature: it means that one of the two parties
does not have to provide evidence of trustworthiness to the otner party (e.g.
one of the two parties may remain anonymous to the other). If a symmetrical
trust / IOU issuance relationship is desired, this can be accomplished by
establishing IOU channels in both directions.


##Depositing
The 'channelClass' attribute in the Deposit message must be equal to
'IOUChannel', to indicate that this channel type is being used.

After the Deposit message, the depositing side sends a single ChannelMessage.
The 'message' attribute of this ChannelMessage is a number, indicating the
amount (in Satoshi) "deposited" into the channel.


##Withdrawing
This is not yet implemented. Presumably, this involves a two-step operation,
where the non-issuing peer first sends a Bitcoin address to the issuing peer,
who then sends back a Bitcoin transaction that sends the required amount to
that address.


##Fund locking
No ChannelMessages are exchanged on fund locking.


##Commit settlement
No ChannelMessages are exchanged on commit settlement.


##Rollback settlement
No ChannelMessages are exchanged on rollback settlement.

Note: Settling for rollback is not yet implemented.

