#Amiko Pay payment link protocol

Note 1: this information is to be moved to other documents, once it is
determined where this information belongs. In the end, most Amiko Pay specific
protocol choices should either be removed, or raised to become a universal
standard for Lightning nodes.

Note 2: Since Amiko Pay is in a prototype stage, its design is expected to
change. The information written here shows the current situation in Amiko Pay,
which is not necessarily the desired final design. The information is subject to
change.


##Introduction
This protocol is used for direct two-way communication between payer and payee
endpoints. The payer and payee are generally not connected to each other
directly (one-hop) with direct micro-payment channels, but they are expected to
communicate directly. This can be done, for instance, by NFC, Bluetooth or WiFi
in a physical shop, or by connecting over the internet or via a TOR hidden
service.

The following sections describe the sequence for performing a single payment.


##Creation of a payment request
The payee defines a payment request by entering the following information:
* The to-be-paid amount
* The receipt

The receipt is, for now, an arbitrary contents string. Its purpose is to
specify the goods/services that are exchanged in the transaction. In the future,
some standard may be defined on how the payee may cryptographically sign the
receipt, so the receipt may act as a proof of ownership for the payer.

The payee generates a commit token, which will be used for committing the
transaction. It also calculates the transaction ID, which is a secure hash
of the commit token (currently, RIPEMD160(SHA256(x)) is used as hash function).

The payee's node stores the payment request data, and assigns it a payment ID.
The payment ID is an arbitrary-content string, with the restriction that it
must only contain characters that are valid in the path section of an URL.
The payment ID must be unique in the node that creates it.

The payee's node returns a payment URL, of the form:

amikopay://host[:port]/ID

where ID is the payment ID, and host[:port] is the address where the payer can
connect to the payee.

The payee sends the payment request to the payer. This can be done, for
instance, through NFC, with a QR code, or with a hyperlink on a website.


##Payment initiation
The payer uses the payment URL to connect to the payee. Session initiation is
done with a Pay message.

When the payee receives the Pay message, it responds with a Receipt message:

###Receipt
Attributes:
* 'amount': number. The amount.
* 'receipt': string. The receipt.
* 'transactionID': string. The transaction ID (the hash of the commit token).
* 'meetingPoints': array of strings. Each entry is the ID of a meeting point
  that is acceptable to the payee for routing this transaction.

Once the payer receives the receipt data, the receipt and the amount are shown
to the user, and the user chooses whether or not to agree with the transaction.
Depending on the technology used, it may be impossible on the payer side to
show the receipt and the amount to the user. More low-tech wallets, used for
smaller payments, may simply assume that the user agrees, or only ask for
authentication / confirmation without showing the receipt.

The payer node then responds by sending either a Confirm message or a Cancel
message to the payee:

###Confirm
Attributes:
* 'ID': string. The payment ID.
* 'meetingPointID': string. The meeting point that will be used for routing
  this transaction. This must be one of the meeting points mentioned in the
  Receipt message.

###Cancel
Attributes:
* 'ID': string. The payment ID.

In the case of a Cancel message, the connection between payer and payee is
closed, and no further action is performed for the payment. The payment is
considered cancelled: the payee should erase the commit token, and not release
it to anyone; goods / services should not be delivered.


##Routing
In the case of a Confirm message, both the payer and the payee start routing
the payment towards the meeting point.


##Failed routing
The case that one or both sides fail to find a route is not yet implemented.
The goal is that the transaction ends up as cancelled.
Routing failure can be detected by exhausting all routing possibilities, or
by exceeding a time-out before finding a route.


##Locking
When the payee finds a route, it sends a HavePayeeRoute message to the payer:

###HavePayeeRoute
Attributes:
* 'ID': string. Set to the value '__payer__'.
* 'transactionID': string. The transaction ID (the hash of the commit token).

This allows the payer to detect when both routes are established. As soon as
that is the case, the payer starts locking funds on the first hop of the route.
Funds locking continues over the route from payer to meeting point, and then
from meeting point to payee.


##Failed locking
Locking failure can be detected by both payer and payee as a time-out. The
case of a locking failure is not yet implemented. The goal is that the
transaction ends up as cancelled.


##Payer/payee committing
Once the locking reaches the payee, the payee considers the transaction
committed, for as far as the payer is concerned: the payee can safely inform
the payer that the payment has succeeded, and deliver the goods / services
to the payer.

The payee then releases the commit token to the payer with a Commit message:

###Commit
* 'token': string. The commit token.

On receiving the commit token, the payer considers the transaction
committed, for as far as the payee is concerned: the payer can now claim to
the payee that the payment has succeeded, and claim the goods / services.
If necessary, the commit token can act as proof.

Note that the next steps are necessary for successfully committing the
transaction, but the software can handle this in the background. Payer and
payee do not have to wait for this; this is useful, e.g. in POS situations
where speed is important.


##Network committing
The last step is that both the payer and the payee start committing on their
links. Note that, since they work in opposite directions, the incentives are
different: the incentive on the payee side is the strongest (claiming the
locked funds); the only incentive on the payer side is to release the locked
funds to make them available for future transactions in the opposite direction.

Note that it is possible that the network committing in payee->payer direction
reaches the payee earlier than the Commit message from the payee. In that case,
the payer will also consider the payment to be committed.

As soon as both payer and payee have committed their links, they are no longer
involved in the transaction, and they can close the communication session
between them. Further committing can continue through the route,
as long as there are no issues (e.g. misbehaving / non-responsive nodes). If
there are any problems, the slower mechanisms for handling these (HTLC
time-outs) are only required for a sub-set of the route that contains the
problematic nodes.

