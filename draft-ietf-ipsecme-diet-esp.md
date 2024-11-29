---
title: ESP Header Compression Profile
abbrev: EHCP
docname: draft-ietf-ipsecme-diet-esp-03
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
   title: OpenSCHC a Python open-source implementation of SCHC (Static Context Header Compression) RFC8724


--- abstract

The document specifies Diet-ESP, an ESP Header Compression Profile (EHCP) that compresses IPsec/ESP communications using Static Context Header Compression (SCHC).




--- middle

#  Requirements notation

{::boilerplate bcp14}


# Introduction

The Encapsulating Security Payload (ESP) {{!RFC4303}} protocol is part of the IPsec {{!RFC4301}} suite of protocols and can provide confidentiality, data origin authentication, integrity, anti-replay, and traffic flow confidentiality. The set of services ESP provides depends on the Security Association (SA) parameters negotiated between devices.
 
An ESP packet is composed of the ESP Header, the ESP Payload, the ESP Trailer, and the Integrity Check Value (ICV). ESP has two modes of operation: Transport and Tunnel. In Transport mode, the ESP Payload consists of the payload of the original IP packet; the ESP Header is inserted after the original IP packet header. In Tunnel mode, commonly used for VPNs, the ESP Header is placed after an outer IP header and before the inner IP packet headers of the original datagram. This ensures both the original IP headers and payload are protected. Consequently, the ESP Payload field contains either the payload from the original IP packet or the fully-encapsulated IP packet, in transport mode or tunnel mode, respectively.
 
The ESP Trailer, placed at the end of the encrypted payload, includes fields such as Padding and Pad Length to ensure proper alignment, and Next Header to indicate the protocol following the ESP header. The ICV, calculated over the ESP Header, ESP Payload, and ESP Trailer, is appended after the ESP Trailer to ensure packet integrity. For a simplified overview of ESP, readers are referred to Minimal ESP {{?RFC9333}}.
 
While ESP is effective in securing traffic, further optimization can reduce packet sizes, enhancing performance in networks with limited bandwidth. In such environments, reducing the size of transmitted packets is essential to improve efficiency. This document defines the ESP Header Compression Profile (EHCP), namely Diet-ESP, for compression/decompression (C/D) of IPsec/ESP {{!RFC4301}} / {{!RFC4303}} packets using the Static Context Header Compression and Fragmentation (SCHC) framework {{!RFC8724}}. The structure of Diet-ESP is shown in {{fig-esp}}. Compression with SCHC is based on the use of a Set of Rules (SoR) used in a SCHC instance for C/D operations {{!I-D.ietf-schc-architecture}}. One or more SoR constitute the SCHC Context. In the case of IPsec, the Context can be agreed via IKEv2 {{!RFC7296}} and its specific extension {{!I-D.ietf-ipsecme-ikev2-diet-esp-extension}}.
 
As a result of the application of the same SoR, header values shared by the end-points do not need to be sent on the wire. At the receiver, header information is re-generated from the received compressed packet and the application of the proper SoR retrieved from the Context.

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ ----
|               Security Parameters Index (SPI)                 | ^Int.
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |Cov-
|                      Sequence Number                          | |ered
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ | ----
|                    Payload Data* (variable)                   | |   ^
~                                                               ~ |   |
|                                                               | |Conf.
+               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |Cov-
|               |     Padding (0-255 bytes)                     | |ered*
+-+-+-+-+-+-+-+-+               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |   |
|                               |  Pad Length   | Next Header   | v   v
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ ------
|         Integrity Check Value-ICV   (variable)                |
~                                                               ~
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #fig-esp artwork-align="center" title="Top-Level Format of an ESP Packet"}

    
This document defines the ESP Header Compression profile (EHCP) Architecture with the application of SCHC at various layers of the IPsec stack---also called SCHC strata---as defined below:

1. Inner IP Compression (IIPC): The SoR used in this SCHC stratum apply directly to the headers of the inner IP packet. For example, in the case of a UDP packet with ports determined by the SA, fields such as UDP ports and checksums are typically compressed. If no compression of the inner packet is possible, the resulting SCHC packet contains the uncompressed IP packet, as per {{!RFC8724, Section 7.2}}.

2. Clear Text ESP Compression (CTEC): This SCHC stratum contains SoR that compress the fields of the ESP Payload, right before being encrypted, as the encapsulated traffic in tunnel mode. 

3. Encrypted ESP Compression (EEC): This SCHC stratum contains SoR that compress the ESP fields that remain visible after encryption, that is, the ESP Header.

Note that the descriptions of the three SCHC strata provided in this document meet the general purpose of ESP. It is possible that in some deployments, the SCHC instances from different SCHC layers can be merged. Typically, a specific implementation may merge the compression of IIPC and CTEC layers.

Each SoR indicates the ESP header fields to be matched by the rules. The SCHC instances define how the SCHC Context is initialized from the SA and generate the corresponding SCHC rules (i.e., RuleID, SCHC MAX_PACKET_SIZE, new SCHC Compression/Decompression Actions (CDA), and fragmentation). The appendix provides illustrative examples of applications of EHCP implemented with the OpenSCHC {{OpenSCHC}}. 

# Terminology
    
ESP Header Compression Profile (EHCP): 
: A method to reduce the size of ESP headers using predefined compression rules and contexts to improve efficiency.

ESP Trailer: 
: A set of fields added at the end of the ESP payload, including Padding, Pad Length, and Next Header, used to ensure alignment and indicate the next protocol.

SCHC Stratum: 
: Refers to the specific layer in the ESP packet structure where the Set of Rules of a SCHC instance are applied for compression and decompression. 

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
: As defined in {{!RFC8724}}, Section 5.

RuleID: 
: A unique identifier for each SCHC rule, as defined in {{!RFC8724}}, Section 5.1.

SCHC Context: 
: The set of rules shared between communicating entities, as defined in {{!RFC8724}}, Section 5.3.

   
SCHC Parameters: 
: A set of predefined values used for SCHC compression and decompression, ensuring byte alignment and proper packet formatting based on the SCHC profile.
   
SCHC MAX_PACKET_SIZE: 
: The maximum size of a SCHC-compressed packet that can be transmitted without fragmentation. 
   
Traffic Selector (TS): 
: A set of parameters (e.g., IP address range, port range, and protocol) used to define which traffic should be protected by a specific Security Association (SA).

It is assumed that the reader is familiar with other SCHC terminology defined in {{!RFC8376}}, {{!RFC8724}}, and {{?I-D.ietf-schc-architecture}}.


# SCHC Integration into the IPsec Stack

The main principle of the ESP Header Compression Profile (EHCP) is to avoid sending information that has already been shared by the peers. Different profiles and technologies, such as those defined by {{!RFC4301}} and {{!RFC4303}}, ensure that ESP can be tailored to various network requirements and security policies. However, ESP is not optimized for bandwidth efficiency because it has been designed as a general-purpose protocol. EHCP aims to address this by leveraging a profile, expressed via the SCHC architecture, to optimize the ESP header sizes for better efficiency in constrained environments.

{{fig-arch}} illustrates the integration of SCHC into the IPsec stack, detailing the different layers and components involved in the compression and decompression processes. The diagram is divided into two entities, each representing an endpoint of a communication link. 

Rules for compression are derived from parameters associated with the Security Association (SA) and agreed upon via IKEv2 {{!RFC7296}}, as well as specific compression parameters designated as Attributes for Rules Generation (AfRG) (see {{sec-afrg}}). These AfRGs are also agreed upon via IKEv2 in {{!I-D.ietf-ipsecme-ikev2-diet-esp-extension}}.

Upon establishing the SA, Diet-ESP uses the AfRGs listed in {{tab-ehc-ctx-esp}} for derivation of the SoR applicable to each SCHC stratum. The collection of rules are then used for the SCHC Context initialization. The reference column in {{tab-ehc-ctx-esp}} indicates the source where the parameter value is defined. The C/D column specifies in which of the SCHC strata the parameter is being used. 

EHCP defines three SCHC strata for compression: Inner IP Compression (IIPC), Clear Text ESP Compression (CTEC), and Encrypted ESP Compression (EEC). The compression operations for each stratum are described in {{sec-iipc}}, {{sec-ctec}}, and {{sec-eec}}.

EHCP essentially limits the scope of the compression to the inner IP headers and specific fields such as ports and checksums of transports like UDP, UDP-Lite, TCP, SCTP.  Further and more specific compression profiles may be defined in the future to cover compression of headers of different upper layer protocols.

At the receiver endpoint, the decompression of the inbound packet follows a reverse process. First, the Encrypted ESP C/D (EEC) decompresses the encrypted ESP header fields. After the ESP packet is decrypted, the Clear Text ESP C/D (CTEC) decompresses the Clear Text fields of the ESP packet.

Note that implementations MAY differ from the architectural description but it is assumed that the outputs will be the same.


~~~
                 +--------------------------------+ 
                 | ESP Header Compression Context |
                 |   - Security Association       |
                 |   - Additional Parameters      |
                 +--------------------------------+    
                                 |        
Endpoint                         |                 Endpoint
                                 |
+-----------------+              |                +-----------------+
| Inner IP packet |              |                | Inner IP packet |
+-----------------+              |                +-----------------+
| SCHC(IIP + UDP  |              |                | SCHC(IIP + UDP  |
| or ...)         |+--------IIPC layer-----------+|  or ...)        |
+-----------------+          C {IIP}              +-----------------+
| ESP             |              |                | ESP             |
| (Encapsulation) |              |                | (unwrapping)    |
+-----------------+              |                +-----------------+
| SCHC            |              v                |  SCHC           |
| (ESP Payload)   |+--------- CTEC layer --------+| (ESP Payload)   |
+-----------------+      EH, C {C {IIP}, ET}      +-----------------+
| ESP             |              |                | ESP             |
| (Encryption)    |              |                | (decryption)    |
+-----------------+              v                +-----------------+ 
| SCHC(ESP Header)|+--------- EEC layer ---------+| SCHC(ESP Header)|    
+-----------------+  IP, C {EH, C {C {IIP},  ET}} +-----------------+
| IPv6 + ESP      |                               | IPv6 + ESP      |    
+-----------------+                               +-----------------+
|  L2             |                               |  L2             |
+-----------------+                               +-----------------+
~~~
{: #fig-arch artwork-align="center" title="SCHC Integration into the IPsec Stack. Packets are described for IPsec in tunnel mode. C designates the Compressed header for the fields inside. IIP refers to the Inner IP packet, EH refers to the ESP Header, and ET refers to the ESP Trailer. The labels “SCHC (IIPC: Compress Inner IP),” “SCHC (CTEC: Compress Trailer),” and “SCHC (EEC: Compress ESP Header)” are added to indicate that different SCHC instances are applied at the IIPC, CTEC, and EEC layers, respectively."}


## SCHC Parameters for Diet-ESP
  
A SCHC compressed packet is always in the form:

~~~              
0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+---------...----------+~~~~~~~~~+---------------+
|   RuleID    | Compression Residue  | Payload | SCHC padding  | 
+-+-+-+-+-+-+-+---------...----------+~~~~~~~~~+---------------+
|-------- Compressed Header ---------|         |-- as needed --|
~~~
{: #tab-diet-esp-compressed-header-format artwork-align="center" title="Diet-ESP Compressed Header Format"}

The SCHC Profile for Diet-ESP is defined as follows:

RuleID: 
: The RuleID is a unique identifier for each SCHC rule. It is included in packets to ensure the receiver applies the correct decompression rule, maintaining consistency in packet processing. Note that the Rule ID does not need to be explicitly agreed upon and can be defined independently by each party. The RuleID in Diet-ESP is expressed as 1 byte.

Maximum Packet Size:
: MAX_PACKET_SIZE is determined by the specific IPsec ESP configuration and the underlying transport, but it is typically aligned with the network’s MTU. The size constraints are optimized based on the available link capacity and negotiated parameters between endpoints.

SCHC Padding:
: Padding in SCHC is used to align data to a specific boundary (typically byte-aligned or 8-bit aligned) to meet the requirements of the underlying link layer protocol or encryption algorithm. Padding bits are often zero or follow a pattern but do not contain significant data. In Diet-ESP, the SCHC padding is added in the CTEC strata to align the packet for encryption.

The resulting IP/ESP packet size is variable. In some networks, the packet will require fragmentation before transmission over the wire. Fragmentation is out of the scope of this document since it is dependent on the layer 2 technology.

{{tab-diet-esp-compressed-pck}} illustrates how the final compressed packet looks when using SCHC compression for ESP headers in the Diet-ESP profile.



~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                  SCHC EEC Header (EEC stratum)                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ ----
|                 SCHC CTEC Header (CTEC stratum)               | ^
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |
|                 SCHC IIP Header (IIPC stratum)                | |
+---------------------------------------------------------------+ En-
|               Inner IP Payload Data* (variable)               | cry-
~                                                               ~ pted
|                                                               | |  
+---------------------------------------------------------------+ |
|                       SCHC CTEC Padding                       | v
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ ----
|                                                               |
|                             ICV                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~
{: #tab-diet-esp-compressed-pck artwork-align="center" title="Diet-ESP Compressed Packet Format with SCHC"}

## Set of Rules (SoR) for Diet-ESP

SCHC SoR are predefined sets of instructions that specify how to compress and decompress the header fields of an ESP packet. The identification of a particular SoR will follow the specification in {{!I-D.ietf-schc-architecture}}.

Similarly to the SA, Rules are directional and the Direction Indicator (DI) is set to Up for outbound SA and Down for inbound SA. Each Rule also contains a Field Position parameter that is set to 1, unless specified otherwise. 

## Attributes for Rules Generation {#sec-afrg}

The list of attributes for the Rules Generation (AfRG) is shown in {{tab-ehc-ctx-esp}}. These attributes are used to express the various compressions that operate at the IIPC, CTEC, and EEC layers.

The compression of the Inner IP Packet is based on the attributes that are derived from the negotiated Traffic Selectors TSi/TSr, as described in {{!RFC7296, Section 3.13}}. The Traffic Selectors may result in a quite complex expression, and this specification restricts that complexity. In particular, Diet-ESP restricts the Traffic Selector to a single type of IP address (i.e., IPv4 or IPv6), a single protocol (such as UDP, TCP, or not relevant), a single port range, and multiple DSCP numbers. Such simplification corresponds to the expression of an individual Traffic Selector Payload {{!RFC7296, Section 3.13.1}}.  

The ability to derive the EHCP Context for the IIPC from the agreed Traffic Selectors is indicated by the variable iipc_profile. 
 


| EHC Context       | Possible Values             | Reference | C / D |
|-------------------+-----------------------------+-----------+-------+
| iipc_profile      | "diet-esp", "uncompress"    | ThisRFC   | N/A   |
| dscp_cda          | "uncompress", "lower", "sa" | ThisRFC   | IIPC  | 
| ecn_cda           | "uncompress", "lower"       | ThisRFC   | IIPC  | 
| flow_label_cda    | "uncompress", "lower",      | ThisRFC   | IIPC  |
|                   | "generated", "zero"         |           |       | 
| ts_ip_version     | "IPv4-only IPv6-only"       | RFC7296   | IIPC  |
| ts_ip_src_start   | IP4 or IPv6 address         | RFC7296   | IIPC  |
| ts_ip_src_end     | IP4 or IPv6 address         | RFC7296   | IIPC  |
| ts_ip_dst_start   | IPv4 or IPv6 address        | RFC7296   | IIPC  |
| ts_ip_dst_end     | IPv4 or IPv6 address        | RFC7296   | IIPC  |
| ts_proto          | TCP, UDP, UDP-Lite, SCTP,   | RFC7296   | IIPC  |
|                   | ANY, ...                    |           |       |
| ts_port_src_start | Port number                 | RFC7296   | IIPC  |
| ts_port_src_end   | Port number                 | RFC7296   | IIPC  |
| ts_port_dst_start | Port number                 | RFC7296   | IIPC  |
| ts_port_dst_end   | Port number                 | RFC7296   | IIPC  |
| dscp_list         | list of DSCP numbers        | RFCYYYY   | IIPC  |
|-------------------+-----------------------------+-----------+-------+
| alignment         | "8 bit", "16 bit", "32 bit" | ThisRFC   | CTEC  |
|                   | "64 bit"                    |           |       |
| ipsec_mode        | "Tunnel", "Transport"       | RFC4301   | CTEC  | 
| tunnel_ip         | IPv6 address                | RFC4301   | CTEC  |
| esp_encr          | ESP Encryption Algorithm    | RFC4301   | CTEC  |
|-------------------+-----------------------------+-----------+-------+
| esp_spi           | ESP SPI                     | RFC4301   | EEC   |
| esp_spi_lsb       | 0-32                        | ThisRFC   | EEC   |
| esp_sn            | ESP Sequence Number         | RFC4301   | EEC   |
| esp_sn_lsb        | 0-64                        | ThisRFC   | EEC   |
|-------------------+-----------------------------+-----------+-------+
{: #tab-ehc-ctx-esp title="EHCP related parameters"}

Any parameter starting with "ts_" is associated with the Traffic Selectors of the SA. The notation is introduced by this specification but the definition of the parameters is in {{!RFC4301}} and {{!RFC7296}}.

This specification limits the expression of the Traffic Selector to be of the form (IP source range, IP destination range, Port source range, Port destination range, Protocol ID list, DSCP list). This limits the original flexibility of the expression of TS, but provides sufficient flexibility. 

The details of each parameter are the following:  

iipc_profile:
: designates the profile used by IIPC. When set to "uncompress" IIPC is not performed. This specification describes IIPC that corresponds to the "diet-esp" profile.
 
flow_label_cda:
: indicates how the Flow Label field of the inner IPv6 packet or the Identification field of the inner IPv4 packet is compressed / decompressed - See {{sec-cda}} for more information. In a nutshell, "uncompress" indicates that Flow Label (resp. Identification) are not compressed. "lower" indicates the value is read from the outer IP header - eventually with some adaptations when inner IP packet and outer IP packets have different versions. "generated" indicates that the field is generated by the receiving party. In that case, the decompressed value may take a different value compared to its original value. "zero" indicates the field is set to zero.

dscp_cda:
: indicates how the DSCP values of the inner IP packet are generated. (See flow_label_cda). "sa" indicates, compression is performed according to the DSCP values agreed by the SA (dscp_list). 

ecn_cda:
: indicates how the ECN values of the inner IP packet are generated. (See flow_label_cda). 

ts_ip_version:
: designates the IP version of the Traffic Selectors and its value is set  to "IPv4-only" when only IPv4 IP addresses are considered and to "IPv6-only" when only IPv6 addresses are considered.
Practically, when IKEv2 is used, it means that the agreed TSi or TSr results only in a mutually exclusive combination of TS_IPV4_ADDR_RANGE or TS_IPV6_ADDR_RANGE payloads.

ts_ip_src_start:
: designates the starting value range of source IP addresses of the inner packet and has the same meaning as the Starting Address field of the Traffic Selector payload defined in {{!RFC7296, Section 3.13}}.

ts_ip_src_end:
: designates the high end value range of source IP addresses of the inner packet and has the same meaning as the Ending Address field of the Traffic Selector payload defined in {{!RFC7296, Section 3.13}}.

ts_port_src_start:
: designates the starting value of the port range of the inner packet and has the same meaning as the Start Port field of the Traffic Selector payload defined in {{!RFC7296, Section 3.13}}.

ts_port_src_end:
: designates the starting value of the port range of the inner packet and has the same meaning as the End Port field of the Traffic Selector payload defined in {{!RFC7296, Section 3.13}}.

IP addresses and ports are defined as a range and compressed using the LSB.
For a range defined by start and end values, msb( start, end ) is defined as the function that returns the MSB that remains unchanged while the value evolves between start and end.
Similarly, lsb( start, end ) is defined as the function that returns the LSB that changes while the value evolves between start and end. 
Finally, len( x ) is defined as the function that returns the number of bits of the bit array x.

ts_proto:
: designates the list of Protocol ID field, whose meaning is defined in {{!RFC7296, Section 3.13}}. This profile considers the specific protocols values "TCP", "UDP", "UDP-Lite", "SCTP" and "ANY". "ANY" designates any possible values as defined in {{!RFC5996, Section 3.13}}. 

dscp_list:
: designates the list of DSCP values with the same meaning as the List of DSCP Values defined in {{?I-D.mglt-ipsecme-dscp-np}}. These are not Traffic Selectors, but the compression mandates that the packets take one of these listed DSCP values. 


alignment:
: indicates the byte alignement supported by the OS for the ESP extension. By default, the alignment is 32 bit for IPv6, but some systems may also support an 8 bit alignment. Note that when a block cipher such as AES-CCM is used, a 128 bit alignment is overwritten by the block size. 

ipsec_mode:
: designates the IPsec mode defined in {{!RFC4301}}. In this document, the possible values are "tunnel" for the Tunnel mode and "transport" for the Transport mode. 

tunnel_ip:
: designates the IP address of the tunnel defined in {{!RFC4301}}.
This field is only applicable when the Tunnel mode is used.
That IP address can be an IPv4 or IPv6 address. 

esp_encr:
: designates the encryption algorithm used. For the purpose of compression it is RECOMMENDED to use algorithms that already compresse their IV {{!RFC8750}}.

esp_spi:
: designates the Security Policy Index defined in {{!RFC4301}}. 

esp_spi_lsb: 
: designates the LSB to be considered for the compressed SPI. A value of 32 for esp_spi_lsb will leave the SPI unchanged.

esp_sn:
: designates the Sequence Number (SN) field defined in {{!RFC4301}}.

esp_sn_lsb:
: designates the LSB to be considered for the compressed SN and is defined by this specification. It works similarly to esp_spi_lsb. 


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
: is only used in a Tunnel mode and indicates that the fields of the inner IP packet header are generated from the corresponding fields of the Tunnel IP header fields. This CDA can be used for the DSCP, ECN, and IPv6 Flow Label (resp. IPv4 identification) fields.

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

When iipc_profile is set to "uncompress", the packet is uncompressed. 
When iipc_profile is set to "diet-esp", IIPC proceeds to the compression of the inner IP Packet composed of an IP Header and an IP Payload.
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
The field Total Length (16 bits) is compressed similarly to the IPv6 field Payload Length. The field Identification (16 bits) is compressed similarly to the IPv6 field Flow Label. If the IP Header is an IPv6 Header, the Identification is placed as the LSB of the IPv6 Header and the 4 remaining MSB are set to 0.  The field Time to Live is compressed similarly to the IPv6 Hop Limit field. The Protocol field is compressed similarly to the last IPv6 Next Header field.


IHL is uncompressed, FL is set to 4 bits, TV is not set, MO is set to ignore and CDA is set to "value-sent".
 
The IPv4 Header checksum is elided.
FL is set to 16, TV is omitted, MO is set to "ignore," and CDA is set to "checksum."


## ESP Data Byte alignment {#sec-byte-align}

SCHC operates on bits, while protocols like ESP expect payloads to be aligned to byte boundaries (8-bit alignment). To ensure this, we apply a padding by appending the SCHC_padding bits and the SCHC_padding_len. SCHC_padding_len is encoded over 3 bits to encode the values 0-7. SCHC_padding is randomly generated. Let's call the complementing bits, the bits that are needed to have a byte boundary. If the complementing bits are less than or equal to 2 bits, the padding will result in adding an extra byte.

## Clear Text ESP Compression (CTEC) {#sec-ctec}
    
The Clear Text ESP Compression is applied to compress the unencrypted ESP fields, including the ESP Payload Data and the Next Header field, which indicates the type of the inner packet.

SCHC Compression efficiently compresses the Next Header field, reducing overhead and aligning the packet to byte boundaries using SCHC Padding. After this, the SCHC Pad Length field is added. If required by the encryption algorithm, additional ESP Padding and Pad Length fields are introduced to ensure the packet fits the specified encryption block size.

In tunnel mode, the Next Header field is elided as the inner IP packet (either IPv4 or IPv6) is determined by the Traffic Selector, which is expressed by a single Traffic Selector Payload in the SA.


The ESP Padding and Pad Length fields are reduced by the SCHC compression rule. This process minimizes the size of the Clear Text ESP packet by eliminating unnecessary padding, while maintaining proper alignment for transmission. The ESP Padding and Pad Length may differ from the decompressed versions due to alignment requirements, which depend on the maximum of the encryption block alignment or the IPv6 Header alignment (32 bits). Since the padding is stripped by the IPsec process, these differences do not impact packet processing. This is expressed as FL is defined by the type that is to (Pad Length + 1 ) * 8 bits, TV is unset, MO is set to "ignore" and CDA is set to padding.


## Encrypted ESP Compression (EEC) {#sec-eec}


SPI is compressed to its LSB.
FL is set to 32 bits, TV is not set, MO is set to "MSB( 4 - esp_spi_lsb)" and CDA is set to "LSB".

If the esp_encr considers implicit IV {{!RFC8750}}, Sequence Numbers are not compressed. Otherwise, SN are compressed to their LSB similarly to the SPI. 
FL is set to 32 bits, TV is not set, MO is set to "MSB( 4 - esp_spi_lsb)" and CDA is set to "LSB".

Note that the use of implicit IV always results in a better compression as a 64 bit IV to be sent while compression of the SN alone results at best in a reduction of 32 bits. 

The IPv6 Next Header field or the IPv4 Protocol field that contains the "ESP" value is changed to "SCHC".
  
  
# SCHC Compression for IPsec in Transport mode

The transport mode mostly differs from the Tunnel mode in that the IP header of the packet is not encrypted. As a result, the IP Payload is compressed as described in {{sec-payload}}. The IP header is not compressed. The byte alignment of the Compressed Payload is performed as described in {{sec-byte-align}}. The Clear Text ESP Compression is performed as described in {{sec-ctec}} except for the Next Header Field, which is compressed as described in {{sec-inner-ip6}}.

# IANA Considerations

We request the IANA to create a new registry for the IIPC Profile


~~~
| IIPC Profile value | Reference |
+--------------------+-----------+
| "uncompress"       | ThisRFC   |
| "diet-esp"         | ThisRFC   |
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


# Security Considerations

There are no specific considerations associated with the profile other than the security considerations of ESP {{!RFC4303}} and those of SCHC {{!RFC8724}}.

# Acknowledgements

We would like to thank Laurent Toutain for his guidance on SCHC. Robert Moskowitz for inspiring the name "Diet-ESP" from Diet-HIP. The authors would like to acknowledge the support from Mitacs through the Mitacs Accelerate program.

--- back 



# Appendix

This appendix provides the details of the SCHC rules defined for Diet-ESP compression, alongside an explanation and an example outcome.


## JSON Representation of SCHC Rules for Diet-ESP Header Compression

The JSON file defines a set of rules within the SCHC_Context that are used for compressing and decompressing ESP headers. Each rule has a RuleID, a Description, and a set of Fields. Each field specifies how a particular part of the packet should be handled during compression or decompression. Note that the RuleID can be set by the user in any numeric order.
The rules include all the compression_levels, including IIPC, CTEC, and EEC as defined in the Terminology section.

~~~json
[
  {
    "RuleIDValue": 1,
    "RuleIDLength": 8,
    "Compression": [
      {
        "FID": "ESP.SPI",
        "TV": 5,
        "MO": "equal",
        "CDA": "not-sent"
      },
      {
        "FID": "ESP.SEQ",
        "TV": 1,
        "MO": "MSB",
        "MO.VAL": 16,
        "CDA": "LSB"
      }
    ]
  },
  {
    "RuleIDValue": 2,
    "RuleIDLength": 8,
    "Compression": [
      {
        "FID": "UDP.DEV_PORT",
        "TV": 123,
        "MO": "MSB",
        "MO.VAL": 12,
        "CDA": "LSB"
      },
      {
        "FID": "UDP.APP_PORT",
        "TV": 4567,
        "MO": "MSB",
        "MO.VAL": 12,
        "CDA": "LSB"
      },
      {
        "FID": "UDP.LEN",
        "TV": 0,
        "MO": "ignore",
        "CDA": "compute-length"
      },
      {
        "FID": "UDP.CKSUM",
        "TV": 0,
        "MO": "ignore",
        "CDA": "compute-checksum"
      }
    ]
  },
  {
    "RuleIDValue": 0,
    "RuleIDLength": 0,
    "schc_header": [
      {
        "FID": "SCHC.NXT",
        "TV": [17, 50, 41],
        "MO": "match-mapping",
        "CDA": "mapping-sent"
      }
    ]
  },
  {
    "RuleIDValue": 4,
    "RuleIDLength": 8,
    "Compression": [
      {
        "FID": "IPV6.DEV_PREFIX",
        "TV": "ff02::5678",
        "MO": "equal",
        "CDA": "value-sent"
      },
      {
        "FID": "IPV6.APP_PREFIX",
        "TV": "2001:db8::1000",
        "MO": "equal",
        "CDA": "value-sent"
      }
    ]
  },
  {
    "RuleIDValue": 5,
    "RuleIDLength": 8,
    "Compression": [
      {
        "FID": "ESP.NXT",
        "TV": 41,
        "MO": "equal",
        "CDA": "not-sent"
      }
    ]
  }
]

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