---
title: "CoAP over GATT (Bluetooth Low Energy Generic Attributes)"
docname: draft-amsuess-core-coap-over-gatt-latest
ipr: trust200902
stand_alone: true
cat: std
wg: CoRE
kw: CoAP, bluetooth, gatt
author:
- ins: C. Amsüss
  name: Christian Amsüss
  country: Austria
  email: christian@amsuess.com
normative:
  RFC7252:
  RFC7595:
informative:
  RFC7668:
  webbluetooth:
    title: Web Bluetooth
    author:
      -
        ins: R. Grant
      -
        ins: O. Ruiz-Henríquez
    date: 2020-02-24
    format:
      HTML: https://webbluetoothcg.github.io/web-bluetooth/
  goldengate:
    title: Golden Gate
    author:
      -
        ins: Fitbit, Inc.
    date: 2020
    format:
      HTML: https://fitbit.github.io/golden-gate/
  nefzger:
    title: Talk CoAP to me – IoT over Bluetooth Low Energy
    author:
      -
        ins: Matthias Nefzger
    date: 2021-03-01
    format:
      HTML: https://www.maibornwolff.de/en/blog/talk-coap-me-iot-over-bluetooth-low-energy
  RFC8323:
  RFC8613:
  RFC7959:
  RFC7668:
  bluetooth52:
    title: Bluetooth Core Specification v5.2
    date: 2019-12-31
    format:
      PDF: https://www.bluetooth.org/docman/handlers/downloaddoc.ashx?doc_id=478726



--- abstract

Interaction from computers and cell phones to constrained devices is limited by the different network technologies used,
and by the available APIs.
This document describes a transport for the Constrained Application Protocol (CoAP) that uses Bluetooth GATT (Generic Attribute Profile)
and its use cases.

--- middle

# Introduction {#introduction}

The Constrained Application Protocol (CoAP) {{RFC7252}} can be used with different network and transport technologies,
for example UDP on 6LoWPAN networks.

Not all those network technologies are available at end user devices in the vicinity of the constrained devices,
which inhibits direct communication and necessitates the use of gateway devices or cloud services.
In particular, 6LoWPAN is not available at all in typical end user devices,
and while 6LoWPAN-over-BLE (IPSP, the Internet Protocol Support Profile of Bluetooth Low Energy (BLE), {{RFC7668}}) might be compatible from a radio point of view,
many operating systems or platforms lack support for it,
especially in a user-accessible way.

As a workaround to access constrained CoAP devices from end user devices,
this document describes a way encapsulate generic CoAP exchanges in Bluetooth GATT (Generic Attribute Profile).
This is explicitly not designed as means of communication between two devices in full control of themselves --
those should rather build an IP based network and transport CoAP as originally specified.
It is intended as a means for an application to escape the limitations of its environment,
with a special focus on web applications that use the Web Bluetooth {{webbluetooth}}.
In that, it is similar to CoAP-over-WebSockets {{RFC8323}}.
GATT, which has read and write semantics, is not a perfect match for CoAP's request/response semantics;
this specification bridges the gap in order to make CoAP transportable over what is sometimes the only available protocol.

## Application example

Consider a network of home automation light bulbs and switches,
which internally uses CoAP on a 6LoWPAN network
and whose basic pairing configuration can be done without additional electronic devices.

Without CoAP-over-GATT,
an application that offers advanced configuration requires the use of a dedicated gateway device
or a router that is equipped and configured to forward between the 6LoWPAN and the local network.
In practice, this is often delivered as a wired gateway device and a custom app.

With CoAP-over-GATT,
the light bulbs can advertise themselves via BLE,
and the configuration application can run as a web site.
The user navigates to that web site, and it asks permission to contact the light bulbs using Web Bluetooth.
The web application can then exchange CoAP messages directly with the light bulb,
and have it proxy requests to other devices connected in the 6LoWPAN network.

For browsers that do not support Web Bluetooth,
the same web application can be packaged into an native application
consisting of a proxy process that forwards requests received via CoAP-over-WebSockets on the loopback interface to CoAP-over-GATT,
and a browser view that runs the original web application in a configuration to use WebSockets rather than CoAP-over-GATT.

That connection is no replacement when remote control of the system is desired
(in which case, again, a router is required that translates 6LoWPAN to the rest of the network),
but suffices for many commissioning tasks.

## Alternatives

Several approaches were considered, but considered unsuitable for the intended use cases:

* CoAP over 6LoWPAN over BLE (BLE IPSP):
  While this is the natural choice for transporting CoAP over BLE,
  it is unavailable on typical end user devices.
  There is no clear path toward how that would be integrated in platforms like Android or iOS,
  and even if it were, creating a network connection to a nearby device from within an application might not be possible (if how WLAN networks are managed is any indication).

* GoldenGate {{goldengate}}:
  This introduces significant network overhead,
  and burdens the end user device application with shipping a full network stack
  that is executed in a position where it can not integrate fully with the operating system's network stack.

  Moreover, this places a retransmission layer on top of a partially reliable transport (GATT),
  duplicating effort and possibly aggravating congestion situations.

* CoAP over UDP over SLIP over GATT UART {{nefzger}}:
  This is similar to the GoldenGate approach,
  but built on the GATT UART provided with Nordic Semiconductor's libraries<!-- https://learn.adafruit.com/introducing-adafruit-ble-bluetooth-low-energy-friend/uart-service -->.

  This shares the network stack duplication and retransmission concerns of GoldenGate.

* slipmux {{?I-D.bormann-t2trg-slipmux}} over BLE GATT UART service:
  This is similar to the previous item;
  the stack duplication concern is addressed,
  but retransmissions are still active atop of a service that already provides some reliability.

# Terminology {#terminology}

# Protocol description

## Boundary conditions: GATT properties {#gatt-basics}

\[ This section may be shortened in later iterations,
but is kept around while the protocol is being developed
to easily fix mistakes made from wrong assumptions. \]

CoAP-over-GATT has different properties than UDP transported over the Internet:

* Messages sent by one party are received by the other party in the order in which they are sent.
  There is no re-ordering.

  (There is also a total order on messages sent by any party,
  but that property is not useful because it's often not accessible through the Bluetooth stacks.)

* There is limited reliabiliy built into the protocol.

  Data transmissions initiated by the data source can be
  unreliable ("write without response", "notify")
  or reliable ("write with response", "indicate").

  The caveat with their relability is that acknowledgements are sent by the BLE stack,
  without consulting with the application.
  (This is not only done for simplicity but also for power efficiency:
  There is only a short time window in which the data source is listening for confirmations).
  Thus, these confirmations can not serve to acknowledge that the a CoAP request contained in the event was read, understood and is being processed.

  The reliability mechanisms are still useful, though:
  Both "write" and "notify"/"indicate" update the GATT characteristic's state,
  and while a slow application may miss data when sent in fast succession,
  it is reasonable to expect from the BLE stack to deliver the last data to the application
  when no more data is sent.

* Reads and writes may be subtly confused:
  When a characteristic is written to,
  and it is read before the BLE server application has had time to interact with its BLE stack,
  the written value may be echoed back at read time.

  This is likely not problematic when "notify"/"indicate" is used
  instead of polling reads,
  but it seems prudent to take precautions.

## Requests and responses

CoAP-over-GATT uses a GATT Characteristics to transport requst and response messages.
Similar CoAP-over-UDP it offers both reliable and unreliable transfer and message deduplication,
but as GATT's properties (see {{gatt-basics}}) differ from UDP's,
it uses a different serialization and a different kind of message IDs.

Tokens are used like with other CoAP transports,
and allow keeping multiple requests active at the same time.

A GATT server announces service of UUID 8df804b7-3300-496d-9dfa-f8fb40a236bc (abbreviated US in this document),
with one or more characteristics of UUID 3d4190a8-f322-4ff8-93fa-8d7bed520333 (abbreviated UC)
through BLE advertisements from a BLE peripheral (typically a constrained device),
which are discovered by a BLE central (typically an end user device).
The server and client roles of CoAP and GATT are independent of each other:
either BLE participant can send requests in a CoAP client role.

### Message sub-layer

At its CoAP-over-GATT characteristic, each party maintains a single bit Message ID (initialized at 1 when a connection is created),
and the last Message ID sent by the peer (initialized at 0 when a connection is created).

Messages are serialized as GATT values.
The GATT client sends a message by writing it to the characteristic (reliably using the "write with response" or unreliably using "write without response" operation);
the GATT server sends them reliably using an "indicate" or unreliably "notify" event.
The serialization format is the same for all, and illustrated in {{fig-message}}:

~~~
0   1   2   3   4       8       16      varying
+---+---+---+---+-------+-------+-------+---------+----+---------+
| R | M | C | A |  TKL  |  Code | Token | Options | ff | Payload |
+---+---+---+---+-------+-------+-------+---------+----+---------+
~~~
{: #fig-message title="Components of a message"}

* a single message description byte,
  compose of 4 bits R (Role), M (Message ID), C (Confirm) and A (Acknowledge ID),
  followed by 4 bits of token length (TKL).

* Code, token, options, payload marker and payload as in {{RFC7252}}.

  Unlike there, there is no 16-bit Message ID field
  (a similar role is taken by bits M and A),
  and in empty messages,
  the code is not sent.

The bits are set as follows:

* The R bit is always set to 0 by the GATT server,
  and to 1 by the GATT client.

* The Message ID bit is always set to the current Message ID of the sender.

* The Confirm bit is set if the sender asks the peer to acknowledge that the message has been noted.

* The Acknowledge ID is always set to the peer's last sent Message ID that had the Confirm bit set.

When receiving a message with the C bit set,
the recipient MUST eventually send a response message with radio reliability.

### Using the message sub-layer

\[ This section reflects ongoing experimentation with the above serialization format and rules.
Senders may use other patterns as long as they do not stall their peer by not sending any messages after the Confirm bit was set. \]

To send a message unreliably in terms of CoAP transmission,
a sender sets its latest Message ID in the M bit, sets C to 0, and populates the remaining bits per the rules above.
It then sends the message unreliably on the radio
(it may be sent reliably, especially when the peer set the C bit before).
After a CoAP-unreliable message, the sender may send more CoAP-unreliable messages.
It should avoid sending multiple messages in the same connection event.

To send a message reliably in terms of CoAP transmission,
a sender sets its latest Message ID in the M bit, sets C to 1, and populates the remaining bits per the rules above.
It then sends the message reliably on the radio
(it may send unreliably if a message is expected from the peer soon, but then needs to be prepared to send the same message again).
After sending that message,
the sender does not send any other message until a message is received with A equal to the sent message's M bit.
The sender may need to send the very same message again if no earlier transmission of the message happened reliably.
\[ Do we need to give timing guidance here? Probably not, because it only happens if there is some expectation in the first place. \]
The sender may cancel the transmission by sending an empty message with the same M and C bits,
or by sending different message with these bits (which are then all unreliable transmissions).

When receiving a message with the C bit set,
it is up to the recipient when to send the radio-reliable message.
If it is expected that a radio-reliable message will be sent soon,
it is permissible and useful to send unrelated unreliable messages that already account for the set C bit in their A bit.

### Message deduplication

CoAP-over-GATT participants MUST ignore a message arriving at a characteristic
if it is identical to the one received previously in the same connection.
(The first message is never ignored).

Note that it is not possible to send two identical consecutive messages unreliably.
When sending identical requests, the sender may vary the token.
Sending identical responses generally is rarely significant, even with the generalized {{?I-D.bormann-core-responses}},
because the mechanism to make responses "non-matching" in that document's terminology typically incurs variation.
When it does not, but the repetition is still significant, sending the messages reliably becomes necessary.

### Requests and responses

CoAP requests and responses are built on the message sub-layer
as they are in {{RFC7252}}:
requests are sent with a token chosen by the CoAP client,
and the CoAP server sends a response with the same token.

Responses and message-layer acknowledgments can happen in the same message.
Unlike in {{RFC7252}}, there is no association between a request and its message ID:
Any message may serve as an acknowledgement;
it is always only the token that matches requests to responses.

### Fragmentation

Attribute values are limited to 512 Bytes ({{bluetooth52}} Part F Section 3.2.9),
practically limiting blockwise operation ({{RFC7959}}) to size exponents to 4 (resulting in a block size of 256 byte).
Even smaller messages might enhance the transfer efficiency
when they avoid fragmentation at the L2CAP level. \[ TBD: Verify: \]

### Multiple characteristics

If a server provides multiple OC typed characteristics,
multiple messages can be sent without waiting for individual confirmation.
This is similar to using RFC7252 with NSTART > 1,
and may be used by the GATT client if the GATT server lists multiple UC characteristics.
The GATT server can send messages only through characteristics on which the GATT client enabled "indicate" or "notify";
if the GATT client does not support multiple characteristics,
it will just pick any and only enable them on that one.

Each characteristic has its independent message ID bits.
All characteristics of a service share a single token space,
and responses need not necessarily be sent on the characteristic the request was sent on.

The use of muliple characteristics is primarily practical
when large amounts of data are to be transferred.
These transfers can utilize much of BLE's bandwidth
because they make it easy to send much data within a single BLE connection event.

### Development directions

* Is there any good reason to allow read operations?

  A GATT client that is waiting for a Confirm bit to be acknowledged might attempt a Read
  (for the case that the confirmation arrived in an unreliable message),
  but might just as well perform the last write again.

  Reading would be more efficient (because it can happen without application intervention, and no data is sent),
  but the added complexity might not be worth the enhancements.

* Fragmentation.
  If the current approach of requiring devices to support large MTU sizes turns out to be impractical,
  or if GATT level fragmentation vastly outperforms CoAP fragmentation,
  it may be necessary to use composite reads and writes on GATT.

  Care has to be taken to use only operations supported by {{webbluetooth}}: that API does not expose reads with offsets.

  Offset based fragmentation may also be incompatible with the write-with-response approach suggested for reliability.

## Addresses

The URI scheme associated with CoAP over GATT is "coap+gatt".
The default value of Uri-Host is the MAC address of the CoAP server,
in hexadecimal encoding, with the dash character ("-") separating the bytes.
\[ Some bikeshedding is expected on these details. \]

User information and port are always absent with this scheme.

Assembling the URI of a request for the discovery resource of a BLE device with the MAC address 00:11:22:33:44:55 would thus be assembled, under the rules of {{Section 6.4 of RFC7252}}, to `coap+gatt://00-11-22-33-44-55/.well-known/core`.

Locally defined host or service name registries may be used to create names
that are more suitable for human interaction.
For DNS, which is widely used for this purpose,
no record types are registered that map to Bluetooth MAC addresses at the time of writing.

Note that on some platforms (e.g. Web Bluetooth {{webbluetooth}}),
the peer's or the own address may not be known application.
They may come up with an application-internal registered name component
(e. g. `coap+gatt://id-SomeInternalIdentifier/.well-known/core`),
but must be aware that those can not be expressed towards anything outside the local stack --
the same way they would avoid using IPv6 zone identifiers or URIs whose host name is `localhost`.


### Scheme-free alternative

As an alternative to the abovementioned scheme,
a zone in .arpa could be registered to use addresses like

~~~
coap://001122334455.ble.arpa/.well-known/core
~~~

where the .ble.arpa address do not resolve to any IP addresses.

\[ Accepting this will require a .arpa registering IANA consideration to replace the URI one. \]

### Use with persistent addresses

When services are meant to provide long-lived and universally usable URIs,
addresses based on MAC addresses can be impractical,
because they fluctuate on hardware changes.
(Moreover, privacy mechanisms on the device or the platform can render them unusable even before hardware changes).

In the absence of a usable host or service name registry,
implementers may opt for non-GATT addresses right away.
{{?I-D.ietf-core-transport-indication}} provides the means to advertise a different canonical address,
and to announce availability of that advertised service on the present transport, CoAP-over-GATT.
If the device is not generally reachable,
the canonical address might also be unreachable (see {{?I-D.ietf-core-transport-indication}} section "Unreachable canonical origin address").

When long-lived addresses circumvent privacy preserving measures,
considerations concering the tracking of devices \[ are TBD along the lines of "don't make it discoverable to unauthorized sources, and in case of doubt let the peer show its credentials first" \].

## Compression and reinterpretation of non-CoAP characteristics

The use of SCHC is being evaluated in combination with CoAP-over-GATT;
the device can use the characteristic UUID to announce the static context used.

Together with non-traditional response forms ({{?I-D.bormann-core-responses}}
and contexts that expand, say, a numeric value 0x1234 to a message like

```
2.05 Content
Response-For: GET /temperature
Content-Format: application/senml+cbor
Payload (in JSON-ish equivalent):
[
    {1 /* unit */: "K", 2 /* value */: 0x1234}
]
```

This enables a different use case than dealing with limited environments:
Accessing BLE devices via CoAP without application specific gateways.
Any required information about the application can be expressed in the SCHC context.

## Additional use of advertisements

In the current specification,
advertisements are used to indicate that CoAP-over-GATT is being used.

Two more uses of them are being considered:

* Some resource metadata might already be transported in advertisements.

  These would need to be compact (in the order of magnitude of 10 bytes or less),
  and could contain data otherwise only discovered by querying the .well-known/core resource,
  or (hashes of) AS and audience values for ACE
  to facilitate connection creation with a device known by its managed identity.

* Advertisements could contain broadcast CoAP messages.

  Given that these non-traditional responses can not have embedded requests (as defined in {{?I-D.bormann-core-responses}}) due to size contraints,
  a mechanism such as {{?I-D.ietf-core-observe-multicast-notifications}} could be used to distribute some consensus request.

# IANA considerations

## Uniform Resource Identifier (URI) Schemes

IANA is asked to enter a new scheme into the "Uniform Resource Identifier (URI) Schemes" registry set up in {{RFC7595}}:

* URI Scheme: "coap+gatt"
* Description: CoAP over Bluetooth GATT (sharing the footnote of coap+tcp)
* Well-Known URI Support: yes, analogous to {{RFC7252}}

# Security considerations

All data received over GATT is considered untrusted;
secure communication can be achieved using OSCORE {{RFC8613}}.

Physical proximity can not be inferred from this means of communication.

--- back

# Change log

Since -02:

* Message format extended by a leading byte, the option to have a token.
  This enables role reversal and concurrent requests.
* The UC identifier was changed to reflect the incompatible change in protocol.
* A section on used BLE properties was added.
* A section providing outlook on other data for advertisements was added.

Since -01:

* Point out (possibly conflicting) development directions.
* Describe URI scheme more completely, including persistent addresses.
* Aim for standards track.
* Describe rejeced alternative approaches.

Since -00:

* Add note on SCHC possibilities.
