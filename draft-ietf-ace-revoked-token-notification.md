---
v: 3

title: Notification of Revoked Access Tokens in the Authentication and Authorization for Constrained Environments (ACE) Framework
abbrev: Notification of Revoked Tokens in ACE
docname: draft-ietf-ace-revoked-token-notification-latest


# stand_alone: true

ipr: trust200902
area: Security
wg: ACE Working Group
kw: Internet-Draft
cat: std
submissiontype: IETF

coding: utf-8
pi:    # can use array (if all yes) or hash here

  toc: yes
  sortrefs:   # defaults to yes
  symrefs: yes

author:
      -
        ins: M. Tiloca
        name: Marco Tiloca
        org: RISE AB
        street: Isafjordsgatan 22
        city: Kista
        code: SE-16440
        country: Sweden
        email: marco.tiloca@ri.se
      -
        ins: F. Palombini
        name: Francesca Palombini
        org: Ericsson AB
        street: Torshamnsgatan 23
        city: Kista
        code: SE-16440
        country: Sweden
        email: francesca.palombini@ericsson.com
      -
        ins: S. Echeverria
        name: Sebastian Echeverria
        org: CMU SEI
        street: 4500 Fifth Avenue
        city: Pittsburgh, PA
        code: 15213-2612
        country: United States of America
        email: secheverria@sei.cmu.edu
      -
        ins: G. Lewis
        name: Grace Lewis
        org: CMU SEI
        street: 4500 Fifth Avenue
        city: Pittsburgh, PA
        code: 15213-2612
        country: United States of America
        email: glewis@sei.cmu.edu

normative:
  RFC2119:
  RFC3629:
  RFC4648:
  RFC6347:
  RFC6749:
  RFC6838:
  RFC6920:
  RFC7120:
  RFC7252:
  RFC7641:
  RFC8259:
  RFC7519:
  RFC8126:
  RFC8174:
  RFC8392:
  RFC8446:
  RFC8610:
  RFC8613:
  RFC8725:
  RFC8949:
  RFC9147:
  RFC9200:
  RFC9528:
  RFC9202:
  RFC9203:
  RFC9290:
  RFC9431:
  RFC9528:
  Named.Information.Hash.Algorithm:
    author:
      org: IANA
    date: false
    title: Named Information Hash Algorithm
    target: https://www.iana.org/assignments/named-information/named-information.xhtml

informative:
  RFC7009:
  I-D.ietf-core-conditional-attributes:
  I-D.bormann-t2trg-stp:

entity:
  SELF: "[RFC-XXXX]"

--- abstract

This document specifies a method of the Authentication and Authorization for Constrained  Environments (ACE) framework, which allows an Authorization Server to notify Clients and Resource Servers (i.e., registered devices) about revoked access tokens. As specified in this document, the method allows Clients and Resource Servers to access a Token Revocation List on the Authorization Server by using the Constrained Application Protocol (CoAP), with the possible additional use of resource observation. Resulting (unsolicited) notifications of revoked access tokens complement alternative approaches such as token introspection, while not requiring additional endpoints on Clients and Resource Servers.

--- middle

# Introduction # {#intro}

Authentication and Authorization for Constrained Environments (ACE) {{RFC9200}} is a framework that enforces access control on IoT devices acting as Resource Servers. In order to use ACE, both Clients and Resource Servers have to register with an Authorization Server (AS) and become a registered device. Once registered, a Client can send a request to the AS, to obtain an access token for a Resource Server (RS). For a Client to access the RS, the Client must present the issued access token at the RS, which then validates it before storing it (see {{Section 5.10.1.1 of RFC9200}}).

Even though access tokens have expiration times, there are circumstances by which an access token may need to be revoked before its expiration time, such as: (1) a registered device has been compromised, or is suspected of being compromised; (2) a registered device is decommissioned; (3) there has been a change in the ACE profile for a registered device; (4) there has been a change in access policies for a registered device; and (5) there has been a change in the outcome of policy evaluation for a registered device (e.g., if policy assessment depends on dynamic conditions in the execution environment, the user context, or the resource utilization).

As discussed in {{Section 6.1 of RFC9200}}, only client-initiated revocation is currently specified {{RFC7009}} for OAuth 2.0 {{RFC6749}}, based on the assumption that access tokens in OAuth are issued with a relatively short lifetime. However, this is not expected to be the case for constrained, intermittently connected devices, that need access tokens with relatively long lifetimes.

This document specifies a method for allowing registered devices to access and possibly subscribe to a Token Revocation List (TRL) on the AS, in order to obtain updated information about pertaining access tokens that were revoked prior to their expiration. As specified in this document, the registered devices use the Constrained Application Protocol (CoAP) {{RFC7252}} to communicate with the AS and with one another, and can subscribe to the TRL on the AS by using resource observation for CoAP {{RFC7641}}. Other underlying protocols than CoAP are not prohibited from being supported in the future, if they are defined to be used in the ACE framework for Authentication and Authorization.

Unlike in the case of token introspection (see {{Section 5.9 of RFC9200}}), a registered device does not provide an owned access token to the AS for inquiring about its current state. Instead, registered devices simply obtain updated information about pertaining access tokens that were revoked prior to their expiration, as efficiently identified by corresponding hash values.

The benefits of this method are that it complements token introspection, and it does not require the registered devices to support any additional endpoints (see {{terminology}}). The only additional requirements for registered devices are a request/response interaction with the AS to access and possibly subscribe to the TRL (see {{sec-overview}}), and the lightweight computation of hash values to use as access token identifiers (see {{sec-token-name}}).

The process by which access tokens are declared revoked is out of the scope of this document. It is also out of scope the method by which the AS determines or is notified of revoked access tokens, according to which the AS consequently updates the TRL as specified in this document.

## Terminology ## {#terminology}

{::boilerplate bcp14-tagged}

Readers are expected to be familiar with the terms and concepts described in the ACE framework for Authentication and Authorization {{RFC9200}}, as well as with terms and concepts related to CBOR Web Tokens (CWTs) {{RFC8392}} and JSON Web Tokens (JWTs) {{RFC7519}}.

The terminology for entities in the considered architecture is defined in OAuth 2.0 {{RFC6749}}. In particular, this includes Client, Resource Server (RS), and Authorization Server (AS).

Readers are also expected to be familiar with the terms and concepts related to CDDL {{RFC8610}}, CBOR {{RFC8949}}, JSON {{RFC8259}}, CoAP {{RFC7252}}, CoAP Observe {{RFC7641}}, and the use of hash functions to name objects as defined in {{RFC6920}}.

Note that the term "endpoint" is used here following its OAuth definition {{RFC6749}}, aimed at denoting resources such as /token and /introspect at the AS, and /authz-info at the RS. This document does not use the CoAP definition of "endpoint", which is "An entity participating in the CoAP protocol."

This specification also refers to the following terminology.

* Token hash: identifier of an access token, in binary format encoding. The token hash has no relation to other access token identifiers possibly used, such as the 'cti' (CWT ID) claim of CBOR Web Tokens (CWTs) {{RFC8392}}.

* Token Revocation List (TRL): a collection of token hashes such that the corresponding access tokens have been revoked but are not expired yet.

* TRL endpoint: an endpoint at the AS with a TRL as its representation. The default name of the TRL endpoint in a url-path is '/revoke/trl'. Implementations are not required to use this name, and can define their own instead.

* Registered device: a device registered at the AS, i.e., as a Client, or an RS, or both. A registered device acts as a requester towards the TRL endpoint.

* Administrator: entity authorized to get full access to the TRL at the AS, and acting as a requester towards the TRL endpoint. An administrator is not necessarily a registered device as defined above, i.e., a Client requesting access tokens or an RS consuming access tokens.

* Pertaining access token:

   - With reference to an administrator, an access token issued by the AS.

   - With reference to a registered device, an access token intended to be owned by that device. An access token pertains to a Client if the AS has issued the access token for that Client following its request. An access token pertains to an RS if the AS has issued the access token to be consumed by that RS.

* Token hash pertaining to a requester: a token hash corresponding to an access token pertaining to that requester, i.e., an administrator or a registered device.

* TRL update pertaining to a requester: an update to the TRL through which token hashes pertaining to that requester have been added to the TRL or removed from the TRL.

Examples throughout this document are expressed in CBOR diagnostic notation as defined in {{Section 8 of RFC8949}} and {{Section G of RFC8610}}. Diagnostic notation comments are often used to provide a textual representation of the numeric parameter names and values.

In the CBOR diagnostic notation used in this document, constructs of the form e'SOME_NAME' are replaced by the value assigned to SOME_NAME in the CDDL model shown in {{fig-cddl-model}} of {{sec-cddl-model}}. For example, {e'full_set': \[\], e'cursor': 3} stands for {0: \[\], 2: 3}.

Note to RFC Editor: Please delete the paragraph immediately preceding this note. Also, in the CBOR diagnostic notation used in this document, please replace the constructs of the form e'SOME_NAME' with the value assigned to SOME_NAME in the CDDL model shown in {{fig-cddl-model}} of {{sec-cddl-model}}. Finally, please delete this note.

# Protocol Overview # {#sec-overview}

This protocol defines how a CoAP-based Authorization Server informs Clients and Resource Servers, i.e., registered devices, about pertaining revoked access tokens. How the relationship between a registered device and the AS is established is out of the scope of this specification.

At a high level, the steps of this protocol are as follows.

* Upon startup, the AS creates a single TRL accessible through the TRL endpoint. At any point in time, the TRL represents the list of all revoked access tokens issued by the AS that are not expired yet.

* When a device registers at the AS, it also receives the url-path to the TRL endpoint.

  At any time after the registration procedure is finished, the registered device can send a GET request to the TRL endpoint at the AS. When doing so, it can request for: the current list of pertaining revoked access tokens (see {{ssec-trl-full-query}}); or the most recent updates that occurred over the list of pertaining revoked access tokens (see {{ssec-trl-diff-query}}).

  In particular, the registered device can rely on Observation for CoAP {{RFC7641}}. In such a case, the GET request sent to the TRL endpoint includes the CoAP Observe Option set to 0 (register), i.e., it is an Observation Request. By doing so, the registered device effectively subscribes to the TRL, as interested in receiving notifications about its update. Upon receiving the Observation Request, the AS adds the registered device to the list of observers of the TRL endpoint.

* When an access token is revoked, the AS adds the corresponding token hash to the TRL. Also, when a revoked access token eventually expires, the AS removes the corresponding token hash from the TRL.

   In either case, after updating the TRL, the AS sends Observe notifications as per {{RFC7641}}. That is, an Observe notification is sent to each registered device subscribed to the TRL and to which the access token pertains.

   Depending on the specific subscription established through the Observation Request, the notification provides the current updated list of revoked access tokens in the subset of the TRL pertaining to that device (see {{ssec-trl-full-query}}), or the most recent TRL updates occurred over that list of pertaining revoked access tokens (see {{ssec-trl-diff-query}}).

   Further Observe notifications may be sent, consistently with ongoing additional observations of the TRL endpoint.

* An administrator can access and subscribe to the TRL like a registered device, while getting the content of the whole TRL (see {{ssec-trl-full-query}}) or the most recent updates occurred to the whole TRL (see {{ssec-trl-diff-query}}).

{{fig-protocol-overview}} shows a high-level overview of the service provided by this protocol. For the sake of simplicity, the example shown in the figure considers the simultaneous revocation of the three access tokens t1, t2, and t3, whose corresponding token hashes are th1, th2, and th3, respectively. Consequently, the AS adds the three token hashes to the TRL at once, and sends Observe notifications to one administrator and four registered devices. Each dotted line associated with a pair of registered devices indicates the access token that they both own.

~~~~~~~~~~~ aasvg
                    +----------------------+
                    | Authorization Server |
                    +-----------o----------+
                  /revoke/trl   |   TRL: (th1,th2,th3)
                                |
 +-----------------+------------+------------+------------+
 |                 |            |            |            |
 | th1,th2,th3     | th1,th2    | th1        | th3        | th2,th3
 v                 v            v            v            v
+---------------+ +----------+ +----------+ +----------+ +----------+
| Administrator | | Client 1 | | Resource | | Client 2 | | Resource |
|               | |          | | Server 1 | |          | | Server 2 |
+---------------+ +----------+ +----------+ +----------+ +----------+
                     :    :        :           :            :    :
                     :    :   t1   :           :     t3     :    :
                     :    :........:           :............:    :
                     :                   t2                      :
                     :...........................................:
~~~~~~~~~~~
{: #fig-protocol-overview title="Protocol Overview" artwork-align="center"}

{{sec-RS-examples}} provides examples of the protocol flow and message exchanges between the AS and a registered device.

# Token Hash # {#sec-token-name}

This section specifies how token hashes are computed.

First, {{sec-token-hash-input-motivation}} provides the motivation for the used construction.

Building on that, the value used as input to compute a token hash is defined in {{sec-token-hash-input-c-as}} for the Client and the AS, and in {{sec-token-hash-input-rs}} for the RS. Finally, {{sec-token-hash-output}} defines how such an input is used for computing the token hash.

The process outlined below refers to the base64url encoding and decoding without padding (see {{Section 5 of RFC4648}}), and denotes as "binary representation" of a text string the corresponding UTF-8 encoding {{RFC3629}}, which is the implied charset used in JSON (see {{Section 8.1 of RFC8259}}).

## Motivation for the Used Construction # {#sec-token-hash-input-motivation}

An access token can have one among different formats. The most expected formats are CWT {{RFC8392}} and JWT {{RFC7519}}, with the former being the default format to use in the ACE framework (see {{Section 3 of RFC9200}}). While access tokens are opaque to Clients, an RS is aware of whether access tokens that are issued for it to consume are either CWTs or JWTs.

### Issuing of the Access Token to the Client

There are two possible encodings that the AS can use for the AS-to-Client response (see {{Section 5.8.2 of RFC9200}}), where the issued access token is included and provided to the requester Client. The RS may not be aware of which encoding is used for that response to that particular requester Client.

* One way relies on CBOR, which is required if CoAP is used (see {{Section 5 of RFC9200}}) and is recommended otherwise (see {{Section 3 of RFC9200}}). That is, the AS-to-Client response has media-type "application/ace+cbor".

   This implies that, within the CBOR map specified as message payload, the parameter 'access_token' is a CBOR data item of type CBOR byte string and with value the binary representation BYTES of the access token. In particular:

   * If the access token is a CWT, then BYTES is the binary representation of the CWT (i.e., of the CBOR array that encodes the CWT).

   * If the access token is a JWT, then BYTES is the binary representation of the JWT (i.e., of the text string that encodes the JWT).

* An alternative way relies on JSON. That is, the AS-to-Client response has media-type "application/ace+json".

  This implies that, within the JSON object specified as message payload, the parameter 'access_token' has as value a text string TEXT encoding the access token. In particular:

  * If the access token is a JWT, then TEXT is the text string that encodes the JWT.

  * If the access token is a CWT, then TEXT is the base64url-encoded text string of the binary representation of the CWT (i.e., of the CBOR array that encodes the CWT).

### Provisioning of Access Tokens to the RS # {#sec-token-hash-input-motivation-rs}

In accordance with the used transport profile of ACE (e.g., {{RFC9202}}, {{RFC9203}}, {{RFC9431}}), the RS receives a piece of token-related information hereafter denoted as TOKEN_INFO.

In particular:

* If the AS-to-Client response was encoded in CBOR, then TOKEN_INFO is the value of the CBOR byte string conveyed by the 'access_token' parameter of that response. This is irrespective of the access token being a CWT or a JWT. That is, TOKEN_INFO is the binary representation of the access token.

* If the AS-to-Client response was encoded in JSON and the access token is a JWT, then TOKEN_INFO is the binary representation of the text string conveyed by the 'access_token' parameter of that response. That is, TOKEN_INFO is the binary representation of the access token.

* If the AS-to-Client response was encoded in JSON and the access token is a CWT, then TOKEN_INFO is the binary representation of the base64url-encoded text string that encodes the binary representation of the access token. That is, TOKEN_INFO is the binary representation of the base64url-encoded text string conveyed by the 'access_token' parameter.

The following overviews how the above specifically applies to the existing transport profiles of ACE.

* The access token can be uploaded to the RS by means of a POST request to the /authz-info endpoint (see {{Section 5.10.1 of RFC9200}}), using a CoAP Content-Format or HTTP media-type that reflects the format of the access token, if available (e.g., "application/cwt" for CWTs), or "application/octet-stream" otherwise. When doing so (e.g., like in {{RFC9202}}), TOKEN_INFO is the payload of the POST request.

* The access token can be uploaded to the RS by means of a POST request to the /authz-info endpoint, using the media-type "application/ace+cbor". When doing so (e.g., like in {{RFC9203}}), TOKEN_INFO is the value of the CBOR byte string conveyed by the 'access_token' parameter, within the CBOR map specified as payload of the POST request.

* The access token can be uploaded to the RS during a DTLS session establishment, e.g., like it is defined in {{Section 3.2.2 of RFC9202}}. When doing so, TOKEN_INFO is the value of the 'psk_identity' field of the ClientKeyExchange message (when using DTLS 1.2 {{RFC6347}}), or of the 'identity' field of a PSKIdentity, within the PreSharedKeyExtension of a ClientHello message (when using DTLS 1.3 {{RFC9147}}).

* The access token can be uploaded to the RS within the MQTT CONNECT packet, e.g., like it is defined in {{Section 2.2.4.1 of RFC9431}}. When doing so, TOKEN_INFO is specified within the 'Authentication Data' field of the MQTT CONNECT packet, following the property identifier 22 (0x16) and the token length.

### Design Rationale

Considering the possible variants discussed above, it must always be ensured that the same HASH_INPUT value is used as input for generating the token hash of a given access token, by the AS that has issued the access token and by the registered devices to which the access token pertains (both Client and RS).

This is achieved by building HASH_INPUT according to the content of the 'access_token' parameter in the AS-to-Client responses, since that is what all among the AS, the Client, and the RS are able to see.

## Hash Input on the Client and the AS # {#sec-token-hash-input-c-as}

The Client and the AS consider the content of the 'access_token' parameter in the AS-to-Client response, where the access token is included and provided to the requester Client.

The following defines how the Client and the AS determine the HASH_INPUT value to use as input for computing the token hash of the conveyed access token, depending on the AS-to-Client response being encoded in CBOR (see {{sec-token-hash-input-c-as-cbor}}) or in JSON (see {{sec-token-hash-input-c-as-json}}).

Once determined HASH_INPUT, the Client and the AS use it to compute the token hash of the conveyed access token as defined in {{sec-token-hash-output}}.

### AS-to-Client Response Encoded in CBOR # {#sec-token-hash-input-c-as-cbor}

If the AS-to-Client response is encoded in CBOR, then HASH_INPUT is defined as follows:

* BYTES denotes the value of the CBOR byte string conveyed in the parameter 'access_token'.

  With reference to the example in {{fig-as-response-cbor}}, BYTES is the bytes \{0xd0 0x83 0x43 ... 0x64 0x3b\}.

  Note that BYTES is the binary representation of the access token, irrespective of this being a CWT or a JWT.

* HASH_INPUT_TEXT is the base64url-encoded text string that encodes BYTES.

* HASH_INPUT is the binary representation of HASH_INPUT_TEXT.

~~~~~~~~~~~
Header: Created (Code=2.01)
Content-Format: application/ace+cbor
Max-Age: 85800
Payload:
{
   / access_token / 1 : h'd08343a1010aa2044c53796d6d65
                          74726963313238054d99a0d7846e
                          762c49ffe8a63e0b5858b918a11f
                          d81e438b7f973d9e2e119bcb2242
                          4ba0f38a80f27562f400ee1d0d6c
                          0fdb559c02421fd384fc2ebe22d7
                          071378b0ea7428fff157444d45f7
                          e6afcda1aae5f6495830c5862708
                          7fc5b4974f319a8707a635dd643b',
   / token_type /  34 : 2 / PoP /,
   / expires_in /   2 : 86400,
   / ace_profile / 38 : 1 / coap_dtls /,
   / (remainder of the response omitted for brevity) /
}
~~~~~~~~~~~
{: #fig-as-response-cbor title="Example of AS-to-Client CoAP response using CBOR" artwork-align="left"}

### AS-to-Client Response Encoded in JSON # {#sec-token-hash-input-c-as-json}

If the AS-to-Client response is encoded in JSON, then HASH_INPUT is the binary representation of the text string conveyed by the 'access_token' parameter.

With reference to the example in {{fig-as-response-json}}, HASH_INPUT is the binary representation of "2YotnFZFEjr1zCsicMWpAA".

Note that:

* If the access token is a JWT, then HASH_INPUT is the binary representation of the JWT.

* If the access token is a CWT, then HASH_INPUT is the binary representation of the base64url-encoded text string that encodes the binary representation of the CWT.

~~~~~~~~~~~
HTTP/1.1 200 OK
Content-Type: application/ace+json
Cache-Control: no-store
Pragma: no-cache
Payload:
{
   "access_token" : "2YotnFZFEjr1zCsicMWpAA",
   "token_type"   : "pop",
   "expires_in"   : 86400,
   "ace_profile"  : "1"
}
~~~~~~~~~~~
{: #fig-as-response-json title="Example of AS-to-Client HTTP response using JSON" artwork-align="left"}


## HASH\_INPUT on the RS # {#sec-token-hash-input-rs}

The following defines how the RS determines the HASH_INPUT value to use as input for computing the token hash of an access token, depending on the RS using either CWTs (see {{sec-token-hash-input-rs-cwt}}) or JWTs (see {{sec-token-hash-input-rs-jwt}}).

### Access Tokens as CWTs # {#sec-token-hash-input-rs-cwt}

If the RS expects access tokens to be CWTs, then the RS performs the following steps.

1. The RS receives the token-related information TOKEN_INFO, in accordance with what is specified by the used profile of ACE (see {{sec-token-hash-input-motivation-rs}}).

2. The RS assumes that the Client received the access token in an AS-to-Client response encoded in CBOR (see {{sec-token-hash-input-c-as-cbor}}). Hence, the RS assumes TOKEN_INFO to be the binary representation of the access token.

3. The RS verifies the access token as per {{Section 5.10.1.1 of RFC9200}}. If the verification fails, then the RS does not discard the access token yet, and it instead moves to step 4.

   Otherwise, the RS stores the access token and computes the corresponding token hash, as defined in {{sec-token-hash-output}}. In particular, the RS considers HASH_INPUT_TEXT as the base64url-encoded text string that encodes TOKEN_INFO. Then, HASH_INPUT is the binary representation of HASH_INPUT_TEXT.

   After that, the RS stores the computed token hash as associated with the access token, and then terminates this algorithm.

4. The RS assumes that the Client received the access token in an AS-to-Client response encoded in JSON (see {{sec-token-hash-input-c-as-json}}). Hence, the RS assumes TOKEN_INFO to be the binary representation of HASH_INPUT_TEXT, which is the base64url-encoded text string that encodes the binary representation of the access token.

5. The RS performs the base64url decoding of HASH_INPUT_TEXT, and considers the result as the binary representation of the access token.

6. The RS verifies the access token as per {{Section 5.10.1.1 of RFC9200}}. If the verification fails, then the RS terminates this algorithm.

   Otherwise, the RS stores the access token and computes the corresponding token hash, as defined in {{sec-token-hash-output}}. In particular, HASH_INPUT is TOKEN_INFO.

   After that, the RS stores the computed token hash as associated with the access token.

### Access Tokens as JWTs # {#sec-token-hash-input-rs-jwt}

If the RS expects access tokens to be JWTs, then the RS performs the following steps.

1. The RS receives the token-related information TOKEN_INFO, in accordance with what is specified by the used profile of ACE (see {{sec-token-hash-input-motivation-rs}}).

2. The RS verifies the access token as per {{Section 5.10.1.1 of RFC9200}}. If the verification fails, then the RS terminates this algorithm. Otherwise, the RS stores the access token.

3. The RS computes a first token hash associated with the access token, as defined in {{sec-token-hash-output}}.

   In particular, the RS assumes that the Client received the access token in an AS-to-Client response encoded in JSON (see {{sec-token-hash-input-c-as-json}}). Hence, HASH_INPUT is TOKEN_INFO.

   After that, the RS stores the computed token hash as associated with the access token.

4. The RS computes a second token hash associated with the access token, as defined in {{sec-token-hash-output}}.

   In particular, the RS assumes that the Client received the access token in an AS-to-Client response encoded in CBOR (see {{sec-token-hash-input-c-as-cbor}}). Hence, HASH_INPUT is the binary representation of HASH_INPUT_TEXT, which in turn is the base64url-encoded text string that encodes TOKEN_INFO.

   After that, the RS stores the computed token hash as associated with the access token.

The RS skips step 3 only if it is certain that all its pertaining access tokens are provided to any Client by means of AS-to-Client responses encoded as CBOR messages. Otherwise, the RS MUST perform step 3.

The RS skips step 4 only if it is certain that all its pertaining access tokens are provided to any Client by means of AS-to-Client responses encoded as JSON messages. Otherwise, the RS MUST perform step 4.

If the RS performs both step 3 and step 4 above, then the RS MUST store, maintain, and rely on both token hashes as associated with the access token, consistent with what is specified in {{sec-handling-token-hashes}}.

{{sec-seccons-two-hashes-jwt}} discusses how computing and storing both token hashes neutralizes an attack against the RS, where a dishonest Client can induce the RS to compute a token hash different from the correct one.

## Computing the Token Hash # {#sec-token-hash-output}

Once determined HASH\_INPUT as defined in {{sec-token-hash-input-c-as}} and {{sec-token-hash-input-rs}}, a hash value of HASH\_INPUT is generated as per {{Section 6 of RFC6920}}. The resulting output in binary format is used as the token hash. Note that the used binary format embeds the identifier of the used hash function, in the first byte of the computed token hash.

The specifically used hash function MUST be collision-resistant on byte-strings, and MUST be selected from the "Named Information Hash Algorithm" Registry {{Named.Information.Hash.Algorithm}}.

The AS specifies the used hash function to registered devices during their registration procedure (see {{sec-registration}}).

# Token Revocation List (TRL) # {#sec-trl-resource}

Upon startup, the AS creates a single Token Revocation List (TRL), encoded as a CBOR array.

Each element of the array is a CBOR byte string, with value the token hash of an access token. The CBOR array MUST be treated as a set, i.e., the order of its elements has no meaning.

The TRL is initialized as empty, i.e., its initial content MUST be the empty CBOR array. The TRL is accessible through the TRL endpoint at the AS.

## Update of the TRL ## {#ssec-trl-update}

The AS updates the TRL in the following two cases.

* When a non-expired access token is revoked, the token hash of the access token is added to the TRL. That is, a CBOR byte string with the token hash as its value is added to the CBOR array encoding the TRL.

* When a revoked access token expires, the token hash of the access token is removed from the TRL. That is, the CBOR byte string with the token hash as its value is removed from the CBOR array encoding the TRL.

The AS MAY perform a single update to the TRL such that one or more token hashes are added or removed at once. For example, this can be the case if multiple access tokens are revoked or expire at the same time, or within an acceptably narrow time window.

# The TRL Endpoint # {#sec-trl-endpoint}

Consistent with {{Section 6.5 of RFC9200}}, all communications between a requester towards the TRL endpoint and the AS MUST be encrypted, as well as integrity and replay protected. Furthermore, responses from the AS to the requester MUST be bound to the corresponding requests.

Following a request to the TRL endpoint, the corresponding, success response messages sent by the AS use Content-Format "application/ace-trl+cbor". Their payload is formatted as a CBOR map, and the CBOR values used to abbreviate the parameters included therein are defined in {{trl-registry-parameters}}.

The AS MUST implement measures to prevent access to the TRL endpoint by entities other than registered devices and authorized administrators (see {{sec-registration}}).

The TRL endpoint supports only the GET method, and allows two types of queries of the TRL.

* Full query: the AS returns the token hashes of the revoked access tokens currently in the TRL and pertaining to the requester.

   The AS MUST support this type of query. The processing of a full query and the related response format are defined in {{ssec-trl-full-query}}.

* Diff query: the AS returns a list of diff entries. Each diff entry is related to one update occurred to the TRL, and it contains a set of token hashes pertaining to the requester. In particular, all such token hashes were added to the TRL or removed from the TRL at the update related to the diff entry in question.

   The AS MAY support this type of query. In such a case, the AS maintains the history of updates to the TRL as defined in {{sec-trl-endpoint-supporting-diff-queries}}. The processing of a diff query and the related response format are defined in {{ssec-trl-diff-query}}.

If it supports diff queries, the AS MAY additionally support its "Cursor" extension, which has two benefits. First, the AS can avoid excessively long messages when several diff entries have to be transferred, by delivering several diff query responses, each containing one adjacent subset of diff entries at a time. Second, a requester can retrieve diff entries associated with TRL updates that, even if not the most recent ones, occurred after a TRL update associated with a diff entry indicated as reference point.

If it supports the "Cursor" extension, the AS stores additional information when maintaining the history of updates to the TRL, as defined in {{sec-trl-endpoint-supporting-cursor}}. Also, the processing of full query requests and diff query requests, as well as the related response format, are further extended as defined in {{sec-using-cursor}}.

{{sec-trl-parameteters}} provides an aggregated overview of the local supportive parameters that the AS internally uses at its TRL endpoint, when supporting diff queries and the "Cursor" extension.

## Error Responses with Problem Details # {#sec-error-responses}

Some error responses from the TRL endpoint at the AS can convey error-specific information according to the problem-details format defined in {{RFC9290}}. Such error responses MUST have Content-Format set to "application/concise-problem-details+cbor". The payload of these error responses MUST be a CBOR map specifying a Concise Problem Details data item (see {{Section 2 of RFC9290}}). The CBOR map is formatted as follows.

* It MUST include the Custom Problem Detail entry 'ace-trl-error' registered in {{iana-custom-problem-details}} of this document. This entry is formatted as a CBOR map, which includes the following fields.

  - The field 'error-id' MUST be present. The map key used for this field is the CBOR unsigned integer with value 0. The value of this field is a CBOR integer specifying the error occurred at the AS. This value is taken from the 'Value' column of the "ACE Token Revocation List Errors" registry defined in {{iana-token-revocation-list-errors}} of this document.

  - The field 'cursor' MAY be present. The map key used for this field is the CBOR unsigned integer with value 1. The value of this field is a CBOR unsigned integer or the CBOR simple value `null` (0xf6).

  The CDDL notation {{RFC8610}} of the 'ace-trl-error' entry is given below.

~~~~~~~~~~~ CDDL
   ace-trl-error = {
       0: int,        ; error-id
     ? 1: uint / null ; cursor
   }
~~~~~~~~~~~

* It MAY include further Standard Problem Detail entries or Custom Problem Detail entries (see {{RFC9290}}).

  In particular, it can include the Standard Problem Detail entry 'detail' (map key -2), whose value is a CBOR text string that specifies a human-readable, diagnostic description of the error occurred at the AS. The diagnostic text is intended for software engineers as well as for device and network operators, in order to aid debugging and provide context for possible intervention. The diagnostic message SHOULD be logged by the AS. The 'detail' entry is unlikely relevant in an unattended setup where human intervention is not expected.

An example of error response using the problem-details format is shown in {{fig-example-error-response}}.

~~~~~~~~~~~
Header: Bad Request (Code=4.00)
Content-Format: application/concise-problem-details+cbor
Payload:
{
  / title /     -1: "Invalid parameter value",
  / detail /    -2: "Invalid value for 'cursor': -53",
  / ace-trl-error / e'ace-trl-error': {
    / error-id / 0: 0 / "Invalid parameter value" /,
    / cursor /   1: 42
  }
}
~~~~~~~~~~~
{: #fig-example-error-response title="Example of Error Response with Problem Details"}

The problem-details format in general and the Custom Problem Detail entry 'ace-trl-error' in particular are OPTIONAL to support for registered devices. A registered device supporting the entry 'ace-trl-error' and able to understand the specified error may use that information to determine what actions to take next.

## Supporting Diff Queries # {#sec-trl-endpoint-supporting-diff-queries}

If the AS supports diff queries, it is able to transfer a list of diff entries, each of which is related to one update occurred to the TRL (see {{sec-trl-endpoint}}). That is, when replying to a diff query performed by a requester, the AS specifies the diff entries related to the most recent TRL updates pertaining to the requester.

The following defines how the AS builds and maintains an ordered list of diff entries, for each registered device and administrator, hereafter referred to as requesters. In particular, a requester's diff entry associated with a TRL update contains a set of token hashes pertaining to that requester, which were added to the TRL or removed from the TRL at that update.

The AS defines the single, constant positive integer MAX\_N >= 1. For each requester, the AS maintains an update collection of maximum MAX\_N series items, each of which is a diff entry. For each requester, the AS MUST keep track of the MAX\_N most recent TRL updates pertaining to the requester. If the AS supports diff queries, the AS MUST provide requesters with the value of MAX\_N, upon their registration (see {{sec-registration}}).

The series items in the update collection MUST be strictly ordered in a chronological fashion. That is, at any point in time, the current first series item is the one least recently added to the update collection and still retained by the AS, while the current last series item is the one most recently added to the update collection. The particular method used to achieve this is implementation-specific.

Each time the TRL changes, the AS performs the following operations for each requester.

1. The AS considers the subset of the TRL pertaining to that requester. If the TRL subset is not affected by this TRL update, the AS stops the processing for that requester. Otherwise, the AS moves to step 2.

2. The AS creates two sets "trl_patch" of token hashes, i.e., one  "removed" set and one "added" set, as related to this TRL update.

3. The AS fills the two sets with the token hashes of the removed and added access tokens, respectively, from/to the TRL subset considered at step 1.

4. The AS creates a new series item, which includes the two sets from step 3.

5. If the update collection associated with the requester currently includes MAX\_N series items, the AS MUST delete the oldest series item in the update collection.

6. The AS adds the series item to the update collection associated with the requester, as the last (most recent) one.

### Supporting the "Cursor" Extension # {#sec-trl-endpoint-supporting-cursor}

If it supports the "Cursor" extension for diff queries, the AS performs also the following actions.

The AS defines the single, constant unsigned integer MAX\_INDEX <= ((2^64) - 1), where "^" is the exponentiation operator. The value of MAX\_INDEX is REQUIRED to be at least (MAX\_N - 1), and is RECOMMENDED to be at least ((2^32) - 1). MAX\_INDEX SHOULD be orders of magnitude greater than MAX\_N.

The following applies separately for each requester's update collection.

* Each series item X in the update collection is also associated with an unsigned integer 'index', whose minimum value is 0 and whose maximum value is MAX\_INDEX. The first series item ever added to the update collection MUST have 'index' with value 0.

   If i_X is the value of 'index' associated with a series item X, then the following series item Y will take 'index' with value i_Y = (i_X + 1) % (MAX\_INDEX + 1). That is, after having added a series item whose associated 'index' has value MAX\_INDEX, the next added series item will result in a wrap-around of the 'index' value, and will thus take 'index' with value 0.

   For example, assuming MAX\_N = 3, the values of 'index' in the update collection chronologically evolve as follows, as new series items are added and old series items are deleted.

   - ...
   - (i_A = MAX\_INDEX - 2, i_B = MAX\_INDEX - 1, i_C = MAX\_INDEX)
   - (i_B = MAX\_INDEX - 1, i_C = MAX\_INDEX, i_D = 0)
   - (i_C = MAX\_INDEX, i_D = 0, i_E = 1)
   - (i_D = 0, i_E = 1, i_F = 2)
   - ...

* The unsigned integer 'last_index' is also defined, with minimum value 0 and maximum value MAX\_INDEX.

   If the update collection is empty (i.e., no series items have been added yet), the value of 'last_index' is not defined. If the update collection is not empty, 'last_index' has the value of 'index' currently associated with the last series item in the update collection.

   That is, after having added V series items to the update collection, the last and most recently added series item has 'index' with value 'last_index' = (V - 1) % (MAX_INDEX + 1).

   As long as a wrap-around of the 'index' value has not occurred, the value of 'last_index' is the absolute counter of series items added to that update collection, minus 1.

When processing a diff query using the "Cursor" extension, the values of 'index' are used as cursor information, as defined in {{sec-using-cursor-diff-query-response}}.

For each requester's update collection, the AS also defines a constant, positive integer MAX_DIFF_BATCH <= MAX_N, whose value specifies the maximum number of diff entries to be included in a single diff query response. The specific value MAY depend on the specific registered device or administrator associated with the update collection in question. If supporting the "Cursor" extension, the AS MUST provide registered devices and administrators with the corresponding value of MAX_DIFF_BATCH, upon their registration (see {{sec-registration}}).

## Query Parameters # {#sec-trl-endpoint-query-parameters}

A GET request to the TRL endpoint can include the following query parameters. The AS MUST silently ignore unknown query parameters.

* 'diff': if included, it indicates to perform a diff query of the TRL (see {{ssec-trl-diff-query}}). Its value MUST be either:

   - the integer 0, indicating that a (notification) response should include as many diff entries as the AS can provide in the response; or

   - a positive integer strictly greater than 0, indicating the maximum number of diff entries that a (notification) response should include.

   If the AS does not support diff queries, it ignores the 'diff' query parameter when present in the GET request, and proceeds like when processing a full query of the TRL (see {{ssec-trl-full-query}}).

   Otherwise, the AS MUST return a 4.00 (Bad Request) response in case the 'diff' query parameter of the GET request specifies a value that is neither 0 nor a positive integer, irrespective of the presence of the 'cursor' parameter and its value (see below). The response MUST have Content-Format "application/concise-problem-details+cbor" and its payload is formatted as defined in {{sec-error-responses}}. Within the Custom Problem Detail entry 'ace-trl-error', the value of the 'error-id' field MUST be set to 0 ("Invalid parameter value"), and the field 'cursor' MUST NOT be present.

* 'cursor': if included, it indicates to perform a diff query of the TRL together with the "Cursor" extension, as defined in {{sec-using-cursor-diff-query-response}}. Its value MUST be either 0 or a positive integer. If the 'cursor' query parameter is included, then the 'diff' query parameter MUST also be included.

   If included, the 'cursor' query parameter specifies an unsigned integer value that was provided by the AS in a previous response from the TRL endpoint (see {{sec-using-cursor-full-query-response}}, {{sec-using-cursor-diff-query-response-no-cursor}}, and {{sec-using-cursor-diff-query-response-cursor}}).

   If the AS does not support the "Cursor" extension, it ignores the 'cursor' query parameter when present in the GET request. In such a case, the AS proceeds as specified elsewhere in this document, i.e.: i) it performs a diff query of the TRL (see {{ssec-trl-diff-query}}), if it supports diff queries and the 'diff' query parameter is present in the GET request; or ii) it performs a full query of the TRL (see {{ssec-trl-full-query}}) otherwise.

   If the AS supports both diff queries and the "Cursor" extension, and the GET request specifies the 'cursor' query parameter, then the AS MUST return a 4.00 (Bad Request) response in case any of the conditions below holds.

   The 4.00 (Bad Request) response MUST have Content-Format "application/concise-problem-details+cbor" and its payload is formatted as defined in {{sec-error-responses}}.

   * The GET request does not specify the 'diff' query parameter, irrespective of the value of the 'cursor' parameter.

      Within the Custom Problem Detail entry 'ace-trl-error', the value of the 'error-id' field MUST be set to 1 ("Invalid set of parameters"), and the field 'cursor' MUST NOT be present.

   * The 'cursor' query parameter has a value that is neither 0 nor a positive integer, or it has a value strictly greater than MAX_INDEX (see {{sec-trl-endpoint-supporting-cursor}}).

      Within the Custom Problem Detail entry 'ace-trl-error', the value of the 'error-id' field MUST be set to 0 ("Invalid parameter value"). The entry 'ace-trl-error' MUST include the field 'cursor', whose value is either: the CBOR simple value `null` (0xf6), if the update collection associated with the requester is empty; or the corresponding current value of 'last_index' otherwise.

   * All of the following hold: the update collection associated with the requester is not empty; no wrap-around of its 'index' value has occurred; and the 'cursor' query parameter has a value strictly greater than the current 'last_index' on the update collection (see {{sec-trl-endpoint-supporting-cursor}}).

      Within the Custom Problem Detail entry 'ace-trl-error', the value of the 'error-id' field MUST be set to 2 ("Out of bound cursor value"), and the field 'cursor' MUST NOT be present.

# Full Query of the TRL ## {#ssec-trl-full-query}

In order to produce a (notification) response to a GET request asking for a full query of the TRL, the AS performs the following actions.

1. From the TRL, the AS builds a set HASHES such that:

    * If the requester is a registered device, HASHES specifies the token hashes currently in the TRL and associated with the access tokens pertaining to that registered device. The AS can always use the authenticated identity of the registered device to perform the necessary filtering on the TRL content.

    * If the requester is an administrator, HASHES specifies all the token hashes currently in the TRL.

2. The AS sends a 2.05 (Content) response to the requester. The response MUST have Content-Format "application/ace-trl+cbor". The payload of the response is a CBOR map, which MUST be formatted as follows.

   * The 'full_set' parameter MUST be included and specifies a CBOR array 'full_set_value'. Each element of 'full_set_value' is a CBOR byte string, with value one of the token hashes from the set HASHES. If the set HASHES is empty, the 'full_set' parameter specifies the empty CBOR array.

      The CBOR array MUST be treated as a set, i.e., the order of its elements has no meaning.

   * The 'cursor' parameter MUST be included if the AS supports both diff queries and the related "Cursor" extension (see {{sec-trl-endpoint-supporting-diff-queries}} and {{sec-trl-endpoint-supporting-cursor}}). Its value is set as specified in {{sec-using-cursor-full-query-response}}, and provides the requester with information for performing a follow-up diff query using the "Cursor" extension (see {{sec-using-cursor-diff-query-response}}).

      If the AS does not support both diff queries and the "Cursor" extension, this parameter MUST NOT be included. In case the requester does not support both diff queries and the "Cursor" extension, it MUST silently ignore the 'cursor' parameter if present.

{{cddl-full}} provides the CDDL definition {{RFC8610}} of the CBOR array 'full_set_value' specified in the response from the AS, as value of the 'full_set' parameter.

~~~~~~~~~~~ CDDL
token_hash = bytes
full_set_value = [* token_hash]
~~~~~~~~~~~
{: #cddl-full title="CDDL definition of 'full_set_value'" artwork-align="left"}

{{response-full}} shows an example response from the AS, following a full query request to the TRL endpoint. In this example, the AS does not support diff queries nor the "Cursor" extension, hence the 'cursor' parameter is not included in the payload of the response. Also, full token hashes are omitted for brevity.

~~~~~~~~~~~
Header: Content (Code=2.05)
Content-Format: application/ace-trl+cbor
Payload:
{
   e'full_set' : [
     h'01fa51cc...4819', / elided for brevity /
     h'01748190...223d'  / elided for brevity /
   ]
}
~~~~~~~~~~~
{: #response-full title="Example of response following a full query request to the TRL endpoint" artwork-align="left"}

# Diff Query of the TRL ## {#ssec-trl-diff-query}

In order to produce a (notification) response to a GET request asking for a diff query of the TRL, the AS performs the following actions.

Note that, if the AS supports both diff queries and the related "Cursor" extension, the steps 3 and 4 defined below are extended as defined in {{sec-using-cursor-diff-query-response}}.

1. The AS defines the positive integer NUM as follows. If the value N specified in the 'diff' query parameter in the GET request is equal to 0 or greater than the pre-defined positive integer MAX\_N (see {{sec-trl-endpoint-supporting-diff-queries}}), then NUM takes the value of MAX_N. Otherwise, NUM takes N.

2. The AS determines U = min(NUM, SIZE), where SIZE <= MAX_N. In particular, SIZE is the number of diff entries currently stored in the requester's update collection.

3. The AS prepares U diff entries. If U is equal to 0 (e.g., because SIZE is equal to 0 at step 2), then no diff entries are prepared.

    The prepared diff entries are related to the U most recent TRL updates pertaining to the requester, as maintained in the update collection for that requester (see {{sec-trl-endpoint-supporting-diff-queries}}). In particular, the first diff entry refers to the most recent of such updates, the second diff entry refers to the second from last of such updates, and so on.

    Each diff entry is a CBOR array 'diff_entry', which includes the following two elements.

    * The first element is a 'trl_patch' set of token hashes, encoded as a CBOR array 'removed'. Each element of the array is a CBOR byte string, with value the token hash of an access token such that: it pertained to the requester; and it was removed from the TRL during the update associated with the diff entry.

    * The second element is a 'trl_patch' set of token hashes, encoded as a CBOR array 'added'. Each element of the array is a CBOR byte string, with value the token hash of an access token such that: it pertains to the requester; and it was added to the TRL during the update associated with the diff entry.

    The CBOR arrays 'removed' and 'added' MUST be treated as sets, i.e., the order of their elements has no meaning.

4. The AS prepares a 2.05 (Content) response for the requester. The response MUST have Content-Format "application/ace-trl+cbor". The payload of the response is a CBOR map, which MUST be formatted as follows.

   * The 'diff_set' parameter MUST be present and specifies a CBOR array 'diff_set_value' of U elements. Each element of 'diff_set_value' specifies one of the CBOR arrays 'diff_entry' prepared above as a diff entry. Note that U might have value 0, in which case 'diff_set_value' is the empty CBOR array.

      Within 'diff_set_value', the CBOR arrays 'diff_entry' MUST be sorted to reflect the corresponding updates to the TRL in reverse chronological order. That is, the first 'diff_entry' element of 'diff_set_value' relates to the most recent TRL update pertaining to the requester. The second 'diff_entry' element relates to the second from last most recent TRL update pertaining to the requester, and so on.

   * The 'cursor' parameter and the 'more' parameter MUST be included if the AS supports both diff queries and the related "Cursor" extension (see {{sec-trl-endpoint-supporting-cursor}}). Their values are set as specified in {{sec-using-cursor-diff-query-response}}, and provide the requester with information for performing a follow-up query of the TRL (see {{sec-using-cursor-diff-query-response}}).

      In case the AS supports diff queries but not the "Cursor" extension, these parameters MUST NOT be included. In case the requester supports diff queries but not the "Cursor" extension, it MUST silently ignore the 'cursor' parameter and the 'more' parameter if present.

{{cddl-diff}} provides the CDDL definition {{RFC8610}} of the CBOR array 'diff_set_value' specified in the response from the AS, as value of the 'diff_set' parameter.

~~~~~~~~~~~ CDDL
   token_hash = bytes
   trl_patch = [* token_hash]
   diff_entry = [removed: trl_patch, added: trl_patch]
   diff_set_value = [* diff_entry]
~~~~~~~~~~~
{: #cddl-diff title="CDDL definition of 'diff_set_value'" artwork-align="left"}

{{response-diff}} shows an example response from the AS, following a diff query request to the TRL endpoint, where U = 3 diff entries are specified. In this example, the AS does not support the "Cursor" extension, hence the 'cursor' parameter and the 'more' parameter are not included in the payload of the response. Also, full token hashes are omitted for brevity.

~~~~~~~~~~~
Header: Content (Code=2.05)
Content-Format: application/ace-trl+cbor
Payload:
{
   e'diff_set' : [
     [
       [ h'01fa51cc...0f6a', / elided for brevity /
         h'01748190...8bce'  / elided for brevity /
       ],
       [ h'01cdf1ca...563d', / elided for brevity /
         h'01be41a6...a057'  / elided for brevity /
       ]
     ],
     [
       [ h'0144dd12...77bc', / elided for brevity /
         h'01231fff...a2ce'  / elided for brevity /
       ],
       []
     ],
     [
       [],
       [ h'01ca986f...ffc1', / elided for brevity /
         h'01fe1a2b...def0'  / elided for brevity /
       ]
     ]
   ]
}
~~~~~~~~~~~
{: #response-diff title="Example of response following a diff query request to the TRL endpoint" artwork-align="left"}

{{sec-series-pattern}} discusses how performing a diff query of the TRL is in fact a usage example of the Series Transfer Pattern defined in {{I-D.bormann-t2trg-stp}}.

# Response Messages when Using the "Cursor" Extension ## {#sec-using-cursor}

If the AS supports both diff queries and the "Cursor" extension, it composes a response to a full query request or diff query request as defined in {{sec-using-cursor-full-query-response}} and {{sec-using-cursor-diff-query-response}}, respectively.

The exact format of the response depends on the request being a full query or diff query request, on the presence of the 'diff' and 'cursor' query parameters and their values in the diff query request, and on the current status of the update collection associated with the requester.

Error handling and the possible resulting error responses are as defined in {{sec-trl-endpoint-query-parameters}}.

## Response to Full Query {#sec-using-cursor-full-query-response}

When processing a full query request to the TRL endpoint, the AS composes a response as defined in {{ssec-trl-full-query}}.

In particular, the 'cursor' parameter included in the CBOR map carried in the response payload specifies either the CBOR simple value `null` (0xf6) or a CBOR unsigned integer.

The 'cursor' parameter MUST specify the CBOR simple value `null` in case there are currently no TRL updates pertaining to the requester, i.e., the update collection for that requester is empty. This is the case from when the requester registers at the AS until the first update pertaining to that requester occurs to the TRL.

Otherwise, the 'cursor' parameter MUST specify a CBOR unsigned integer. This MUST take the 'index' value of the last series item in the update collection associated with the requester (see {{sec-trl-endpoint-supporting-cursor}}), as corresponding to the most recent TRL update pertaining to the requester. Such a value is in fact the current value of 'last_index' for the update collection associated with the requester.

## Response to Diff Query {#sec-using-cursor-diff-query-response}

When processing a diff query request to the TRL endpoint, the AS composes a response as defined in the following.

### Empty Collection {#sec-using-cursor-diff-query-response-empty}

If the update collection associated with the requester has no elements, the AS returns a 2.05 (Content) response. The response MUST have Content-Format "application/ace-trl+cbor" and its payload MUST be a CBOR map formatted as follows.

* The 'diff_set' parameter MUST be included and specifies the empty CBOR array.

* The 'cursor' parameter MUST be included and specifies the CBOR simple value `null` (0xf6).

* The 'more' parameter MUST be included and specifies the CBOR simple value `false` (0xf4).

Note that the above applies when the update collection associated with the requester has no elements, regardless of whether the 'cursor' query parameter is included or not in the diff query request, and irrespective of the specified unsigned integer value if present.

### Cursor Not Specified in the Diff Query Request {#sec-using-cursor-diff-query-response-no-cursor}

If the update collection associated with the requester is not empty and the diff query request does not include the 'cursor' query parameter, the AS performs the actions defined in {{ssec-trl-diff-query}}, with the following differences.

* At step 3, the AS considers the value MAX_DIFF_BATCH (see {{sec-trl-endpoint-supporting-cursor}}), and prepares L = min(U, MAX_DIFF_BATCH) diff entries.

   If U <= MAX_DIFF_BATCH, the prepared diff entries are the last series items in the update collection associated with the requester, corresponding to the L most recent TRL updates pertaining to the requester.

   If U > MAX_DIFF_BATCH, the prepared diff entries are the eldest of the last U series items in the update collection associated with the requester, as corresponding to the first L of the U most recent TRL updates pertaining to the requester.

* At step 4, the CBOR map to carry in the payload of the 2.05 (Content) response MUST be formatted as follows.

   * The 'diff_set' parameter MUST be present and specifies a CBOR array 'diff_set_value' of L elements. Each element of 'diff_set_value' specifies one of the CBOR arrays 'diff_entry' prepared as a diff entry.

   * The 'cursor' parameter MUST be present and specifies a CBOR unsigned integer. This MUST take the 'index' value of the series item of the update collection included as first diff entry in the 'diff_set_value' CBOR array, which is specified by the 'diff_set' parameter. That is, the 'cursor' parameter takes the 'index' value of the series item in the update collection corresponding to the most recent TRL update pertaining to the requester and returned in this diff query response.

      Note that the 'cursor' parameter takes the same 'index' value of the last series item in the update collection when U <= MAX_DIFF_BATCH.

   * The 'more' parameter MUST be present and MUST specify the CBOR simple value `false` (0xf4) if U <= MAX_DIFF_BATCH, or the CBOR simple value `true` (0xf5) otherwise.

If the 'more' parameter in the payload of the received 2.05 (Content) response has value `true`, the requester can send a follow-up diff query request including the 'cursor' query parameter, with the same value of the 'cursor' parameter specified in this diff query response. As defined in {{sec-using-cursor-diff-query-response-cursor}}, this would result in the AS transferring the following subset of series items as diff entries, thus resuming from where interrupted in the previous transfer.

### Cursor Specified in the Diff Query Request {#sec-using-cursor-diff-query-response-cursor}

If the update collection associated with the requester is not empty and the diff query request includes the 'cursor' query parameter with value P, the AS proceeds as follows, depending on which of the following two cases hold.

* Case A - The series item X with 'index' having value P and the series item Y with 'index' having value (P + 1) % (MAX_INDEX + 1) are both not found in the update collection associated with the requester. This occurs when the item Y (and possibly further ones after it) has been previously removed from the update collection for that requester (see step 5 at {{sec-trl-endpoint-supporting-diff-queries}}).

   In this case, the AS returns a 2.05 (Content) response. The response MUST have Content-Format "application/ace-trl+cbor" and its payload MUST be a CBOR map formatted as follows.

    * The 'diff_set' parameter MUST be included and specifies the empty CBOR array.

    * The 'cursor' parameter MUST be included and specifies the CBOR simple value `null` (0xf6).

    * The 'more' parameter MUST be included and specifies the CBOR simple value `true` (0xf5).

   With the combination ('cursor', 'more') = (`null`, `true`), the AS is indicating that the update collection is in fact not empty, but that one or more series items have been lost due to their removal. These include the item with 'index' value (P + 1) % (MAX_INDEX + 1), that the requester wished to obtain as the first one following the specified reference point with 'index' value P.

   When receiving this diff query response, the requester SHOULD send a new full query request to the AS. A successful response provides the requester with the full, current pertaining subset of the TRL, as well as with a valid value of the 'cursor' parameter (see {{sec-using-cursor-full-query-response}}) to be possibly used as query parameter in a following diff query request.

* Case B - The series item X with 'index' having value P is found in the update collection associated with the requester; or the series item X is not found and the series item Y with 'index' having value (P + 1) % (MAX_INDEX + 1) is found in the update collection associated with the requester.

   In this case, the AS performs the actions defined in {{ssec-trl-diff-query}}, with the following differences.

   * At step 3, the AS considers the value MAX_DIFF_BATCH (see {{sec-trl-endpoint-supporting-cursor}}), and prepares L = min(SUB_U, MAX_DIFF_BATCH) diff entries, where SUB_U = min(NUM, SUB_SIZE), and SUB_SIZE is the number of series items in the update collection starting from and including the series item added immediately after X. If L is equal to 0 (e.g., because SUB_U is equal to 0), then no diff entries are prepared.

      If SUB_U <= MAX_DIFF_BATCH, the prepared diff entries are the last series items in the update collection associated with the requester, corresponding to the L most recent TRL updates pertaining to the requester.

      If SUB_U > MAX_DIFF_BATCH, the prepared diff entries are the eldest of the last SUB_U series items in the update collection associated with the requester, corresponding to the first L of the SUB_U most recent TRL updates pertaining to the requester.

   * At step 4, the CBOR map to carry in the payload of the 2.05 (Content) response MUST be formatted as follows.

      * The 'diff_set' parameter MUST be present and specifies a CBOR array 'diff_set_value' of L elements. Each element of 'diff_set_value' specifies one of the CBOR arrays 'diff_entry' prepared as a diff entry. Note that L might have value 0, in which case 'diff_set_value' is the empty CBOR array.

      * The 'cursor' parameter MUST be present and MUST specify a CBOR unsigned integer. In particular:

         - If L is equal to 0, i.e., the series item X is the last one in the update collection, then the 'cursor' parameter MUST take the same 'index' value of the last series item in the update collection. Such a value is in fact the current value of 'last_index' for the update collection.

         - If L is different than 0, then the 'cursor' parameter MUST take the 'index' value of the series element of the update collection included as first diff entry in the 'diff_set' CBOR array. That is, the 'cursor' parameter takes the 'index' value of the series item in the update collection corresponding to the most recent TRL update pertaining to the requester and returned in this diff query response.

         Note that the 'cursor' parameter takes the same 'index' value of the last series item in the update collection when SUB_U <= MAX_DIFF_BATCH.

      * The 'more' parameter MUST be present and MUST specify the CBOR simple value `false` (0xf4) if SUB_U <= MAX_DIFF_BATCH, or the CBOR simple value `true` (0xf5) otherwise.

   If the 'more' parameter in the payload of the received 2.05 (Content) response has value `true`, the requester can send a follow-up diff query request including the 'cursor' query parameter, with the same value of the 'cursor' parameter specified in this diff query response. This would result in the AS transferring the following subset of series items as diff entries, thus resuming from where interrupted in the previous transfer.

# Registration at the Authorization Server # {#sec-registration}

During the registration process at the AS, an administrator or a registered device receives the following information as part of the registration response.

* The url-path to the TRL endpoint at the AS.

* The hash function used to compute token hashes. This is specified by identifying an entry in the "Named Information Hash Algorithm" Registry {{Named.Information.Hash.Algorithm}}. The specific means for this is outside the scope of this document.

* A positive integer MAX\_N, if the AS supports diff queries of the TRL (see {{sec-trl-endpoint-supporting-diff-queries}} and {{ssec-trl-diff-query}}).

* A positive integer MAX\_DIFF\_BATCH, if the AS supports diff queries of the TRL as well as the related "Cursor" extension (see {{sec-trl-endpoint-supporting-cursor}} and {{sec-using-cursor}}).

When communicating with one another, the registered devices and the AS have to use a secure communication association and be mutually authenticated (see {{Section 5 of RFC9200}}).

In the same spirit, it MUST be ensured that communications between the AS and an administrator are mutually authenticated, encrypted and integrity protected, as well as protected against message replay.

Before starting its registration process at the AS, an administrator has to establish such a secure communication association with the AS, if they do not share one already. In particular, mutual authentication is REQUIRED during the establishment of the secure association. To this end, the administrator and the AS can rely, e.g., on establishing a TLS or DTLS secure session with mutual authentication {{RFC8446}}{{RFC9147}}, or an OSCORE Security Context {{RFC8613}} by running the authenticated key exchange protocol EDHOC {{RFC9528}}.

When receiving authenticated requests from the administrator for accessing the TRL endpoint, the AS can always check whether the requester is authorized to take such a role, i.e., to access the full TRL.

To this end, the AS may rely on a local access control list or similar, which specifies the authentication credentials of trusted, authorized administrators. In particular, the AS verifies the requester to the TRL endpoint as an authorized administrator, only if the access control list includes the same authentication credential used by the requester when establishing the mutually-authenticated secure communication association with the AS.

Further details about the registration process at the AS are out of scope for this specification. Note that the registration process is also out of the scope of the ACE framework for Authentication and Authorization (see {{Section 5.5 of RFC9200}}).

# Notification of Revoked Access Tokens # {#sec-notification}

Once registered at the AS, the administrator or registered device can send a GET request to the TRL endpoint at the AS. The request can express the wish for a full query (see {{ssec-trl-full-query}}) or a diff query (see {{ssec-trl-diff-query}}) of the TRL. Also, the request can include the CoAP Observe Option set to 0 (register), in order to start an observation of the TRL endpoint as per {{Section 3.1 of RFC7641}}.

In case the request is successfully processed, the AS replies with a response specifying the CoAP response code 2.05 (Content). In particular, if the AS supports diff queries but not the "Cursor" extension (see {{sec-trl-endpoint-supporting-diff-queries}} and {{sec-trl-endpoint-supporting-cursor}}), then the payload of the response is formatted as defined in {{ssec-trl-full-query}} or in {{ssec-trl-diff-query}}, in case the GET request has yielded the execution of a full query or of a diff query of the TRL, respectively. Instead, if the AS supports both diff queries and the related "Cursor" extension, then the payload of the response is formatted as defined in {{sec-using-cursor}}.

In case a requester does not receive a response from the TRL endpoint or it receives an error response from the TRL endpoint, the requester does not make any assumption or draw any conclusion regarding the revocation or expiration of its pertaining access tokens.

When the TRL is updated (see {{ssec-trl-update}}), the AS sends Observe notifications to the observers whose pertaining subset of the TRL has changed. Observe notifications are sent as per {{Section 4.2 of RFC7641}}. If supported by the AS, an observer may configure the behavior according to which the AS sends those Observe notifications. To this end, a possible way relies on the conditional control attribute "c.pmax" defined in {{I-D.ietf-core-conditional-attributes}}, which can be included as a "name=value" query parameter in an Observation Request. This ensures that no more than c.pmax seconds elapse between two consecutive notifications sent to that observer, regardless of whether the TRL has changed or not.

Following a first exchange with the AS, an administrator or a registered device can send additional GET (Observation) requests to the TRL endpoint at any time, analogously to what is defined above. When doing so, the requester towards the TRL endpoint can perform a full query (see {{ssec-trl-full-query}}) or a diff query (see {{ssec-trl-diff-query}}) of the TRL. In the latter case, the requester can additionally rely on the "Cursor" extension (see {{sec-trl-endpoint-query-parameters}} and {{sec-using-cursor-diff-query-response}}).

As specified in {{sec-trl-endpoint-supporting-diff-queries}}, an AS supporting diff queries maintains an update collection of maximum MAX_N series items for each administrator or registered device, hereafter referred to as requester. In particular, if an update collection includes MAX\_N series items, adding a further series item to that update collection results in deleting the oldest series item from that update collection.

From then on, the requester associated with the update collection will not be able to retrieve the deleted series item, when sending a new diff query request to the TRL endpoint. If that series item reflected the revocation of an access token pertaining to the requester, then the requester will not learn about that when receiving the corresponding diff query response from the AS.

Sending a diff query request specifically as an Observation request, and thus relying on Observe notifications, largely reduces the chances for a requester to miss updates occurred to its associated update collection altogether. In turn, this relies on the requester successfully receiving the Observe notification responses from the TRL (see also {{sec-security-considerations-comm-patterns}}).

In order to limit the amount of time during which the requester is unaware of pertaining access tokens that have been revoked but are not expired yet, a requester SHOULD NOT rely solely on diff query requests. In particular, a requester SHOULD also regularly send a full query request to the TRL endpoint according to a related application policy.

## Handling of Access Tokens and Token Hashes # {#sec-handling-token-hashes}

When receiving a response from the TRL endpoint, a registered device MUST expunge every stored access token associated with a token hash specified in the response. In case the registered device is an RS, it MUST NOT delete the stored token hash after having expunged the associated access token.

An RS MUST NOT accept and store an access token, if the corresponding token hash is among the currently stored ones.

An RS MUST store the token hash th1 corresponding to an access token t1 until both the following conditions hold.

* The RS has received and seen t1, irrespective of having accepted and stored it.

* The RS has gained knowledge that t1 has expired. This can be achieved, e.g., through the following means.

   - A response from the TRL endpoint indicating that t1 has expired after its earlier revocation, i.e., the token hash th1 has been removed from the TRL. This can be indicated, for instance, in a response from the TRL endpoint following a diff query of the TRL (see {{ssec-trl-diff-query}}).

   - The value of the 'exp' claim specified in t1 indicates that t1 has expired.

   - The locally determined expiration time for t1 has passed, based on the time at the RS when t1 was first accepted and on the value of its 'exi' claim.

   - The result of token introspection performed on t1 (see {{Section 5.9 of RFC9200}}), if supported by both the RS and the AS.

The RS MUST NOT delete the stored token hashes whose corresponding access tokens do not fulfill both the two conditions above, unless it becomes necessary due to memory limitations. In such a case, the RS MUST delete the earliest stored token hashes first.

Retaining the stored token hashes as specified above limits the impact from a (dishonest) Client whose pertaining access token: i) specifies the 'exi' claim; ii) is uploaded at the RS for the first time after it has been revoked and later expired; and iii) has the sequence number encoded in the 'cti' claim (for CWTs) or in the 'jti' claim (for JWTs) greater than the highest sequence number among the expired access tokens specifying the 'exi' claim for the RS (see {{Section 5.10.3 of RFC9200}}). That is, the RS would not accept such a revoked and expired access token as long as it stores the corresponding token hash.

In order to further limit such a risk, when receiving an access token that specifies the 'exi' claim and for which a corresponding token hash is not stored, the RS can introspect the access token (see {{Section 5.9 of RFC9200}}), if token introspection is implemented by both the RS and the AS.

When, due to the stored and corresponding token hash th2, an access token t2 that includes the 'exi' claim is expunged or is not accepted upon its upload, the RS retrieves the sequence number sn2 encoded in the 'cti' claim (for CWTs) or in the 'jti' claim (for JWTs) (see {{Section 5.10.3 of RFC9200}}). Then, the RS stores sn2 as associated with th2. If expunging or not accepting t2 yields the deletion of th2, then the RS MUST associate sn2 with th2 before continuing with the deletion of th2.

When deleting any token hash, the RS checks whether the token hash is associated with a sequence number sn\_th. In such a case, the RS checks whether sn\_th is greater than the highest sequence number sn\* among the expired access tokens specifying the 'exi' claim for the RS. If that is the case, sn\* MUST take the value of sn\_th.

By virtue of what is defined in {{Section 5.10.3 of RFC9200}}, this ensures that, following the deletion of the token hash associated with an access token specifying the 'exi' claim and uploaded for the first time after it has been revoked and later expired, the RS will not accept the access token at that point in time or in the future.

# ACE Token Revocation List Parameters # {#trl-registry-parameters}

This specification defines a number of parameters that can be transported in the response from the TRL endpoint, when the response payload is a CBOR map. Note that such a response MUST use the Content-Format "application/ace-trl+cbor" defined in {{iana-content-type}} of this specification.

The table below summarizes the parameters. For each of them, it specifies the value to use as CBOR key, i.e., as abbreviation in the key of the map pair for the parameter, instead of the parameter's name as a text string.

| Name              | CBOR Key | CBOR Type                |
| full_set          |  0       | array                    |
| diff_set          |  1       | array                    |
| cursor            |  2       | Null or unsigned integer |
| more              |  3       | True or False            |
{: #table-cbor-trl-params title="CBOR abbreviations for the ACE Token Revocation List parameters" align="center"}

# ACE Token Revocation List Error Identifiers {#error-types}

This specification defines a number of values that the AS can use as error identifiers. These are used in error responses with Content-Format "application/concise-problem-details+cbor", as values of the 'error-id' field within the Custom Problem Detail entry 'ace-trl-error' (see {{sec-error-responses}}).

| Value | Description               |
| 0     | Invalid parameter value   |
| 1     | Invalid set of parameters |
| 2     | Out of bound cursor value |
{: #table-ACE-TRL-Error Identifiers title="ACE Token Revocation List Error Identifiers" align="center"}

# Security Considerations # {#sec-security-considerations}

The protocol defined in this document inherits the security considerations from the ACE framework for Authentication and Authorization {{RFC9200}}, from {{RFC8392}} as to the usage of CWTs, from {{RFC7519}} and {{RFC8725}} as to the usage of JWTs, from {{RFC7641}} as to the usage of CoAP Observe, and from {{RFC6920}} with regard to computing the token hashes. The following considerations also apply.

## Content Retrieval from the TRL

The AS MUST ensure that each registered device can access and retrieve only its pertaining subset of the TRL. To this end, the AS can always perform the required filtering based on the authenticated identity of the registered device, i.e., a (non-public) identifier that the AS can securely relate to the registered device and the secure association that they use to communicate.

The AS MUST ensure that, other than registered devices accessing their own pertaining subset of the TRL, only authorized and authenticated administrators can retrieve the full TRL (see {{sec-registration}}).

## Size of the TRL

If many non-expired access tokens associated with a registered device are revoked, the pertaining subset of the TRL could grow to a size bigger than what the registered device is prepared to handle upon reception of a response from the TRL endpoint, especially if relying on a full query of the TRL (see {{ssec-trl-full-query}}).

This could be exploited by attackers to negatively affect the behavior of a registered device. Therefore, in order to help reduce the size of the TRL, the AS SHOULD refrain from issuing access tokens with an excessively long expiration time.

## Communication Patterns # {#sec-security-considerations-comm-patterns}

The communication about revoked access tokens presented in this specification is expected to especially rely on CoAP Observe notifications sent from the AS to a requester (i.e., an administrator or a registered device). The suppression of those notifications by an external attacker that has access to the network would prevent requesters from ever knowing that their pertaining access tokens have been revoked.

In order to avoid this, a requester SHOULD NOT rely solely on the CoAP Observe notifications. In particular, a requester SHOULD also regularly poll the AS for the most current information about revoked access tokens, by sending GET requests to the TRL endpoint according to a related application policy.

## Request of New Access Tokens

If a Client stores an access token that it still believes to be valid, and it accordingly attempts to access a protected resource at the RS, the Client may receive an unprotected 4.01 (Unauthorized) response from the RS.

This can be due to a number of causes. For example, the access token has been revoked, and the RS has become aware of it and has expunged the access token, but the Client is not aware of it (yet). As another example, the access token is still valid, but an on-path active adversary might have injected a forged 4.01 (Unauthorized) response, or the RS might have deleted the access token from its local storage due to its dedicated storage space being all consumed.

In either case, if the Client believes that the access token is still valid, it SHOULD NOT immediately ask for a new access token to the Authorization Server upon receiving a 4.01 (Unauthorized) response from the RS. Instead, the Client SHOULD send a request to the TRL endpoint at the AS. If the Client gains knowledge that the access token is not valid anymore, the Client expunges the access token and can ask for a new one. Otherwise, the Client can try again to upload the same access token to the RS, or instead to request a new one.

## Vulnerable Time Window at the RS

A Client may attempt to access a protected resource at an RS after the access token allowing such an access has been revoked, but before the RS is aware of the revocation.

In such a case, if the RS is still storing the access token, the Client will be able to access the protected resource, even though it should not. Such an access is a security violation, even if the Client is not attempting to be malicious.

In order to minimize such a risk, if an RS relies solely on polling through individual requests to the TRL endpoint to learn of revoked access tokens, the RS SHOULD implement an adequate trade-off between the polling frequency and the maximum length of the vulnerable time window.

## Two Token Hashes at the RS using JWTs # {#sec-seccons-two-hashes-jwt}

{{sec-token-hash-input-rs-jwt}} defines that an RS using JWTs as access tokens has to compute and store two token hashes associated with the same access token. This is because, when using JWTs, the RS does not know for sure if the AS provided the access token to the Client by means of an AS-to-Client response encoded in CBOR or in JSON.

Taking advantage of that, a dishonest Client can attempt to perform an attack against the RS. That is, the Client can first receive the JWT in an AS-to-Client response encoded in CBOR (JSON). Then, the Client can upload the JWT to the RS in a way that makes the RS believe that the Client instead received the JWT in an AS-to-Client response encoded in JSON (CBOR).

Consequently, the RS considers a HASH_INPUT different from the one considered by the AS and the Client (see {{sec-token-hash-input-c-as}}). Hence, the RS computes a token hash h' different from the token hash h computed by the AS and the Client. It follows that, if the AS revokes the access token and advertises the right token hash h, then the RS will not learn about the access token revocation and thus will not delete the access token.

Fundamentally, this would happen because the HASH_INPUT used to compute the token hash of a JWT depends on whether the AS-to-Client response is encoded in CBOR or in JSON. This makes the RS vulnerable to the attack described above, when JWTs are used as access tokens. Instead, this is not a problem if the access token is a CWT, since the HASH_INPUT used to compute the token hash of a CWT does not depend on whether the AS-to-Client response is encoded in CBOR or in JSON.

While this asymmetry cannot be avoided altogether, the method defined for the AS and the Client in {{sec-token-hash-input-c-as}} deliberately penalizes the case where the RS uses JWTs as access tokens. In such a case, the RS effectively neutralizes the attack described above, by computing and storing two token hashes associated with the same access token (see {{sec-token-hash-input-rs-jwt}}).

Conversely, this design deliberately favors the case where the RS uses CWTs as access tokens, which is a preferable option for resource-constrained RSs as well as the default case in the ACE framework (see {{Section 3 of RFC9200}}). That is, if an RS uses CWTs as access tokens, then the RS is not exposed to the attack described above, and thus it safely computes and stores only one token hash per access token (see {{sec-token-hash-input-rs-cwt}}).

## Additional Security Measures

By accessing the TRL at the AS, registered devices and administrators are able to learn that their pertaining access tokens have been revoked. However, they cannot learn the reason why that happened, including when that reason is the compromise, misbehavior, or decommissioning of a registered device.

In fact, even the AS might not know that a registered device to which a revoked access token pertains has been specifically compromised, misbehaving, or decommissioned. At the same time, it might not be acceptable to only revoke the access tokens pertaining to such a registered device.

Therefore, in order to preserve the security of the system and application, the entity that authoritatively declares a registered device to be compromised, misbehaving, or decommissioned should also promptly trigger the execution of additional revocation processes as deemed appropriate. These include, for instance:

* The de-registration of the registered device from the AS, so that the AS does not issue further access tokens pertaining to that device.

* If applicable, the revocation of the public authentication credential associated with the registered device (e.g., its public key certificate).

The methods by which these processes are triggered and carried out are out of the scope of this document.

# IANA Considerations # {#iana}

This document has the following actions for IANA.

Note to RFC Editor: Please replace all occurrences of "{{&SELF}}" with the RFC number of this specification and delete this paragraph.

## Media Type Registrations {#iana-media-type}

IANA is asked to register the media type "application/ace-trl+cbor" for messages of the protocol defined in this document encoded in CBOR. This registration follows the procedures specified in {{RFC6838}}.

Type name: application

Subtype name: ace-trl+cbor

Required parameters: N/A

Optional parameters: N/A

Encoding considerations: Must be encoded as a CBOR map containing the protocol parameters defined in {{&SELF}}.

Security considerations: See {{sec-security-considerations}} of this document.

Interoperability considerations: N/A

Published specification: {{&SELF}}

Applications that use this media type: The type is used by Authorization Servers, Clients, and Resource Servers that support the notification of revoked access tokens, according to a Token Revocation List maintained by the Authorization Server as specified in {{&SELF}}.

Fragment identifier considerations: N/A

Additional information: N/A

Person & email address to contact for further information: ACE WG mailing list (ace@ietf.org) or IETF Applications and Real-Time Area (art@ietf.org)

Intended usage: COMMON

Restrictions on usage: None

Author/Change controller: IETF

Provisional registration: No

## CoAP Content-Formats Registry {#iana-content-type}

IANA is asked to add the following entry to the "CoAP Content-Formats" registry within the "Constrained RESTful Environments (CoRE) Parameters" registry group.

Content Type: application/ace-trl+cbor

Content Coding: -

ID: TBD

Reference: {{&SELF}}

## Custom Problem Detail Keys Registry {#iana-custom-problem-details}

IANA is asked to register the following entry in the "Custom Problem Detail Keys" registry within the "Constrained RESTful Environments (CoRE) Parameters" registry group.

* Key Value: TBD
* Name: ace-trl-error
* Brief Description: Carry {{&SELF}} problem details in a Concise Problem Details data item.
* Change Controller: IETF
* Reference: {{sec-error-responses}} of {{&SELF}}

## ACE Token Revocation List Parameters Registry ## {#iana-token-revocation-list}

IANA is asked to establish the "ACE Token Revocation List Parameters" IANA registry within the "Authentication and Authorization for Constrained Environments (ACE)" registry group.

As registration policy, the registry uses either "Standards Action with Expert Review", or "Specification Required" per {{Section 4.6 of RFC8126}}, or "Expert Review" per {{Section 4.5 of RFC8126}}. Expert Review guidelines are provided in {{review}}.

All assignments according to "Standards Action with Expert Review" are made on a "Standards Action" basis per {{Section 4.9 of RFC8126}}, with Expert Review additionally required per {{Section 4.5 of RFC8126}}. The procedure for early IANA allocation of Standards Track code points defined in {{RFC7120}} also applies. When such a procedure is used, review and approval by the designated expert are also required, in order for the WG chairs to determine that the conditions for early allocation are met (see step 2 in {{Section 3.1 of RFC7120}}).

The columns of this registry are:

* Name: This field contains a descriptive name that enables easier reference to the item. The name MUST be unique and it is not used in the encoding.

* CBOR Key: This field contains the value used as CBOR map key of the item. The value MUST be unique. The value is an unsigned integer or a negative integer. Different ranges of values use different registration policies {{RFC8126}}. Integer values from -256 to 255 are designated as "Standards Action With Expert Review". Integer values from -65536 to -257 and from 256 to 65535 are designated as "Specification Required". Integer values greater than 65535 are designated as "Expert Review". Integer values less than -65536 are marked as "Private Use".

* CBOR Type: This field contains the allowable CBOR data types for values of this item, or a pointer to the registry that defines its type, when that depends on another item.

* Reference: This field contains a pointer to the public specification for the item.

This registry has been initially populated by the values in {{trl-registry-parameters}}. The "Reference" column for all of these entries refers to this document.

## ACE Token Revocation List Errors {#iana-token-revocation-list-errors}

IANA is asked to establish the "ACE Token Revocation List Errors" IANA registry within the "Authentication and Authorization for Constrained Environments (ACE)" registry group.

As registration policy, the registry uses either "Standards Action with Expert Review", or "Specification Required" per {{Section 4.6 of RFC8126}}, or "Expert Review" per {{Section 4.5 of RFC8126}}. Expert Review guidelines are provided in {{review}}.

All assignments according to "Standards Action with Expert Review" are made on a "Standards Action" basis per {{Section 4.9 of RFC8126}}, with Expert Review additionally required per {{Section 4.5 of RFC8126}}. The procedure for early IANA allocation of Standards Track code points defined in {{RFC7120}} also applies. When such a procedure is used, review and approval by the designated expert are also required, in order for the WG chairs to determine that the conditions for early allocation are met (see step 2 in {{Section 3.1 of RFC7120}}).

The columns of this registry are:

* Value: The field contains the value to be used to identify the error. The value MUST be unique. The value is an unsigned integer or a negative integer. Different ranges of values use different registration policies {{RFC8126}}. Integer values from -256 to 255 are designated as "Standards Action With Expert Review". Integer values from -65536 to -257 and from 256 to 65535 are designated as "Specification Required". Integer values greater than 65535 are designated as "Expert Review". Integer values less than -65536 are marked as "Private Use".

* Description: This field contains a brief description of the error.

* Reference: This field contains a pointer to the public specification defining the error, if one exists.

This registry has been initially populated by the values in {{error-types}}. The "Reference" column for all of these entries refers to this document.

## Expert Review Instructions {#review}

The IANA registries established in this document are defined as "Standards Action with Expert Review", "Specification Required", or "Expert Review", depending on the range of values for which an assignment is requested. This section gives some general guidelines for what the experts should be looking for, but they are being designated as experts for a reason so they should be given substantial latitude.

Expert reviewers should take into consideration the following points:

* Point squatting should be discouraged. Reviewers are encouraged to get sufficient information for registration requests to ensure that the usage is not going to duplicate one that is already registered and that the point is likely to be used in deployments. The zones tagged as private use are intended for testing purposes and closed environments. Code points in other ranges should not be assigned for testing.

* Specifications are required for the "Standards Action With Expert Review" range of point assignment. Specifications should exist for "Specification Required" ranges, but early assignment before a specification is available is considered to be permissible. For the "Expert Review" range of point assignment, specifications are recommended, and are needed if they are expected to be used outside of closed environments in an interoperable way. When specifications are not provided, the description provided needs to have sufficient information to identify what the point is being used for.

* Experts should take into account the expected usage of fields when approving point assignment. The fact that there is a range for Standards Track documents does not mean that a Standards Track document cannot have points assigned outside of that range. The length of the encoded value should be weighed against how many code points of that length are left, the size of device it will be used on, and the number of code points left that encode to that size.

--- back

# On using the Series Transfer Pattern # {#sec-series-pattern}

Performing a diff query of the TRL as specified in {{ssec-trl-diff-query}} is in fact a usage example of the Series Transfer Pattern defined in {{I-D.bormann-t2trg-stp}}.

That is, a diff query enables the transfer of a series of diff entries, with the AS specifying U <= MAX_N diff entries as related to the U most recent TRL updates pertaining to a requester, i.e., a registered device or an administrator.

When responding to a diff query request from a requester (see {{ssec-trl-diff-query}}), 'diff_set' is a subset of the update collection associated with the requester, where each 'diff_entry' record is a series item from that update collection. Note that 'diff_set' specifies the whole current update collection when the value of U is equal to SIZE, i.e., the current number of series items in the update collection.

The value N of the 'diff' query parameter in the GET request allows the requester and the AS to trade the amount of provided information with the latency of the information transfer.

Since the update collection associated with each requester includes up to MAX_N series items, the AS deletes the oldest series item when a new one is generated and added to the end of the update collection, due to a new TRL update pertaining to that requester (see {{sec-trl-endpoint-supporting-diff-queries}}). This addresses the question "When can the server decide to no longer retain older items?" raised in {{Section 3.2 of I-D.bormann-t2trg-stp}}.

Furthermore, performing a diff query of the TRL together with the "Cursor" extension as specified in {{sec-using-cursor}} in fact relies on the "Cursor" pattern of the Series Transfer Pattern (see {{Section 3.3 of I-D.bormann-t2trg-stp}}).


# Local Supportive Parameters of the TRL Endpoint # {#sec-trl-parameteters}

{{table-TRL-endpoint-parameters}} provides an aggregated overview of the local supportive parameters that the AS internally uses at its TRL endpoint, when supporting diff queries (see {{sec-trl-endpoint}}) and the "Cursor" extension (see {{sec-trl-endpoint-supporting-cursor}}).

Except for MAX_N defined in {{sec-trl-endpoint-supporting-diff-queries}}, all the other parameters are defined in {{sec-trl-endpoint-supporting-cursor}} and are used only if the AS supports the "Cursor" extension.

For each parameter, the columns of the table specify the following information. Both a registered device and an administrator are referred to as "requester".

* Name: parameter name. A name with letters in uppercase denotes a parameter whose value does not change after its initialization.

* Single instance: "Y", if there is a single parameter instance associated with the TRL; or "N", if there is one parameter instance per update collection (i.e., per requester).

* Description: short parameter description.

* Values: the unsigned integer values that the parameter can assume, where LB and UB denote the inclusive lower bound and upper bound, respectively, and "^" is the exponentiation operator.

| Name           | Single <br> instance | Description                                                                      | Values                                                                   |
| MAX_N          | Y                    | Max number of series items in the update collection of each requester            | LB = 1 <br><br> If supporting <br> "Cursor", then <br> UB = MAX_INDEX+1 |
| MAX_DIFF_BATCH | N                    | Max number of diff entries included in a diff query response when using "Cursor" | LB = 1 <br><br> UB = MAX_N                                              |
| MAX_INDEX      | Y                    | Max value of each instance of the 'index' parameter                              | LB = MAX_N-1 <br><br> UB = (2^64)-1                                     |
| index          | N                    | Value associated with a series item of an update collection                      | LB = 0 <br><br> UB = MAX_INDEX                                          |
| last_index     | N                    | The 'index' value of the most recently added series item in an update collection | LB = 0 <br><br> UB = MAX_INDEX                                          |
{: #table-TRL-endpoint-parameters title="Local Supportive Parameters of the TRL Endpoint" align="center"}

# Interaction Examples # {#sec-RS-examples}

This section provides examples of interactions between an RS as a registered device and an AS. In the examples, all the access tokens issued by the AS are intended to be consumed by the considered RS.

The AS supports both full queries and diff queries of the TRL, as defined in {{ssec-trl-full-query}} and {{ssec-trl-diff-query}}, respectively.

Registration is assumed to be done by the RS sending a POST request with an unspecified payload to the AS, which replies with a 2.01 (Created) response. The payload of the registration response is assumed to be a CBOR map, which in turn is assumed to include the following entries:

* a 'trl_path' parameter, specifying the path of the TRL endpoint;

* a 'trl_hash' parameter, specifying the "Hash Name String" of the hash function used to compute token hashes as defined in {{sec-token-name}};

* a 'max_n' parameter, specifying the value of MAX_N, i.e., the maximum number of series items that the AS retains in the update collection associated with a registered device (see {{ssec-trl-diff-query}});

* possible further parameters related to the registration process.

Furthermore, 'h(x)' refers to the hash function used to compute the token hashes, as defined in {{sec-token-name}} of this specification and according to {{RFC6920}}. Assuming the usage of CWTs transported in AS-to-Client responses encoded in CBOR (see {{sec-token-hash-input-c-as-cbor}}), 'bstr.h(t1)' and 'bstr.h(t2)' denote the CBOR byte strings with value the token hashes of the access tokens t1 and t2, respectively.

## Full Query with Observe # {#sec-RS-example-1}

{{fig-RS-AS}} shows an interaction example considering a CoAP observation and a full query of the TRL.

In this example, the AS does not support the "Cursor" extension. Hence, the 'cursor' parameter is not included in the payload of the responses to a full query request.

~~~~~~~~~~~ aasvg
RS                                                  AS
|                                                    |
|  Registration: POST                                |
+--------------------------------------------------->|
|                                                    |
|<---------------------------------------------------+
|                   2.01 Created                     |
|                     Payload: {                     |
|                       / ... /                      |
|                       "trl_path" : "/revoke/trl",  |
|                       "trl_hash" : "sha-256",      |
|                          "max_n" : 10              |
|                     }                              |
|                                                    |
|  GET coap://as.example.com/revoke/trl/             |
|    Observe: 0                                      |
+--------------------------------------------------->|
|                                                    |
|<---------------------------------------------------+
|      2.05 Content                                  |
|        Observe: 42                                 |
|        Content-Format: application/ace-trl+cbor    |
|        Payload: {                                  |
|          e'full_set' : []                          |
|        }                                           |
|                                                    |
|                        ...                         |
|                                                    |
|          (Access tokens t1 and t2 issued           |
|          and successfully submitted to RS)         |
|                                                    |
|                        ...                         |
|                                                    |
|             (Access token t1 is revoked)           |
|                                                    |
|<---------------------------------------------------+
|      2.05 Content                                  |
|        Observe: 53                                 |
|        Content-Format: application/ace-trl+cbor    |
|        Payload: {                                  |
|          e'full_set' : [bstr.h(t1)]                |
|        }                                           |
|                                                    |
|                        ...                         |
|                                                    |
|             (Access token t2 is revoked)           |
|                                                    |
|<---------------------------------------------------+
|      2.05 Content                                  |
|        Observe: 64                                 |
|        Content-Format: application/ace-trl+cbor    |
|        Payload: {                                  |
|          e'full_set' : [bstr.h(t1), bstr.h(t2)]    |
|        }                                           |
|                                                    |
|                        ...                         |
|                                                    |
|             (Access token t1 expires)              |
|                                                    |
|<---------------------------------------------------+
|      2.05 Content                                  |
|        Observe: 75                                 |
|        Content-Format: application/ace-trl+cbor    |
|        Payload: {                                  |
|          e'full_set' : [bstr.h(t2)]                |
|        }                                           |
|                                                    |
|                        ...                         |
|                                                    |
|             (Access token t2 expires)              |
|                                                    |
|<---------------------------------------------------+
|      2.05 Content                                  |
|        Observe: 86                                 |
|        Content-Format: application/ace-trl+cbor    |
|        Payload: {                                  |
|          e'full_set' : []                          |
|        }                                           |
|                                                    |
~~~~~~~~~~~
{: #fig-RS-AS title="Interaction for full query with Observe" artwork-align="center"}

## Diff Query with Observe # {#sec-RS-example-2}

{{fig-RS-AS-2}} shows an interaction example considering a CoAP observation and a diff query of the TRL.

The RS indicates N = 3 as value of the 'diff' query parameter, i.e., as the maximum number of diff entries to be specified in a response from the AS.

In this example, the AS does not support the "Cursor" extension. Hence, the 'cursor' parameter and the 'more' parameter are not included in the payload of the responses to a diff query request.

~~~~~~~~~~~ aasvg
RS                                                  AS
|                                                    |
|  Registration: POST                                |
+--------------------------------------------------->|
|                                                    |
|<---------------------------------------------------+
|                   2.01 Created                     |
|                     Payload: {                     |
|                       / ... /                      |
|                       "trl_path" : "/revoke/trl",  |
|                       "trl_hash" : "sha-256",      |
|                          "max_n" : 10              |
|                     }                              |
|                                                    |
|  GET coap://as.example.com/revoke/trl?diff=3       |
|    Observe: 0                                      |
+--------------------------------------------------->|
|                                                    |
|<---------------------------------------------------+
|      2.05 Content                                  |
|        Observe: 42                                 |
|        Content-Format: application/ace-trl+cbor    |
|        Payload: {                                  |
|          e'diff_set' : []                          |
|        }                                           |
|                                                    |
|                        ...                         |
|                                                    |
|          (Access tokens t1 and t2 issued           |
|          and successfully submitted to RS)         |
|                                                    |
|                        ...                         |
|                                                    |
|            (Access token t1 is revoked)            |
|                                                    |
|<---------------------------------------------------+
|      2.05 Content                                  |
|        Observe: 53                                 |
|        Content-Format: application/ace-trl+cbor    |
|        Payload: {                                  |
|          e'diff_set' : [                           |
|                         [ [], [bstr.h(t1)] ]       |
|                        ]                           |
|        }                                           |
|                                                    |
|                        ...                         |
|                                                    |
|            (Access token t2 is revoked)            |
|                                                    |
|<---------------------------------------------------+
|      2.05 Content                                  |
|        Observe: 64                                 |
|        Content-Format: application/ace-trl+cbor    |
|        Payload: {                                  |
|          e'diff_set' : [                           |
|                         [ [], [bstr.h(t2)] ],      |
|                         [ [], [bstr.h(t1)] ]       |
|                        ]                           |
|        }                                           |
|                                                    |
|                        ...                         |
|                                                    |
|              (Access token t1 expires)             |
|                                                    |
|<---------------------------------------------------+
|      2.05 Content                                  |
|        Observe: 75                                 |
|        Content-Format: application/ace-trl+cbor    |
|        Payload: {                                  |
|          e'diff_set' : [                           |
|                         [ [bstr.h(t1)], [] ],      |
|                         [ [], [bstr.h(t2)] ],      |
|                         [ [], [bstr.h(t1)] ]       |
|                        ]                           |
|        }                                           |
|                                                    |
|                        ...                         |
|                                                    |
|              (Access token t2 expires)             |
|                                                    |
|<---------------------------------------------------+
|      2.05 Content                                  |
|        Observe: 86                                 |
|        Content-Format: application/ace-trl+cbor    |
|        Payload: {                                  |
|          e'diff_set' : [                           |
|                         [ [bstr.h(t2)], [] ],      |
|                         [ [bstr.h(t1)], [] ],      |
|                         [ [], [bstr.h(t2)] ]       |
|                        ]                           |
|        }                                           |
|                                                    |
~~~~~~~~~~~
{: #fig-RS-AS-2 title="Interaction for diff query with Observe" artwork-align="center"}

## Full Query with Observe plus Diff Query # {#sec-RS-example-3}

{{fig-RS-AS-3}} shows an interaction example considering a CoAP observation and a full query of the TRL.

The example also considers one of the notifications from the AS to get lost in transmission, and thus not reaching the RS.

When this happens, and after a waiting time defined by the application has elapsed, the RS sends a GET request with no Observe Option to the AS, to perform a diff query of the TRL. The RS indicates N = 8 as value of the 'diff' query parameter, i.e., as the maximum number of diff entries to be specified in a response from the AS.

In this example, the AS does not support the "Cursor" extension. Hence, the 'cursor' parameter is not included in the payload of the responses to a full query request. Also, the 'cursor' parameter and the 'more' parameter are not included in the payload of the responses to a diff query request.

~~~~~~~~~~~ aasvg
RS                                                  AS
|                                                    |
|  Registration: POST                                |
+--------------------------------------------------->|
|                                                    |
|<---------------------------------------------------+
|                   2.01 Created                     |
|                     Payload: {                     |
|                       / ... /                      |
|                       "trl_path" : "/revoke/trl",  |
|                       "trl_hash" : "sha-256",      |
|                          "max_n" : 10              |
|                     }                              |
|                                                    |
|  GET coap://as.example.com/revoke/trl/             |
|    Observe: 0                                      |
+--------------------------------------------------->|
|                                                    |
|<---------------------------------------------------+
|      2.05 Content                                  |
|        Observe: 42                                 |
|        Content-Format: application/ace-trl+cbor    |
|        Payload: {                                  |
|          e'full_set' : []                          |
|        }                                           |
|                                                    |
|                        ...                         |
|                                                    |
|          (Access tokens t1 and t2 issued           |
|          and successfully submitted to RS)         |
|                                                    |
|                        ...                         |
|                                                    |
|            (Access token t1 is revoked)            |
|                                                    |
|<---------------------------------------------------+
|      2.05 Content                                  |
|        Observe: 53                                 |
|        Content-Format: application/ace-trl+cbor    |
|        Payload: {                                  |
|          e'full_set' : [bstr.h(t1)]                |
|        }                                           |
|                                                    |
|                        ...                         |
|                                                    |
|            (Access token t2 is revoked)            |
|                                                    |
|<---------------------------------------------------+
|      2.05 Content                                  |
|        Observe: 64                                 |
|        Content-Format: application/ace-trl+cbor    |
|        Payload: {                                  |
|          e'full_set' : [bstr.h(t1), bstr.h(t2)]    |
|        }                                           |
|                                                    |
|                        ...                         |
|                                                    |
|             (Access token t1 expires)              |
|                                                    |
|<---------------------------------------------------+
|      2.05 Content                                  |
|        Observe: 75                                 |
|        Content-Format: application/ace-trl+cbor    |
|        Payload: {                                  |
|          e'full_set' : [bstr.h(t2)]                |
|        }                                           |
|                                                    |
|                        ...                         |
|                                                    |
|             (Access token t2 expires)              |
|                                                    |
|  Lost X <------------------------------------------+
|      2.05 Content                                  |
|        Observe: 86                                 |
|        Content-Format: application/ace-trl+cbor    |
|        Payload: {                                  |
|          e'full_set' : []                          |
|        }                                           |
|                                                    |
|                        ...                         |
|                                                    |
|           (Enough time has passed since            |
|         the latest received notification)          |
|                                                    |
|                                                    |
|  GET coap://as.example.com/revoke/trl?diff=8       |
+--------------------------------------------------->|
|                                                    |
|<---------------------------------------------------+
|      2.05 Content                                  |
|        Content-Format: application/ace-trl+cbor    |
|        Payload: {                                  |
|          e'diff_set' : [                           |
|                         [ [bstr.h(t2)], [] ],      |
|                         [ [bstr.h(t1)], [] ],      |
|                         [ [], [bstr.h(t2)] ],      |
|                         [ [], [bstr.h(t1)] ]       |
|                        ]                           |
|        }                                           |
|                                                    |
~~~~~~~~~~~
{: #fig-RS-AS-3 title="Interaction for full query with Observe plus diff query" artwork-align="center"}

## Diff Query with Observe and \"Cursor\" # {#sec-RS-example-2-3}

In this example, the AS supports the "Cursor" extension. Hence, the CBOR map conveyed as payload of the registration response additionally includes a "max_diff_batch" parameter. This specifies the value of MAX_DIFF_BATCH, i.e., the maximum number of diff entries that can be included in a response to a diff query request from this RS.

{{fig-RS-AS-4}} shows an interaction example considering a CoAP observation and a diff query of the TRL.

The RS specifies the query parameter 'diff' with value 3, i.e., the maximum number of diff entries to be specified in a response from the AS.

After the RS has not received a notification from the AS for a waiting time defined by the application, the RS sends a GET request with no Observe Option to the AS, to perform a diff query of the TRL.

This is followed up by a further diff query request that specifies the query parameter 'cursor'. Note that the payload of the corresponding response differs from the payload of the response to the previous diff query request.

~~~~~~~~~~~ aasvg
RS                                                      AS
|                                                        |
|  Registration: POST                                    |
+------------------------------------------------------->|
|                                                        |
|<-------------------------------------------------------+
|                  2.01 Created                          |
|                    Payload: {                          |
|                           / ... /                      |
|                           "trl_path" : "/revoke/trl",  |
|                           "trl_hash" : "sha-256",      |
|                              "max_n" : 10,             |
|                      "max_diff_batch": 5               |
|                    }                                   |
|                                                        |
|  GET coap://as.example.com/revoke/trl?diff=3           |
|    Observe: 0                                          |
+------------------------------------------------------->|
|                                                        |
|<-------------------------------------------------------+
|          2.05 Content                                  |
|            Observe: 42                                 |
|            Content-Format: application/ace-trl+cbor    |
|            Payload: {                                  |
|              e'diff_set' : [],                         |
|                e'cursor' : null,                       |
|                  e'more' : false                       |
|            }                                           |
|                                                        |
|                          ...                           |
|                                                        |
|            (Access tokens t1 and t2 issued             |
|            and successfully submitted to RS)           |
|                                                        |
|                          ...                           |
|                                                        |
|              (Access token t1 is revoked)              |
|                                                        |
|<-------------------------------------------------------+
|          2.05 Content                                  |
|            Observe: 53                                 |
|            Content-Format: application/ace-trl+cbor    |
|            Payload: {                                  |
|              e'diff_set' : [                           |
|                             [ [], [bstr.h(t1)] ]       |
|                            ],                          |
|                e'cursor' : 0,                          |
|                  e'more' : false                       |
|            }                                           |
|                                                        |
|                          ...                           |
|                                                        |
|              (Access token t2 is revoked)              |
|                                                        |
|<-------------------------------------------------------+
|          2.05 Content                                  |
|            Observe: 64                                 |
|            Content-Format: application/ace-trl+cbor    |
|            Payload: {                                  |
|              e'diff_set' : [                           |
|                             [ [], [bstr.h(t2)] ],      |
|                             [ [], [bstr.h(t1)] ]       |
|                            ],                          |
|                e'cursor' : 1,                          |
|                  e'more' : false                       |
|            }                                           |
|                                                        |
|                          ...                           |
|                                                        |
|              (Access token t1 expires)                 |
|                                                        |
|<-------------------------------------------------------+
|          2.05 Content                                  |
|            Observe: 75                                 |
|            Content-Format: application/ace-trl+cbor    |
|            Payload: {                                  |
|              e'diff_set' : [                           |
|                             [ [bstr.h(t1)], [] ],      |
|                             [ [], [bstr.h(t2)] ],      |
|                             [ [], [bstr.h(t1)] ]       |
|                            ],                          |
|                e'cursor' : 2,                          |
|                  e'more' : false                       |
|            }                                           |
|                                                        |
|                          ...                           |
|                                                        |
|              (Access token t2 expires)                 |
|                                                        |
|<-------------------------------------------------------+
|          2.05 Content                                  |
|            Observe: 86                                 |
|            Content-Format: application/ace-trl+cbor    |
|            Payload: {                                  |
|              e'diff_set' : [                           |
|                             [ [bstr.h(t2)], [] ],      |
|                             [ [bstr.h(t1)], [] ],      |
|                             [ [], [bstr.h(t2)] ]       |
|                            ],                          |
|                e'cursor' : 3,                          |
|                  e'more' : false                       |
|            }                                           |
|                                                        |
|                          ...                           |
|                                                        |
|            (Enough time has passed since               |
|             the latest received notification)          |
|                                                        |
|                                                        |
|  GET coap://as.example.com/revoke/trl?diff=3           |
+------------------------------------------------------->|
|                                                        |
|<-------------------------------------------------------+
|          2.05 Content                                  |
|            Content-Format: application/ace-trl+cbor    |
|            Payload: {                                  |
|              e'diff_set' : [                           |
|                             [ [bstr.h(t2)], [] ],      |
|                             [ [bstr.h(t1)], [] ],      |
|                             [ [], [bstr.h(t2)] ]       |
|                            ],                          |
|                e'cursor' : 3,                          |
|                  e'more' : false                       |
|            }                                           |
|                                                        |
|  GET coap://as.example.com/revoke/trl?diff=3&cursor=3  |
+------------------------------------------------------->|
|                                                        |
|<-------------------------------------------------------+
|          2.05 Content                                  |
|            Content-Format: application/ace-trl+cbor    |
|            Payload: {                                  |
|              e'diff_set' : [],                         |
|                e'cursor' : 3,                          |
|                  e'more' : false                       |
|            }                                           |
|                                                        |
~~~~~~~~~~~
{: #fig-RS-AS-4 title="Interaction for diff query with Observe and \"Cursor\"" artwork-align="center"}

## Full Query with Observe plus Diff Query with \"Cursor\" # {#sec-RS-example-5}

In this example, the AS supports the "Cursor" extension. Hence, the CBOR map conveyed as payload of the registration response additionally includes a "max_diff_batch" parameter. This specifies the value of MAX_DIFF_BATCH, i.e., the maximum number of diff entries that can be included in a response to a diff query request from this RS.

{{fig-RS-AS-5}} shows an interaction example considering a CoAP observation and a full query of the TRL.

The example also considers some of the notifications from the AS to get lost in transmission, and thus not reaching the RS.

When this happens, and after a waiting time defined by the application has elapsed, the RS sends a GET request with no Observe Option to the AS, to perform a diff query of the TRL. In particular, the RS specifies:

* The query parameter 'diff' with value 8, i.e., the maximum number of diff entries to be specified in a response from the AS.

* The query parameter 'cursor' with value 2, thus requesting from the update collection the series items following the one with 'index' value equal to 2 (i.e., following the last series item that the RS successfully received in an earlier notification response).

The response from the AS conveys a first batch of MAX_DIFF_BATCH = 5 series items from the update collection corresponding to the RS. The AS indicates that further series items are actually available in the update collection, by setting the 'more' parameter of the response to `true`. Also, the 'cursor' parameter of the response is set to 7, i.e., to the 'index' value of the most recent series item included in the response.

After that, the RS follows up with a further diff query request specifying the query parameter 'cursor' with value 7, in order to retrieve the next and last batch of series items from the update collection.

~~~~~~~~~~~ aasvg
RS                                                             AS
|                                                               |
|  Registration: POST                                           |
+-------------------------------------------------------------->|
|                                                               |
|<--------------------------------------------------------------+
|                         2.01 Created                          |
|                           Payload: {                          |
|                                  / ... /                      |
|                                  "trl_path" : "/revoke/trl",  |
|                                  "trl_hash" : "sha-256",      |
|                                     "max_n" : 10,             |
|                             "max_diff_batch": 5               |
|                           }                                   |
|                                                               |
|  GET coap://as.example.com/revoke/trl/                        |
|    Observe: 0                                                 |
+-------------------------------------------------------------->|
|                                                               |
|<--------------------------------------------------------------+
|                 2.05 Content                                  |
|                   Observe: 42                                 |
|                   Content-Format: application/ace-trl+cbor    |
|                   Payload: {                                  |
|                     e'full_set' : [],                         |
|                       e'cursor' : null                        |
|                   }                                           |
|                                                               |
|                              ...                              |
|                                                               |
|               (Access tokens t1, t2, t3 issued                |
|                and successfully submitted to RS)              |
|                                                               |
|                              ...                              |
|                                                               |
|               (Access tokens t4, t5, t6 issued                |
|               and successfully submitted to RS)               |
|                                                               |
|                              ...                              |
|                                                               |
|                  (Access token t1 is revoked)                 |
|                                                               |
|<--------------------------------------------------------------+
|                 2.05 Content                                  |
|                   Observe: 53                                 |
|                   Content-Format: application/ace-trl+cbor    |
|                   Payload: {                                  |
|                     e'full_set' : [bstr.h(t1)],               |
|                       e'cursor' : 0                           |
|                   }                                           |
|                                                               |
|                              ...                              |
|                                                               |
|                  (Access token t2 is revoked)                 |
|                                                               |
|<--------------------------------------------------------------+
|                 2.05 Content                                  |
|                   Observe: 64                                 |
|                   Content-Format: application/ace-trl+cbor    |
|                   Payload: {                                  |
|                     e'full_set' : [bstr.h(t1), bstr.h(t2)],   |
|                       e'cursor' : 1                           |
|                   }                                           |
|                                                               |
|                              ...                              |
|                                                               |
|                   (Access token t1 expires)                   |
|                                                               |
|<--------------------------------------------------------------+
|                 2.05 Content                                  |
|                   Observe: 75                                 |
|                   Content-Format: application/ace-trl+cbor    |
|                   Payload: {                                  |
|                     e'full_set' : [bstr.h(t2)],               |
|                     e'cursor'   : 2                           |
|                   }                                           |
|                                                               |
|                              ...                              |
|                                                               |
|                   (Access token t2 expires)                   |
|                                                               |
|  Lost X <-----------------------------------------------------+
|                 2.05 Content                                  |
|                   Observe: 86                                 |
|                   Content-Format: application/ace-trl+cbor    |
|                   Payload: {                                  |
|                     e'full_set' : [],                         |
|                       e'cursor' : 3                           |
|                   }                                           |
|                                                               |
|                              ...                              |
|                                                               |
|                  (Access token t3 is revoked)                 |
|                                                               |
|  Lost X <-----------------------------------------------------+
|                 2.05 Content                                  |
|                   Observe: 88                                 |
|                   Content-Format: application/ace-trl+cbor    |
|                   Payload: {                                  |
|                     e'full_set' : [bstr.h(t3)],               |
|                       e'cursor' : 4                           |
|                   }                                           |
|                                                               |
|                              ...                              |
|                                                               |
|                  (Access token t4 is revoked)                 |
|                                                               |
|  Lost X <-----------------------------------------------------+
|                 2.05 Content                                  |
|                   Observe: 89                                 |
|                   Content-Format: application/ace-trl+cbor    |
|                   Payload: {                                  |
|                     e'full_set' : [bstr.h(t3), bstr.h(t4)],   |
|                       e'cursor' : 5                           |
|                   }                                           |
|                                                               |
|                              ...                              |
|                                                               |
|                    (Access token t3 expires)                  |
|                                                               |
|  Lost X <-----------------------------------------------------+
|                 2.05 Content                                  |
|                   Observe: 90                                 |
|                   Content-Format: application/ace-trl+cbor    |
|                   Payload: {                                  |
|                     e'full_set' : [bstr.h(t4)],               |
|                       e'cursor' : 6                           |
|                   }                                           |
|                                                               |
|                              ...                              |
|                                                               |
|                    (Access token t4 expires)                  |
|                                                               |
|  Lost X <-----------------------------------------------------+
|                 2.05 Content                                  |
|                   Observe: 91                                 |
|                   Content-Format: application/ace-trl+cbor    |
|                   Payload: {                                  |
|                     e'full_set' : [],                         |
|                       e'cursor' : 7                           |
|                   }                                           |
|                                                               |
|                              ...                              |
|                                                               |
|              (Access tokens t5 and t6 are revoked)            |
|                                                               |
|  Lost X <-----------------------------------------------------+
|                 2.05 Content                                  |
|                   Observe: 92                                 |
|                   Content-Format: application/ace-trl+cbor    |
|                   Payload: {                                  |
|                     e'full_set' : [bstr.h(t5), bstr.h(t6)],   |
|                       e'cursor' : 8                           |
|                   }                                           |
|                                                               |
|                              ...                              |
|                                                               |
|                    (Access token t5 expires)                  |
|                                                               |
|  Lost X <-----------------------------------------------------+
|                 2.05 Content                                  |
|                   Observe: 93                                 |
|                   Content-Format: application/ace-trl+cbor    |
|                   Payload: {                                  |
|                     e'full_set' : [bstr.h(t6)],               |
|                       e'cursor' : 9                           |
|                   }                                           |
|                                                               |
|                              ...                              |
|                                                               |
|                    (Access token t6 expires)                  |
|                                                               |
|  Lost X <-----------------------------------------------------+
|                 2.05 Content                                  |
|                   Observe: 94                                 |
|                   Content-Format: application/ace-trl+cbor    |
|                   Payload: {                                  |
|                     e'full_set' : [],                         |
|                       e'cursor' : 10                          |
|                   }                                           |
|                                                               |
|                              ...                              |
|                                                               |
|                (Enough time has passed since                  |
|                 the latest received notification)             |
|                                                               |
|                                                               |
|  GET coap://as.example.com/revoke/trl?diff=8&cursor=2         |
+-------------------------------------------------------------->|
|                                                               |
|<--------------------------------------------------------------+
|                 2.05 Content                                  |
|                   Content-Format: application/ace-trl+cbor    |
|                   Payload: {                                  |
|                     e'diff_set' : [                           |
|                                    [ [bstr.h(t4)], [] ],      |
|                                    [ [bstr.h(t3)], [] ],      |
|                                    [ [], [bstr.h(t4)] ],      |
|                                    [ [], [bstr.h(t3)] ],      |
|                                    [ [bstr.h(t2)], [] ]       |
|                                   ],                          |
|                       e'cursor' : 7,                          |
|                         e'more' : true                        |
|                   }                                           |
|                                                               |
|  GET coap://as.example.com/revoke/trl?diff=8&cursor=7         |
+-------------------------------------------------------------->|
|                                                               |
|<--------------------------------------------------------------+
|        2.05 Content                                           |
|          Content-Format: application/ace-trl+cbor             |
|          Payload: {                                           |
|            e'diff_set' : [                                    |
|                           [ [bstr.h(t6)], [] ],               |
|                           [ [bstr.h(t5)], [] ],               |
|                           [ [], [bstr.h(t5), bstr.h(t6)] ]    |
|                          ],                                   |
|              e'cursor' : 10,                                  |
|                e'more' : false                                |
|          }                                                    |
|                                                               |
~~~~~~~~~~~
{: #fig-RS-AS-5 title="Interaction for full query with Observe plus diff query with \"Cursor\"" artwork-align="center"}


# CDDL Model # {#sec-cddl-model}
{:removeinrfc}

~~~~~~~~~~~~~~~~~~~~ CDDL
full_set = 0
diff_set = 1
cursor = 2
more = 3

ace-trl-error = 1
~~~~~~~~~~~~~~~~~~~~
{: #fig-cddl-model title="CDDL model" artwork-align="left"}


# Document Updates # {#sec-document-updates}
{:removeinrfc}

## Version -07 to -08 ## {#sec-07-08}

* Added definition of pertaining token hash.

* Added definition of pertaining TRL update.

* Rephrased example of token uploading to be more future ready.

* Consistent use of "TRL update" throughout the document.

* Editorial improvements.

## Version -06 to -07 ## {#sec-06-07}

* RFC 9290 is used instead of the custom format for error responses.

* Avoided quotation marks when using CBOR simple values.

* CBOR diagnostic notation uses placeholders from a CDDL model.

* Early mentioning that there is a single MAX_N value.

* Added more details on the authorization of administrators.

* Added recommendations for avoiding lost TRL updates from going unnoticed.

* If diff queries are supported, the AS MUST provide MAX_N at registration.

* If the "Cursor" extension is supported, the AS MUST provide MAX_DIFF_BATCH at registration.

* Clarified that how the token revocation specifically happens is out of scope.

* Clearer, upfront distinction between using CoAP Observe or not.

* Revised and extended method for computing the token hashes.

* Clearer presentation of invalid requests to the TRL endpoint.

* Clearer expected relation between MAX_INDEX and MAX_N values.

* Clarified meaning of registered parameters.

* Generalized security considerations on vulnerable time window at the RS.

* Added security considerations on additional security measures.

* Fixes and improvements in the IANA considerations.

* Used AASVG in diagrams.

* Used actual tables instead of figures.

* Fixed notation in the examples.

* Clarifications and editorial improvements.

## Version -05 to -06 ## {#sec-05-06}

* Clarified instructions for Expert Review in the IANA considerations.

## Version -04 to -05 ## {#sec-04-05}

* Explicit focus on CoAP in the abstract and introduction.

* Removed terminology aliasing ("TRL endpoint" vs. "TRL resource").

* Use "requester" instead of "caller".

* Use "subset" instead of "portion".

* Revised presentation of how token hashes are computed.

* Improved error handling.

* Revised examples.

* More precise security considerations.

* Clarifications and editorial improvements.

* Updated author list.

## Version -03 to -04 ## {#sec-03-04}

* Improved presentation of pre- and post-registration operations.

* Removed moot processing cases with the "Cursor" extension.

* Positive integers as CBOR abbreviations for all parameters.

* Renamed N_MAX as MAX_N.

* Access tokens are not necessarily uploaded through /authz-info.

* The use of the "c.pmax" conditional attribute is just an example.

* Revised handling of token hashes at the RS.

* Extended and improved security considerations.

* Fixed details in IANA considerations.

* New appendix overviewing parameters of the TRL endpoint.

* Examples of message exchange moved to an appendix.

* Added examples of message exchange with the "Cursor" extension.

* Clarifications and editorial improvements.

## Version -02 to -03 ## {#sec-02-03}

* Definition of MAX_INDEX for the "Cursor" extension.

* Handling wrap-around of 'index' when using the "Cursor" extension.

* Error handling for the case where 'cursor' > MAX_INDEX.

* Improved error handling in case 'index' is out-of-bound.

* Clarified parameter semantics, message content and examples.

* Editorial improvements.

## Version -01 to -02 ## {#sec-01-02}

* Earlier mentioning of error cases.

* Clearer distinction between maintaining the history of TRL updates and preparing the response to a diff query.

* Defined the use of "cursor" in the document body, as an extension of diff queries.

* Both success and error responses have a CBOR map as payload.

* Corner cases of message processing explained more explicitly.

* Clarifications and editorial improvements.

## Version -00 to -01 ## {#sec-00-01}

* Added actions to perform upon receiving responses from the TRL endpoint.

* Fixed off-by-one error when using the "Cursor" pattern.

* Improved error handling, with registered error codes.

* Section restructuring (full- and diff-query as self-standing sections).

* Renamed identifiers and CBOR parameters.

* Clarifications and editorial improvements.

# Acknowledgments # {#acknowldegment}
{: numbered="no"}

{{{Ludwig Seitz}}} contributed as a co-author of initial versions of this document.

The authors sincerely thank {{{Christian Amsüss}}}, {{{Carsten Bormann}}}, {{{Dhruv Dhody}}}, {{{Rikard Höglund}}}, {{{Benjamin Kaduk}}}, {{{David Navarro}}}, {{{Joerg Ott}}}, {{{Marco Rasori}}}, {{{Michael Richardson}}}, {{{Kyle Rose}}}, {{{Jim Schaad}}}, {{{Göran Selander}}}, {{{Travis Spencer}}}, {{{Orie Steele}}}, {{{Dale Worley}}}, and {{{Paul Wouters}}} for their comments and feedback.

The work on this document has been partly supported by the Sweden's Innovation Agency VINNOVA and the Celtic-Next projects CRITISEC and CYPRESS; and by the H2020 project SIFIS-Home (Grant agreement 952652).
