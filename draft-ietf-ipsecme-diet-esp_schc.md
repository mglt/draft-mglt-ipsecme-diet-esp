---
title: ESP Header Compression Profile
abbrev: EHCP
docname: draft-ietf-ipsecme-diet-esp-04
ipr: trust200902
area: Security
wg: IPsecme
kw: Internet-Draft
cat: std
stream: IETF

pi:
  toc: yes
  sortrefs: yes
  symrefs: yes

author:
      -
        ins: D.Migault
        name: Daniel Migault
        org: Ericsson
        email: daniel.migault@ericsson.com
      -
        ins: M. Hatami
        name: Maryam Hatami
        org: Concordia University
        email: maryam.hatami@mail.concordia.ca
      -
        ins: S. Céspedes
        name: Sandra Céspedes
        org: Concordia University
        email: sandra.cespedes@concordia.ca
      -  
        ins:  W. Atwood
        name: J. William Atwood
        org: Concordia University
        email: william.atwood@concordia.ca
      -
        ins: D. Liu
        name: Daiying Liu
        org: Ericsson
        email: harold.liu@ericsson.com
      -
        ins: T. Guggemos
        name: Tobias Guggemos
        org: LMU
        email: guggemos@nm.ifi.lmu.de 
      -
        ins: C. Bormann
        name: Carsten Bormann
        org: Universitaet Bremen TZI
        email: cabo@tzi.org
      -
        ins:  D. Schinazi
        name:  David Schinazi
        org: Google LLC
        email: dschinazi.ietf@gmail.com

informative:
  OpenSCHC:
    author:
    target: https://github.com/openschc
    title: OpenSCHC a Python open-source implementation of SCHC (Static Context Header Compression)


--- abstract

This document specifies Diet-ESP, a compression mechanisms for control information in IPsec/ESP communications. The compression is expressed through the Static Context Header Compression architecture.





--- middle

#  Requirements notation

{::boilerplate bcp14}


# Introduction

The Encapsulating Security Payload (ESP) {{!RFC4303}} protocol is part of the IPsec {{!RFC4301}} suite of protocols and can provide confidentiality, data origin authentication, integrity, anti-replay, and traffic flow confidentiality. The set of services ESP provides depends on the Security Association (SA) parameters negotiated between devices.
 
An ESP packet is composed of the ESP Header, the ESP Payload, the ESP Trailer, and the Integrity Check Value (ICV). ESP has two modes of operation: Transport and Tunnel. In Transport mode, the ESP Payload consists of the payload of the original IP packet; the ESP Header is inserted after the original IP packet header. In Tunnel mode, commonly used for VPNs, the ESP Header is placed after an outer IP header and before the inner IP packet headers of the original datagram. This ensures both the original IP headers and payload are protected. Consequently, the ESP Payload field contains either the payload from the original IP packet or the fully-encapsulated IP packet, in transport mode or tunnel mode, respectively.
 
The ESP Trailer, placed at the end of the encrypted payload, includes fields such as Padding and Pad Length to ensure proper alignment, and Next Header to indicate the protocol following the ESP header. The ICV, calculated over the ESP Header, ESP Payload, and ESP Trailer, is appended after the ESP Trailer to ensure packet integrity. For a simplified overview of ESP, readers are referred to Minimal ESP {{?RFC9333}}.
 
While ESP is effective in securing traffic, compression can reduce packet sizes, enhancing performance in networks with limited bandwidth. In such environments, reducing the size of transmitted packets is essential to improve efficiency. This document defines the Diet-ESP, a compression/decompression (C/D) mechanism for IPsec/ESP {{!RFC4301}} / {{!RFC4303}} packet headers (including IP, ESP, UDP, TCP... ) and the ESP trailer. The C/D is expressed through the Static Context Header Compression and Fragmentation (SCHC) framework {{!RFC8724}}{{?I-D.ietf-schc-architecture}}. The structure of the ESP packet to be compressed is shown in Figure {{fig-esp}}. 

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ ----
|               Security Parameters Index (SPI)                 | ^Auth.
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |Cove-
|                      Sequence Number                          | |rage
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ | ----
|                    Payload Data* (variable)                   | |   ^
~  Higher Layer Message (transport) or IP datagram (tunnel)     ~ |   |
|                                                               | |Encr.
+               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |Cove
|               |     Padding (0-255 bytes)                     | |rage*
+-+-+-+-+-+-+-+-+               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |   |
|                               |  Pad Length   | Next Header   | v   v
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ ------
|         Integrity Check Value-ICV   (variable)                |
~                                                               ~
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-esp artwork-align="center" title="Top-Level Format of an ESP Packet"}

## The Three compressors described in this specification 

The document outlines three compressors utilized in Diet-ESP, which are detailed as follows:

1. Inner IP Compression (IIPC): This process pertains to the compression and decompression of the IP packet protected by ESP. For outbound packets, the ESP incorporates the compressed Inner IP (IIP) into the ESP Data Payload (refer to {{fig-esp}}). In the case of inbound packets, decompression occurs after the compressed IIP is retrieved from the Data Payload within the Clear Text ESP packet.

2. Clear Text ESP Compression (CTEC): This process pertains to the compression and decompression of the ESP segment that is destined for encryption. This encompasses the Payload Data and the ESP Trailer, which includes the Padding, Pad Length, and Next Header fields, as illustrated in {{fig-esp}}. At this stage, only the latter fields are eligible for compression. For outbound packets, the ESP subsequently encrypts the compressed Clear Text ESP. For inbound packets, decompression takes place following the decryption process of the ESP.

3. Encrypted ESP Compression (EEC): This process pertains to the compression and decompression of the Encrypted ESP packet (EE), which consists of the ESP Header, the encrypted payload, and the Integrity Check Value (ICV). Since neither the encrypted payload nor the ICV can be compressed, only the ESP Header, specifically the SPI and SN fields, is subject to compression.


## The scope of SCHC in this specification  

SCHC {{!RFC8724}} offers a mechanism for header compression as well as an optional fragmentation feature. This document utilizes SCHC as a practical means to illustrate the capability to compress and decompress a structured payload. It is important to note that any elements of SCHC that pertain to aspects other than compression or decompression, such as fragmentation, fall outside the purview of this document. The reference to SCHC herein is solely for descriptive purposes related to compression and decompression, and it is not anticipated that the general SCHC framework will be integrated into the ESP implementation. The structured payloads addressed in this specification pertain to internal structures managed by ESP for the processing of an IP packet. Consequently, the compression and decompression processes outlined in this document represent supplementary steps for the ESP stack in handling the ESP packet.

## SCHC Rules and SCHC static context in this specification

SCHC facilitates the compression and decompression of headers by utilizing a common context that may encompass multiple Rules. Each Rule is designed to correspond with specific values or ranges of values within the header fields. When a Rule is successfully matched, the corresponding header fields are substituted with the Rule ID and the Compression Residue, which contains the remaining bits after compression.

In the context of IPsec, the process of encryption or decryption between IPsec peers necessitates a common context known as a Security Association (SA). More broadly, the SA encompasses all essential parameters required by the Encapsulating Security Payload (ESP) to handle both inbound and outbound packets. It is important to note that SAs are unidirectional. Furthermore, IPsec can link both outbound and inbound IP packets to the SA through Traffic Selectors (TS) or Security Parameters Index (SPI). This capability allows IPsec to uniquely associate outbound and inbound packets with a specific context (SA), which contains all pertinent information for IPsec processing.

This document adopts a comparable methodology for compression and decompression, ensuring that the SA includes all necessary parameters to create the unique Rule applicable for compressing or decompressing each structured payload. This guarantees that each SA is linked to a single Rule, thereby allowing the Rule ID to be omitted. The Rule associated with each structured payload is generated based on specific parameters referred to in this document as Attributes for Rule Generation (AfRG) (see {{sec-afrg}} for a more detailled description). These AfRGs are negotiated through IKEv2 {{!RFC7296}}, and in such cases, they are likely already included in the SA. Any additional missing AfRGs are negotiated via {{!I-D.ietf-ipsecme-ikev2-diet-esp-extension}}.

# Terminology
    
ESP Header Compression: 
: A method to reduce the size of ESP headers and trailer using predefined compression rules and contexts to improve efficiency.

ESP Trailer: 
: A set of fields added at the end of the ESP payload, including Padding, Pad Length, and Next Header, used to ensure alignment and indicate the next protocol.

Inner IP C/D (IIPC): 
: Expressed via the SCHC framework, IIPC compresses/decompresses the inner IP packet headers.

Clear Text ESP C/D (CTEC): 
: Expressed via the SCHC framework, CTEC compresses/decompresses all fields that will later be encrypted by ESP, which include the ESP Data and ESP Trailer.

Encrypted ESP C/D (EEC): 
: Expressed via the SCHC framework, EEC compresses/decompresses ESP fields that will not be encrypted by ESP.

Security Parameters Index (SPI): 
: As defined in {{!RFC4301, Section 4.1}}.

Sequence Number (SN): 
: As defined in {{!RFC4303, Section 2.2}}.

Static Context Header Compression (SCHC): 
: A framework for header compression designed for LPWANs, as defined in {{!RFC8724}}.

Static Context Header Compression Rules (SCHC Rules): 
: As defined in  {{?I-D.ietf-schc-architecture}}

RuleID: 
: A unique identifier for each Rule part of the Set of Rules.

SCHC Parameters: 
: A set of predefined values used for SCHC compression and decompression, ensuring byte alignment and proper packet formatting based on the SCHC profile.
   
SCHC MAX_PACKET_SIZE: 
: The maximum size of a SCHC-compressed packet that can be transmitted without fragmentation. 
   
Traffic Selector (TS): 
: A set of parameters (e.g., IP address range, port range, and protocol) used to define which traffic should be protected by a specific Security Association (SA).

It is assumed that the reader is familiar with other SCHC terminology defined in {{?RFC8376}}, {{!RFC8724}}, and eventually {{?I-D.ietf-schc-architecture}}.


# Diet-ESP Integration into the IPsec Stack {#sec-schc-ipsec-integration}

Figure {{fig-arch}} depicts the incorporation of Diet-ESP within the IPsec framework.

IPsec necessitates that both endpoints agree on a shared context known as the Security Association (SA). This SA is established via IKEv2 and encompasses all Attributes for Rules Generation (AfRG) (refer to {{sec-afrg}}) essential for formulating the Rules for each compressor, specifically the Inner IP packet Compressor (IIPC), the Clear Text ESP Compressor (CTEC), and the Encrypted ESP Compressor (EEC).

When an Inner IP packet (IIP) is received, IPsec identifies the SA linked to that packet. The ESP then determines the IIPC Rule from the AfRG contained within the SA and compresses the IIP packet (IIPC: C {IIP}). Subsequently, ESP constructs the Clear Text ESP payload (CTE). The CTEC Rule is derived from the AfRG of the SA, allowing for the compression of the CTE (CTEC: C {C {IIP}, ET}, where ET represents the ESP Trailer). The ESP encrypts the payload, computes the Integrity Check Value (ICV), and forms the ESP Encrypted payload (EE). The EE Rule is derived from the AfRG of the SA, and then utilized to compress the EE. The resulting compressed ESP extension is integrated into an IP packet and transmitted as outbound traffic.

For inbound traffic, the endpoint extracts the Security Parameter Index (SPI) from the compressed EE, along with any other selectors from the packet, to conduct a lookup for the SA. As outlined in {{sec-sec}}, since the SPI is derived from a potentially compressed ESP Header, there may be instances where the endpoint must explore multiple options, potentially leading to several lookups or, in the worst-case scenario, multiple signature verifications (see {{sec-sec}} for a more detailled discussion).
Once the SA is retrieved, the ESP accesses the AfRG to ascertain the EEC Rule and proceeds to decompress the EE. The ESP verifies the signature prior to decryption. Following this, the CTEC Rule is derived from the AfRG of the SA, allowing for the subsequent decompression. Finally, ESP extracts the Data Payload from the CTE packet, retrieves the IIPC Rule from the AfRG of the SA, and decompresses the IIP.

Note that implementations MAY differ from the architectural description but it is assumed that the outputs will be the same.

~~~
Endpoint                                 Endpoint
+------------------------+               +------------------------+
| Inner IP packet        |               | Inner IP packet        | 
+------------------------+               +------------------------+
========|=================================================^========
IPsec   |                                                 |
+------------------------+                                |
| SA lookup              |                                |
+------------------------+                                |
========|=================================================|========
ESP     |                                                 |
        |       +-------------------------------------+   |
        |       | Security Association                |   |                           
        |       |   - Attributes for Rule Generations |   |
        |       +-------------------------------------+   | 
        |       |  Generation of the IIPC Rule,       |   |
        |       |   CTEC Rule and EEC Rule            |   |
        |       +-------------------------------------+   | 
        |                                                 |
        v                                                 |            
+------------------------+               +------------------------+
| IIPC: C {IIP}          |               | IIPC: D {IIP}          |
+------------------------+               +------------------------+ 
| Formation of           |               | Extraction of          |
| Clear Text ESP         |               | Data Payload           |
+------------------------+               +------------------------+
| CTEC:                  |               | CTEC:                  |
| C {C {IIP}, ET}        |               | D {C {IIP}, ET}        |       
+------------------------+               +------------------------+
| Encryption             |               | Decryption             |
+------------------------+               +------------------------+
| Formation of           |               | Parsing                |
| Encrypted ESP          |               | Encrypted ESP          |
+------------------------+               +------------------------+
| EEC:                   |               | EEC:                   |  
| C {EH, C {C {IIP}, ET} |               | D {EH, C {C {IIP}, ET} |
+------------------------+               +------------------------+
        |                                | SA lookup              |
        |                                +------------------------+
========|=================================================^========
        |                                                 |
        v                                                 | 
Outbound Traffic                                  Inbound Traffic 
~~~
{: #fig-arch artwork-align="center" title="SCHC Integration into the IPsec Stack. Packets are described for IPsec in tunnel mode. C designates the Compressed header for the fields inside. IIP refers to the Inner IP packet, EH refers to the ESP Header, and ET refers to the ESP Trailer. IIPC, CTEC and EEC respectively designates the Inner IP Compress, the Clear Text ESP Compressor and the Encrypted ESP Compressor."}


## SCHC Parameters for Diet-ESP
  
The SCHC Payload section of a compressed SCHC packet is always in the form:

~~~              
0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+---------...----------+~~~~~~~~~+---------------+
|   RuleID    | Compression Residue  | Payload | SCHC padding  | 
+-+-+-+-+-+-+-+---------...----------+~~~~~~~~~+---------------+
|-------- Compressed Header ---------|         |-- as needed --|
~~~
{: #tab-diet-esp-compressed-header-format artwork-align="center" title="Diet-ESP Compressed Header Format"}

The RuleID is a unique identifier for each SCHC rule. It is included in packets to ensure the receiver applies the correct decompression rule, maintaining consistency in packet processing. Note that the Rule ID does not need to be explicitly agreed upon and can be defined independently by each party. The RuleID in Diet-ESP is expressed as 1 byte and is always elided as Rules are uniquely determined for compressors.

Other variables required for the C/D in Diet-ESP are the following:


Maximum Packet Size:
: MAX_PACKET_SIZE is determined by the specific IPsec ESP configuration and the underlying transport, but it is typically aligned with the network’s MTU. The size constraints are optimized based on the available link capacity and negotiated parameters between endpoints.

SCHC Padding:
: Padding in SCHC is used to align data to a specific boundary (typically byte-aligned or 8-bit aligned)  to meet the requirements for encryption which only considers octets instead of bits. The SCHC Padding is only used over the CTE as described in {{sec-byte-align}}.


## Attributes for Rules Generation {#sec-afrg}

The list of attributes for the Rules Generation (AfRG) is shown in {{tab-ehc-ctx-esp}}. These attributes are used to express the various compressions that operate at the IIPC, CTEC, and EEC.
    
As outlined in {{sec-schc-ipsec-integration}}, this specification does not detail the process by which the AfRG are established between peers. Instead, such negotiations are addressed in {{!I-D.ietf-ipsecme-ikev2-diet-esp-extension}}. However, the AfRG can be classified into two distinct categories. The first category encompasses AfRG that are negotiated through a specific IKEv2 extension tailored for the negotiation of AfRG linked to a particular profile, the Diet-ESP profile in this context. The AfRG referenced in {{tab-ehc-ctx-esp}} in this category are: the DSCP Compression/Decompression Action (CDA) dscp_cda, the ECN CDA ecn_cda, the Flow Label CDA flow_label_cda, the ESP alignment alignment, the ESP SPI Least Significant Bits (LSB) esp_spi_lsb, and the ESP Sequence Number LSB esp_sn_lsb.

The second category pertains to AfRG that are negotiated through IKEv2 exchanges or extensions that are not specifically designed for compression purposes. This category includes AfRG associated with TS, as identified in {{tab-ehc-ctx-esp}}, which are the TS IP Version ts_ip_version, the TS IP Source Start ts_ip_src_start, the TS IP Source End ts_ip_src_end, the TS IP Destination Start ts_ip_dst_start, the TS IP Destination End ts_ip_dst_end, the TS Protocol ts_proto, the TS Port Source Start ts_port_src_start, the TS Port Source End ts_port_src_end, the TS Port Destination Start ts_port_dst_start, and the  TS Port Destination End ts_port_dst_end. These AfRG are derived from the Traffic Selectors established through TSi/TSr payloads during the IKEv2 CREATE_CHILD_SA exchange, as described in {{!RFC7296, Section 3.13}}. The AfRG IPsec Mode designated as ipsec_mode in {{tab-ehc-ctx-esp}} is determined by the presence or absence of the USE_TRANSPORT_MODE Notify Payload during the CREATE_CHILD_SA exchange, as detailed in {{!RFC7296, Section 1.3.1}}. The AfRG Tunnel IP designated as tunnel_ip in {{tab-ehc-ctx-esp}} is obtained from the IP address of the IKE messages exchanged during the CREATE_CHILD_SA process, as noted in {{!RFC7296, Section 1.1.3}}. The AfRGs designated as ESP Encryption Algorythm esp_encr and ESP Security Parameter Index (SPI) esp_spi in {{tab-ehc-ctx-esp}} are established through the SAi2/SAr2 payloads during the CREATE_CHILD_SA exchange, while the AfRG designated as ESP Sequence Number esp_sn in {{tab-ehc-ctx-esp}} is initialized upon the creation of the Child SA and incremented for each subsequent ESP message. 

The ability to derive the SoR for the IIPC from the agreed Traffic Selectors is indicated by the variable iipc_profile. 
 

| Variable          | Possible Values             | Reference | Compressor |
|-------------------+-----------------------------+-----------+-------+
| iipc_profile      | "iipc_diet-esp", "iipc_uncompress"    | ThisRFC   | N/A   |
| dscp_cda          | "uncompress", "lower", "sa" | ThisRFC   | IIPC  | 
| ecn_cda           | "uncompress", "lower"       | ThisRFC   | IIPC  | 
| flow_label_cda    | "uncompress", "lower", "generated", "zero"  | ThisRFC   | IIPC  |
| ts_ip_version     | "IPv4-only", "IPv6-only"       | RFC7296   | IIPC  |
| ts_ip_src_start   | IPv4 or IPv6 address         | RFC7296   | IIPC  |
| ts_ip_src_end     | IPv4 or IPv6 address         | RFC7296   | IIPC  |
| ts_ip_dst_start   | IPv4 or IPv6 address        | RFC7296   | IIPC  |
| ts_ip_dst_end     | IPv4 or IPv6 address        | RFC7296   | IIPC  |
| ts_proto          | TCP, UDP, UDP-Lite, SCTP, ANY, ...  | RFC7296   | IIPC  |
| ts_port_src_start | Port number                 | RFC7296   | IIPC  |
| ts_port_src_end   | Port number                 | RFC7296   | IIPC  |
| ts_port_dst_start | Port number                 | RFC7296   | IIPC  |
| ts_port_dst_end   | Port number                 | RFC7296   | IIPC  |
| dscp_list         | list of DSCP numbers        | RFCYYYY   | IIPC  |
|-------------------+-----------------------------+-----------+-------+
| alignment         | "8 bit", "16 bit", "32 bit", "64 bit" | ThisRFC   | CTEC  |
| ipsec_mode        | "Tunnel", "Transport"       | RFC4301   | CTEC  | 
| tunnel_ip         | IPv4 or IPv6 address        | RFC4301   | CTEC  |
| esp_encr          | ESP Encryption Algorithm    | RFC4301   | CTEC  |
|-------------------+-----------------------------+-----------+-------+
| esp_spi           | ESP SPI                     | RFC4301   | EEC   |
| esp_spi_lsb       | 0-32                        | ThisRFC   | EEC   |
| esp_sn            | ESP Sequence Number         | RFC4301   | EEC   |
| esp_sn_lsb        | 0-32                        | ThisRFC   | EEC   |
|-------------------+-----------------------------+-----------+-------+
{: #tab-ehc-ctx-esp title="Set of Variables to establish Diet-ESP SCHC sessions in Diet-ESP"}

Any variable starting with "ts_" is associated with the Traffic Selectors (TSi/TSr) of the SA. 
The notation is introduced by this specification but the definitions of the variables are in {{!RFC4301}} and {{!RFC7296}}.

The Traffic Selectors may result in a quite complex expression, and this specification restricts that complexity. 
This specification restricts the resulting TSi/TSr to a single type of IP address (IPv4 or IPv6), a single protocol (e.g., UDP, TCP, or ANY), a single port range for source and destination. This specification presumes that the Traffic Selectors can be articulated as a result of CREATE_CHILD_SA with only one Traffic Selector {{!RFC7296, Section 3.13.1}} in both TSi and TSr payloads (as described in {{!RFC7296, Section 3.13}}). The TS Type MUST be either TS_IPV4_ADDR_RANGE or TS_IPV6_ADDR_RANGE. 

Let the resulting Trafic Selectors TSi/TSr be expressed via the Traffic Selector structure defined in {{!RFC7296, Section 3.13.1}}. We designate the local TS the TS - either TSi or TSr - sent by the local peer. Conversely we designate as remote TS the TS - either TSi or TSr - sent by the remote peer.  

The details of each parameter are the following:

iipc_profile:
: designates the behavior of the IIPC layer. When set to "iipc_uncompress" IIPC is not performed. This specification describes IIPC that corresponds to the "iipc_diet-esp" profile.
 
flow_label_cda:
: indicates the Flow Label CDA, that is how the Flow Label field of the inner IPv6 packet or the Identification field of the inner IPv4 packet is compressed / decompressed - See {{sec-cda}} for more information. In a nutshell, "uncompress" indicates that Flow Label (resp. Identification) are not compressed. "lower" indicates the value is read from the outer IP header - eventually with some adaptations when inner IP packet and outer IP packets have different versions. "generated" indicates that the field is generated by the receiving party. In that case, the decompressed value may take a different value compared to its original value. "zero" indicates the field is set to zero.

dscp_cda:
: indicates the DSCP CDA, that is how the DSCP values of the inner IP packet are compressed / decompressed - See {{sec-cda}} for more information. In a nutshell, "uncompress" indicates that DSCP are not compressed. "lower" indicates the value is read from the outer IP header - eventually with some adaptations when inner IP packet and outer IP packets have different versions.  "sa" indicates, compression is performed according to the DSCP values agreed by the SA (dscp_list). 

ecn_cda:
: indicates ECN CDA, that is how the ECN values of the inner IP packet are compressed / decompressed - See {{sec-cda}} for more information. In a nutshell, "uncompress" indicates that DSCP are not compressed. "lower" indicates the value is read from the outer IP header - eventually with some adaptations when inner IP packet and outer IP packets have different versions. 

ts_ip_version:
: designates the TS IP version. Its value is set  to "IPv4-only" when only IPv4 IP addresses are considered and to "IPv6-only" when only IPv6 addresses are considered.
Practically, when IKEv2 is used, it means that the agreed TSi or TSr results only in a mutually exclusive combination of TS_IPV4_ADDR_RANGE or TS_IPV6_ADDR_RANGE payloads. If TS Type of the resulting TSi/TSr is set to TS_IPV4_ADDR_RANGE, ts_ip_version takes the value "IPv4-only". Respectively, if TS Type is set to TS_IPV6_ADDR_RANGE, ts_ip_version is set to "IPv6-only".

ts_ip_src_start:
: designates the TS IP Source Start, that is the starting value range of source IP addresses of the inner packet and has the same meaning as the Starting Address field of the local TS.

ts_ip_src_end:
: designates TS IP Source End that is the high end value range of source IP addresses of the inner packet and has the same meaning as the Ending Address field of the local TS. 

ts_ip_dst_start:
: designates the TS IP Destination Start, that is the starting value range of destination IP addresses of the inner packet and has the same meaning as the Starting Address field of the remote TS.

ts_ip_dst_end:
: designates the TS IP Destination End, that is the high end value range of destination IP addresses of the inner packet and has the same meaning as the Ending Address field of the remote TS. 

ts_proto:
: designates the TS Protocol, that is the Protocol ID of the resulting TSi/TSr. This profile considers the specific protocol values "TCP", "UDP", "UDP-Lite", "SCTP" and "ANY". The representation of "ANY" is given in {{!RFC4301, Section 4.4.4.2}}.

ts_port_src_start:
: designates the TS Port Source Start, that is the the starting value of the source port range of the inner packet and has the same meaning as the Start Port field of the local TS. 

ts_port_src_end:
: designates the TS Port Source End, that is the high end value range of the source port range of the inner packet and has the same meaning as the End Port field of the local TS. 

ts_port_dst_start:
: designates TS Port Destination Start, that is the starting value of the destination port range of the inner packet and has the same meaning as the Start Port field of the remote TS. 

ts_port_dst_end:
: designates TS Port Destination End, that is the high end value range of the destination port range of the inner packet and has the same meaning as the End Port field of the remote TS. 


IP addresses and ports are defined as a range and compressed using the Least Significant Bits (LSB).
For a range defined by start and end values, msb( start, end ) is defined as the function that returns the Most Significant Bits (MSB) that remains unchanged while the value evolves between start and end.
Similarly, lsb( start, end ) is defined as the function that returns the LSB that changes while the value evolves between start and end. 
Finally, len( x ) is defined as the function that returns the number of bits of the bit array x.


dscp_list:
: designates the list of DSCP values associated to the inner traffic - see for example {{?I-D.mglt-ipsecme-dscp-np}}. These are not Traffic Selectors, but the compression mandates that the packets take one of these listed DSCP values. 


alignment:
: designates the ESP alignment as defined by {{!RFC4303}}. 

ipsec_mode:
: designates the IPsec Mode defined in {{!RFC4301}}. In this document, the possible values are "tunnel" for the Tunnel mode and "transport" for the Transport mode. 

tunnel_ip:
: designates the Tunnel IP address of the tunnel defined in {{!RFC4301}}.
This field is only applicable when the Tunnel mode is used.
That IP address can be an IPv4 or IPv6 address. 

esp_encr:
: designates the ESP Encryption Algorithm - also designated as Transform 1 in {{!RFC7296, Section 3.3.2}}. The algorithm is needed to determine whether the ESP Payload Data needs to be aligned to some predefined block size and if the ESP Pad Length and Padding fields can be compressed.  For the purpose of compression it is RECOMMENDED to use algorithms that already compressed their IV {{!RFC8750}}.

esp_spi:
: designates the Security Parameter Index defined in {{!RFC4301}}. 

esp_spi_lsb: 
: designates the LSB to be considered for the compressed SPI. A value of 32 for esp_spi_lsb will leave the SPI unchanged.

esp_sn:
: designates the ESP Sequence Number (SN) field defined in {{!RFC4301}}.

esp_sn_lsb:
: designates the LSB to be considered for the compressed SN. It works similarly to ESP SPI LSB (see esp_spi_lsb). 


### Compression/Decompression Actions in Diet-ESP {#sec-cda}

In addition to the Compression/Decompression Actions (CDAs) defined in {{!RFC8724, Section 7.4}}, this specification uses the CDAs presented in {{tab-cda}}. These CDAs are either a refinement of the compute- * CDA or the result of a combined CDA. 



| Action                 | Compression | Decompression        |
|------------------------+-------------+----------------------+
| lower                  | elided      | Get from lower layer |
| generated (Flow Label) | elided      | Compute flow label   |
| checksum               | elided      | Compute checksum     |
| ESP padding            | elided      | Compute padding      |
| hop limit              | elided      | Get from lower layer |
| SCHC padding           | send        | Compute padding      |
|------------------------+-------------+----------------------+
{: #tab-cda title="EHCP ESP related parameter"}

lower:
: is only used in a Tunnel Mode and indicates that the fields of the inner IP packet header are generated from the corresponding fields of the Tunnel IP header fields. This CDA can be used for the DSCP, ECN, and IPv6 Flow Label (resp. IPv4 identification) fields.

generated: 
: indicates that a brand new Flow Label/Identification field is generated following {{!RFC6437}}, {{!RFC6864}}.

checksum:
: indicates that a checksum is computed accordingly. Typically, the checksum CDA has a different implementation for IPv4, UDP, TCP,...

ESP padding:
: indicates that the ESP padding bytes are generated accordingly. 

hop limit:
: indicates that the hop limit is derived from the outer IPv6 header. 

SCHC padding:
: indicates that the SCHC padding bits are generated accordingly. 

# SCHC Compression for IPsec in Tunnel mode

## Inner IP Compression (IIPC) {#sec-iipc}

When iipc_profile is set to "iipc_uncompress", the packet is uncompressed. 
When iipc_profile is set to "iipc_diet-esp", IIPC proceeds to the compression of the inner IP Packet composed of an IP Header and an IP Payload.
The compression of the inner IP Payload is described in {{sec-payload}}.  
The IP Header is compressed when ipsec_mode is set to "Tunnel" and left uncompressed otherwise. ts_ip_version determines how the IPv6 Header (resp. the IPv4 header) is compressed - see {{sec-inner-ip6}} (resp. {{sec-inner-ip4}}). 


### Inner IP Payload Compression {#sec-payload}

The compression only affects UDP, UDP-Lite, TCP or SCTP packets and the type of packet is determined by the IP header.

For UDP, UDP-Lite, TCP and SCTP packets, source ports, destination ports, and checksums are compressed. 
For source port (resp. destination port) only the least significant bits are sent. FL is set to 16 bits,  TV is set to msb( ts_port_src_start, ts_port_src_end ) ( resp. ts_port_dst_start, ts_port_dst_end ), MO is set to "MSB" and CDA to "LSB". 
The checksum is elided, FL is set to 16 bits, TV is not set, MO is set to "ignore" and CDA is set to "checksum". 
This may result in decompressing a zero-checksum UDP packet with a valid checksum, but this has no impact as a valid checksum is universally accepted.

For UDP or UDP-Lite the length field is elided. FL is set to 16, TV is not set, MO is set to "ignore". 

###  Inner IPv6 Header Compression {#sec-inner-ip6}

The version field is elided, FL is set to 3, TV is set to ts_ipversion, MO is set to "equal" and CDA is set to "not-sent". 

Traffic Class is composed of the 6 bit DSCP and 2 bit ECN.
The compression of DSCP and ECN are defined independently. 

DSCP values are compressed according to the dscp_cda value:

* If dscp_cda is set to "uncompress", the DSCP values are included in the inner IP header. FL is set to 6 bits, TV is not set, MO is set to "ignore", CDA is set to "sent-value".
* If dscp_cda is set to "lower", the DSCP field is elided and its value is copied from the Tunnel IP header. FL is set to 6 bits, TV is not set, MO is set to "ignore", CDA is set to "lower".
* If dscp_cda is set to "sa", DSCP is compressed according to the DSCP values of the SA. If dscp_list contains a single element, the DSCP is elided, FL is set to 6 bits, TV is set to dscp_list[0], MO is set to "equal" and CDA is set to "not-sent". If dscp_list contains more than one DSCP value, FL is set to 6 bits, TV is set to dscp_list, MO is set to "match-mapping" and the CDA is set to "mapping-sent". 
For ECN, FL is set to 2 bits, TV is not set, MO is set to ignore and CDA is set to "value-sent".

ECN values are compressed according to the ecn_cda value:

* If ecn_cda is set to "uncompress", the ECN field is included in the inner IP header. FL is set to 2 bits, TV is not set, MO is set to "ignore", CDA is set to "sent-value".
* If ecn_cda is set to "lower", the ECN value is elided and the ECN value is copied in the outer IP header. FL is set to 2 bits, TV is not set, MO is set to "ignore", CDA is set to "lower".

Flow label is compressed according to the flow_label_cda value:

* If flow_label_cda is set to "uncompress", the Flow label is included in the IPv6 Header. FL is set to 20 bits, TV is not set, MO is set to "ignore", and CDA is set to "sent-value".
* If flow_label_cda is set to "lower", the Flow Label is elided and read from the outer IP Header (See {{sec-cda}}). FL is set to 20 bits, TV is not set, MO is set to "ignore", and CDA is set to "lower". If the outer IP header is an IPv4 header, only the 16 LSB of the Flow Label are inserted into the IPv4 Header. At the decompression, the 4 MSB of the Flow Label are set to 0. 
* If flow_label_cda is set to "generated", the Flow Label is elided and the Flow Label is then re-generated at the decompression (See {{sec-cda}}). The resulting Flow Label differs from the initial value. FL is set to 20, TV is not set, MO is set to "ignore" and CDA is set to "generated". 
* If flow_label_cda is set to "zero", the Flow Label is elided and set to 0 at decompression. A 0 value indicates no flow label is present. Fl is set to 20 bits, TV is set to 0, MO is set to "equal" and CDA is set to "not-sent". 


Payload Length is elided and determined from the Tunnel IP Header Payload Length as well as the decompressed Payload. FL is set to 16 bits, TV is not set, MO is set to "ignore", CDA is set to "lower". 

Next Header is compressed according to ts_proto:

* If ts_proto is the single value 0, Next Header is not compressed. FL is set to 8 bits, TV is not set, MO is set to "ignore", CDA is set to "sent-value".
* If ts_proto is a single non zero value, Next Header is compressed. FL is set to 8 bits, TV is set to ts_proto, MO is set to "equal" and CDA is set to "not-sent".
 
The IPv6 Hop Limit is read from the Tunnel IP Header Hop Limit. FL is set to 8 bits, TV is not set, MO is set to "ignore" and CDA is set to "lower."

The source and destination IPv6 addresses are compressed using MSB. 
In both cases, FL is set to 128, TV is respectively set to  msb(ts_ip_src_start, ts_ip_src_ed) or msb(ts_ip_dst_start, ts_ip_dst_end), the MO is set to "MSB," and the CDA is set to "LSB."


###  Inner IPv4 Header Compression {#sec-inner-ip4}

The fields Version, DSCP, ECN, Source Address and Destination Address are compressed as described for IPv6 in {{sec-inner-ip6}}.
The field Total Length (16 bits) is compressed similarly to the IPv6 field Payload Length. The field Identification (16 bits) is compressed similarly to the IPv6 field Flow Label. If the tunnel IP Header is an IPv6 Header, the Identification is placed as the LSB of the IPv6 Header and the 4 remaining MSB are set to 0.  The field Time to Live is compressed similarly to the IPv6 Hop Limit field. The Protocol field is compressed similarly to the last IPv6 Next Header field.


The Internet Header Length (IHL) is uncompressed, FL is set to 4 bits, TV is not set, MO is set to ignore and CDA is set to "value-sent".
 
The IPv4 Header checksum is elided.
FL is set to 16, TV is omitted, MO is set to "ignore," and CDA is set to "checksum."


## ESP Payload Data Byte Alignment {#sec-byte-align}

SCHC operates on bits, and the compression performed by SCHC may result in a bit payload that is not aligned to a byte (8 bits) boundary. Protocols such as ESP expect payloads to be aligned to byte boundaries (8-bit alignment).
To ensure this, we apply a padding by appending the SCHC_padding bits and the SCHC_padding_len. SCHC_padding_len is encoded over 3 bits to encode the number of bits that will compose the SCHC_padding field. As a result SCHC_padding field has between 0 and 7 bits coded over the SCHC_padding_len. The two fields are between 3 and 10 bits, so if the complementing bits are less than or equal to 2 bits, the padding will result in adding an extra byte.

When the iipc_profile is set to "iipc_uncompress" there is no ESP Payload Data Byte alignment. When iipc_profile is set to "iipc_diet-esp" ESP Payload Data Byte Alignment is performed over the Compressed Inner IP packet. This ensures that in both transport and tunnel mode, the Payload Data later encrypted by ESP result in an integer number of bytes.


## Clear Text ESP Compression (CTEC) {#sec-ctec}
    

Once the Inner IP Packet has undergone compression as outlined in {{sec-iipc}}, followed by the ESP Byte Alignment detailed in {{sec-byte-align}}, the resulting compressed inner packet comprises a specific number of bytes. To construct the Clear Text ESP Packet, it is necessary to ascertain the ESP Payload Data, the Next Header, the Pad Length, and the Padding fields.

In transport mode, the IP header of the inner packet remains uncompressed during the IIPC phase, and the ESP Payload Data is derived from the inner packet. Conversely, in tunnel mode, the Payload Data encompasses the entirety of the inner packet.

In transport mode, the Next Header field is obtained from either the inner IP Header or the Security Association, as specified in {{sec-inner-ip4}} or {{sec-inner-ip6}}. In tunnel mode, the Next Header is elided, as it is determined by ts_ip_version. FL is set to 8 bit, TV is set to IPv4 or IPv6 depending on ts_ip_version, MO is set to "equal" and CDA is set to "not-sent". 

The ESP Pad Length and Padding fields are omitted only when ESP alignment has been selected to "8bit" and esp_encr does not necessitate a specific block size alignment, or if that block size is one byte. This is represented by setting FL to (Pad Length + 1) * 8 bits, leaving TV unset, configuring MO to "ignore," and designating CDA as padding. The ESP Padding and Pad Length may vary from their decompressed counterparts. However, since the IPsec process removes the padding, these variations do not affect packet processing. When esp_encr requires a specific block size, the ESP Pad Length and Padding fields remain uncompressed.

## Encrypted ESP Compression (EEC) {#sec-eec}


SPI is compressed to its LSB.
FL is set to 32 bits, TV is not set, MO is set to "MSB( 4 - esp_spi_lsb)" and CDA is set to "LSB".

SN is compressed to their LSB, similarly to the SPI. 
FL is set to 32 bits, TV is not set, MO is set to "MSB( 4 - esp_sn_lsb)" and CDA is set to "LSB".
  
# SCHC Compression for IPsec in Transport mode

The transport mode mostly differs from the Tunnel mode in that the IP header of the packet is not encrypted. As a result, the IP Payload is compressed as described in {{sec-payload}}. The IP header is not compressed. The byte alignment of the Compressed Payload is performed as described in {{sec-byte-align}}. The Clear Text ESP Compression is performed as described in {{sec-ctec}} except for the Next Header Field, which is compressed as described in {{sec-inner-ip6}}.

# IANA Considerations

We request the IANA to create a new registry for the IIPC Profile


~~~
| IIPC Profile value | Reference |
+--------------------+-----------+
| "iipc_uncompress" | ThisRFC   |
| "iipc_diet-esp"   | ThisRFC   |
~~~

We request IANA to create the following registries for the "diet-esp" IIPC Profile. 

~~~
| Flow Label CDA Value | Reference |
+----------------------+-----------+
| "uncompress"         | ThisRFC   |
| "generated"          | ThisRFC   |
| "lower"              | ThisRFC   |
| "zero"               | ThisRFC   |
~~~

~~~
| DSCP CDA Value       | Reference |
+----------------------+-----------+
| "uncompress"         | ThisRFC   |
| "lower"              | ThisRFC   |
| "sa"                 | ThisRFC   |
~~~

~~~
| ECN CDA Value       | Reference |
+----------------------+-----------+
| "uncompress"         | ThisRFC   |
| "lower"              | ThisRFC   |
~~~


~~~
| Alignment            | Reference |
+----------------------+-----------+
| "8 bit"              | ThisRFC   |
| "16 bit"             | ThisRFC   |
| "32 bit"             | ThisRFC   |
| "64 bit"             | ThisRFC   |
~~~

~~~
| IPsec mode Value     | Reference |
+----------------------+-----------+
| "Tunnel"             | ThisRFC   |
| "Transport"          | ThisRFC   |
~~~


# Security Considerations {#sec-sec}

The security considerations encompass those outlined in ESP {{!RFC4303}} as well as those pertaining to SCHC {{!RFC8724}}.

When employing ESP {{!RFC4303}} in Tunnel Mode, the complete inner IP packet is safeguarded against man-in-the-middle attacks through cryptographic means, rendering it virtually impossible for an attacker to alter any fields associated with either the inner IP header or the inner IP payload. This level of protection is achieved by configuring the Flow Label CDA Value to "uncompress," the DSCP CDA Value to either "uncompress" or "sa," and the ECN CDA Value to "uncompress."

However, this protection is compromised if the Flow Label CDA Value, DSCP CDA Value, or ECN CDA Values are set to "lower." In such cases, the values from the inner packet for the respective fields will be derived from the outer IP header, leaving these fields unprotected. It is important to note that even the Authentication Header {{?RFC4302}} does not provide protection for these fields. When associated with a CDA value of "lower," the level of protection is akin to that provided in Transport mode. This vulnerability could be exploited by an attacker within an untrusted domain, potentially disrupting load balancing strategies that rely on the Flow Label {{?RFC6438}}{{!RFC6437}}. More broadly, when the Flow Label CDA Value is set to "lower," the relevant Flow Label Security Considerations from {{!RFC6437}} apply. Additionally, an attacker may manipulate the DSCP values to diminish the priority of specific packets, resulting in packets from the same flow having varying priorities, traversing different paths, and introducing additional latency to applications, thereby facilitating DDoS attacks. Consequently, these fields remain unprotected and are susceptible to modification during transmission, which may also be regarded as an acceptable risk.

When the Flow Label CDA Value is designated as "generated" or "zero," an attacker is unable to exploit the Flow Label field in any manner. The inner packet received is anticipated to possess a Flow Label distinct from that of the original encapsulated packet. However, the Flow Label is assigned by the receiving gateway. When the value is set to "zero," it is known, whereas when it is designated as "generated," it must be produced in accordance with the specifications outlined in {{!RFC6437}}.

The DSCP CDA Value is assigned as "sa" when DSCP values are linked to Security Associations (SAs), but it should not be utilized when all DSCP values are encompassed within a single SA. In such instances, "uncompress" is recommended.

The encryption algorithm must adhere to the guidelines provided in {{!RFC8221}} to guarantee contemporary cryptographic protection.

The least significant bits (LSB) of the ESP Security Parameter Index (SPI) determine the number of bits allocated to the SPI. An acceptable quantity of LSB must ensure that the peer possesses a sufficient number of SPIs, which is contingent upon the implementation of the SA lookup employed. If a peer relies solely on the SPI fields for SA lookup, then the LSB must be sufficiently large to satisfy the condition MAX_SPI <= 2**LSB. The SPI may assume various LSB values; however, the operator must be cognizant that if multiple LSB values are permissible for each type of SA lookup, then multiple SA lookups and signature verifications may be required. It is advisable for a peer to ascertain the LSB associated with an incoming packet in a deterministic manner.

The ESP SN LSB must be established in a manner that allows the receiving peer to clearly ascertain the sequence number of the IPsec packet. If this requirement is not met, it will lead to an invalid signature verification, resulting in the rejection of the packet. Furthermore, the LSB should have the capacity to accommodate the maximum number of packets that may be in transit simultaneously. This approach will guarantee that the last packet received is correctly linked to the corresponding sequence number.


# Acknowledgements

We would like to thank Laurent Toutain for his guidance on SCHC. Robert Moskowitz for inspiring the name "Diet-ESP" from Diet-HIP. The authors would like to acknowledge the support from Mitacs through the Mitacs Accelerate program.

--- back 



# Appendix

This appendix provides the details of the SCHC rules defined for Diet-ESP compression, alongside an explanation and an example outcome.


## JSON Representation of SCHC Rules for Diet-ESP Header Compression

The JSON file defines a set of rules within the SCHC_Context that are used for compressing and decompressing ESP headers. Each rule has a RuleID, a Description, and a set of Fields. Each field specifies how a particular part of the packet should be handled during compression or decompression. Note that the RuleID can be set by the user in any numeric order.
The rules include all the compression_levels, including IIPC, CTEC, and EEC as defined in the Terminology section.


### Transport mode:

In Transport Mode, since the IP header remains uncompressed, only the UDP payload and ESP fields are compressed. The new IANA-defined CDAs are applied as follows: "checksum" is used for the UDP checksum, meaning it is recalculated at decompression rather than transmitted. "compute-length" eliminates the UDP Length field, as it can be inferred from the payload size. "padding" ensures ESP padding aligns with 8-bit boundaries. "not-sent" is applied to the IPv6 and ESP Next Header fields because their values are predictable. The UDP Source and Destination Ports use "LSB" compression, reducing overhead by sending only the least significant bits. The ESP SPI and Sequence Number are compressed similarly using "LSB" with an MSB mask. Since the IP header is retained, the Source and Destination IPv6 Addresses are fully transmitted using "sent-value". 
~~~json
[
  {
  "ipsec_mode": "Transport",
  "ts_ip_version": "IPv6-only",
  "compressed_fields": [
    {
      "FID": "IPv6.SourceAddress",
      "FL": 128,
      "TV": "2001:db8::1001",
      "MO": "equal",
      "CDA": "sent-value"
    },
    {
      "FID": "IPv6.DestinationAddress",
      "FL": 128,
      "TV": "2001:db8::2002",
      "MO": "equal",
      "CDA": "sent-value"
    },
    {
      "FID": "IPv6.NextHeader",
      "FL": 8,
      "TV": 17,
      "MO": "equal",
      "CDA": "not-sent"
    },
    {
      "FID": "UDP.SourcePort",
      "FL": 16,
      "TV": 1234,
      "MO": "MSB",
      "CDA": "LSB"
    },
    {
      "FID": "UDP.DestinationPort",
      "FL": 16,
      "TV": 5678,
      "MO": "MSB",
      "CDA": "LSB"
    },
    {
      "FID": "UDP.Length",
      "FL": 16,
      "TV": null,
      "MO": "ignore",
      "CDA": "compute-length"
    },
    {
      "FID": "UDP.Checksum",
      "FL": 16,
      "TV": null,
      "MO": "ignore",
      "CDA": "checksum"
    },
    {
      "FID": "ESP.SPI",
      "FL": 32,
      "TV": null,
      "MO": "MSB(4 - esp_spi_lsb)",
      "CDA": "LSB"
    },
    {
      "FID": "ESP.SequenceNumber",
      "FL": 32,
      "TV": null,
      "MO": "MSB(4 - esp_sn_lsb)",
      "CDA": "LSB"
    },
    {
      "FID": "ESP.Padding",
      "FL": 8,
      "TV": null,
      "MO": "ignore",
      "CDA": "padding"
    },
    {
      "FID": "ESP.NextHeader",
      "FL": 8,
      "TV": 17,
      "MO": "equal",
      "CDA": "not-sent"
    }

]
  }
    ]
~~~
### Tunnel mode:
In Tunnel Mode, the full inner IP packet is compressed before encryption, making it more efficient for VPN scenarios. The "iipc_diet-esp" profile is used to indicate compression of the inner packet. The IPv6 Source and Destination Addresses are compressed using "MSB", sending only the variable parts while keeping the most significant bits. UDP Source and Destination Ports are compressed with "LSB", reducing their size. "compute-length" removes the UDP Length field, and "checksum" omits the UDP checksum, which is recalculated at decompression. ESP SPI and Sequence Number are compressed using "LSB" with an MSB mask. The "not-sent" CDA is used for Next Header fields in both IPv6 and ESP, as their values are predictable. 
~~~json
{
  "compressed": [
    {
      "FID": "IPv6.SourceAddress",
      "FL": 128,
      "TV": "2001:db8::1234",
      "MO": "MSB",
      "CDA": "LSB"
    },
    {
      "FID": "IPv6.DestinationAddress",
      "FL": 128,
      "TV": "2001:db8::5678",
      "MO": "MSB",
      "CDA": "LSB"
    },
    {
      "FID": "IPv6.NextHeader",
      "FL": 8,
      "TV": 17,
      "MO": "equal",
      "CDA": "not-sent"
    },
    {
      "FID": "UDP.SourcePort",
      "FL": 16,
      "TV": 5001,
      "MO": "MSB",
      "CDA": "LSB"
    },
    {
      "FID": "UDP.DestinationPort",
      "FL": 16,
      "TV": 4500,
      "MO": "MSB",
      "CDA": "LSB"
    },
    {
      "FID": "UDP.Length",
      "FL": 16,
      "TV": null,
      "MO": "ignore",
      "CDA": "compute-length"
    },
    {
      "FID": "UDP.Checksum",
      "FL": 16,
      "TV": null,
      "MO": "ignore",
      "CDA": "compute-checksum"
    },
    {
      "FID": "ESP.SPI",
      "FL": 32,
      "TV": null,
      "MO": "MSB(4 - esp_spi_lsb)",
      "CDA": "LSB"
    },
    {
      "FID": "ESP.SequenceNumber",
      "FL": 32,
      "TV": null,
      "MO": "MSB(4 - esp_sn_lsb)",
      "CDA": "LSB"
    },
    {
      "FID": "ESP.Padding",
      "FL": 8,
      "TV": null,
      "MO": "ignore",
      "CDA": "padding"
    }
  ]
}

~~~

## Example Outcome

### Input Packet

The following packet undergoes processing based on the SCHC Diet-ESP profile:

- **IPv6 Header**:
  - `Source Address`: `2001:db8::1000`
  - `Destination Address`: `ff02::5678`
  - Other attributes include `Payload Length: 18`, `Next Header: UDP`, and `Hop Limit: 64`.

- **UDP Header**:
  - `Source Port`: `123`
  - `Destination Port`: `4567`
  - `Length`: `18`
  - `Checksum`: `0x6bc9`

- **Payload**:
  - 10 bytes sample Data: `b'U\xe2(\x88\xbf\xf9\xd91\x08\xc5'`

### Compression Process

1. **UDP Header Compression**:
   - Initial size: `8 bytes`.
   - Compressed using the UDP-specific rules from the Diet-ESP profile.
   - Ports are encoded as LSB fields, reducing the size to `2 bytes`.

2. **IPv6 Header Compression**:
   - Initial size: `40 bytes`.
   - Source and destination addresses are compressed using value-sent rules based on matching prefixes.
   - Final compressed size: `17 bytes`.

3. **ESP Header Compression**:
   - Initial size: `12 bytes`.
   - SPI is not transmitted (`not-sent` CDA), and SEQ is compressed using the LSB technique.
   - Final compressed size: `2 bytes`.

4. **ESP Clear Text Compression**:

   - The ESP.NXT field (Next Header) is compressed using the match-mapping CDA:
Rule: The ESP.NXT value is matched to a single value (41 for the IPv6 Next Header).
CDA: mapping-sent is used to send only the mapped index.

5. **Payload Handling**:
   - The payload is not compressed. Further compression may be possible with additional SCHC rules.

### Decompression Process

The decompression reverses the steps:

1. **ESP Header Reconstruction**:
   - SPI is restored using the fixed value from the rule (TV=5).
   - SEQ is reconstructed from the LSB field.

2. **ESP Clear Text Reconstruction**:

   - The ESP.NXT field is restored using the mapping-sent rule, where the value 41 (Next Header for IPv6) is retrieved from the mapping.

3. **UDP Header Reconstruction**:
   - Ports are restored using the compressed LSB values.
   - Length and checksum fields are calculated using compute-length and compute-checksum CDA.

4. **IPv6 Header Reconstruction**:
   - Prefixes are restored using the value-sent fields in the rule.

5. **Payload Restoration**:
   - The payload is directly restored, as it was not compressed.

### Final Output Packet

After reconstruction, the packet is identical to the original input:

- **IPv6 Header**:
  - `Source Address`: `2001:db8::1000`
  - `Destination Address`: `ff02::5678`
  - `Payload Length`: `18`
  - `Next Header`: `UDP`
  - `Hop Limit`: `64`.

- **UDP Header**:
  - `Source Port`: `123`
  - `Destination Port`: `4567`
  - `Length`: `18`
  - `Checksum`: `0x6bc9`.

- **Payload**:
  - Data: `b'U\xe2(\x88\xbf\xf9\xd91\x08\xc5'`.



This example demonstrates the efficiency and accuracy of the Diet-ESP profile when applied to compress and decompress network packets. 

- **Efficiency**: The SCHC rules reduce packet overhead:
  - The UDP header is compressed from `8 bytes` to `2 bytes`.
  - The IPv6 header is reduced from `40 bytes` to `17 bytes`.
  - The ESP header size is decreased from `12 bytes` to `2 bytes`.
  - The ESP.NXT field is eliminated from transmission (`1 byte` reduction).
 
  These reductions are particularly beneficial in constrained environments such as Low-Power Wide-Area Networks (LPWANs).

- **Accuracy**: The decompression process fully reconstructs the original packet, ensuring no loss of information.

- **Applicability**: By leveraging these rules, the Diet-ESP profile addresses the challenges of transmitting data efficiently in constrained networks, optimizing bandwidth utilization while retaining compatibility with standard protocols.

### GitHub Repository: Diet-ESP SCHC Implementation

The source code for the implementation of the Diet-ESP profile, including the compression and decompression logic using the SCHC rules, is available on GitHub. Access the code at the following link:

GitHub Repository: [Diet-ESP SCHC Implementation](https://github.com/mglt/pyesp/tree/master/examples/draft-diet-esp.py)

This repository contains the rule definitions, examples, and source code for implementing and testing the Diet-ESP profile. Refer to the README file for setup instructions and usage guidelines.
