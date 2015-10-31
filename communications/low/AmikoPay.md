#Amiko Pay low-level communication protocol

Note 1: this information is to be moved to other documents, once it is
determined where this information belongs. In the end, most Amiko Pay specific
protocol choices should either be removed, or raised to become a universal
standard for Lightning nodes.

Note 2: Since Amiko Pay is in a prototype stage, its design is expected to
change. The information written here shows the current situation in Amiko Pay,
which is not necessarily the desired final design. The information is subject to
change.


##Bottom-level

TCP is used for all connections; for internet layer protocols, anything
should do, e.g. no assumptions are made on IP protocol version. Messages are
text-based, and separated by newlines.

Advantages of text-based messages:
* Easy debugging
* Many text-based serialization formats are easily extensible

Disadvantages of text-based messages:
* Bloated

###Future design change: use UDP instead of TCP
Since our need to receive clarity over whether a message has arrive exceed what
TCP provides, the higher-level design performs its own message retransmission
mechanism. Under these circumstances, TCP offers little advantage; a switch to
UDP is planned. Message separation is trivial in UDP; all messages are expected
to fit in the max UDP message size of 65k bytes.

TBD is whether UDP works well over TOR: this is highly desirable.

Also TBD is how to let bi-directional UDP work well with NAT routers,
considering the P2P nature of the network. In some cases, people might connect
through NAT routers that are not under their own control, e.g. when using
the WiFi of a physical shop. There are work-arounds, such as using a VPN.

###Future design change: encryption
Encryption will be added in-between the bottom-level protocol and serialization.
This obviously removes the advantage of readability of text-based messages, but
text-based plaintext may still aid debugging, since it is easy to add
functionality to a node for dumping the plaintext for debugging purposes.

The only non-encrypted part might be the initial authentication and key
exchange; it may be useful to remove as much redundancy as possible from this
non-encrypted part, to make it difficult to distinguish Lightning communication
from other encrypted protocols.


##Serialization
Messages are serialized with JSON. A couple of conventions are used on top of
JSON:

###Strings
Because of the need to transfer binary data, a translation is performed
between the to-be-serialized data string and the string that will end up in
JSON. This translation is designed to leave human-readable strings (mostly)
unchanged, and write binary data in hexadecimal format.

The translation has the following cases:
* Binary strings: encoded as hexadecimal text, prefixed with '!x'.
* Text strings starting with a '!': encoded as the string itself, prefixed with '!'.
* Text strings not starting with a '!': encoded as the string itself

All strings containing byte-values between 0 and 31 and/or between
128 and 255 are considered binary. Note that, for text strings, the regular
JSON escaping still applies, e.g. for quote characters.

###Objects
JSON objects are used in two ways: for serializing dictionaries (key->value
mappings), and for instances of classes.

Instances of classes have a "_class" element in the serialized object; the
corresponding value is the name of the class. All other elements are attributes
of the instance. All JSON objects that do not have a "_class" element are
serialized dictionaries.

Note that, as a consequence, dictionaries must never contain a "_class" key,
and instances must never have an attribute named "_class".

