---
v: 3

title: Notification of Revoked Access Tokens in the Authentication and Authorization for Constrained Environments (ACE) Framework
abbrev: Notification of Revoked Tokens in ACE
docname: draft-ietf-ace-revoked-token-notification-latest


# stand_alone: true

ipr: trust200902
area: Internet
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
        ins: L. Seitz
        name: Ludwig Seitz
        org: Combitech
        street: Djaeknegatan 31
        city: Malmoe
        code: SE-21135
        country: Sweden
        email: ludwig.seitz@combitech.com
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
  I-D.ietf-ace-oauth-authz:
  I-D.ietf-core-conditional-attributes:
  RFC2119:
  RFC6749:
  RFC6838:
  RFC6920:
  RFC7252:
  RFC7641:
  RFC8259:
  RFC7519:
  RFC8126:
  RFC8174:
  RFC8392:
  RFC8610:
  RFC8949:
  Named.Information.Hash.Algorithm:
    author:
      org: IANA
    date: false
    title: Named Information Hash Algorithm
    target: https://www.iana.org/assignments/named-information/named-information.xhtml

informative:
  RFC7009:
  I-D.bormann-t2trg-stp:

entity:
  SELF: "[RFC-XXXX]"

--- abstract

This document specifies a method of the Authentication and Authorization for Constrained  Environments (ACE) framework, which allows an Authorization Server to notify Clients and Resource Servers (i.e., registered devices) about revoked Access Tokens. The method allows Clients and Resource Servers to access a Token Revocation List on the Authorization Server, with the possible additional use of resource observation for the Constrained Application Protocol (CoAP). Resulting (unsolicited) notifications of revoked Access Tokens complement alternative approaches such as token introspection, while not requiring additional endpoints on Clients and Resource Servers.

--- middle

# Introduction # {#intro}

Authentication and Authorization for Constrained Environments (ACE) {{I-D.ietf-ace-oauth-authz}} is a framework that enforces access control on IoT devices acting as Resource Servers. In order to use ACE, both Clients and Resource Servers have to register with an Authorization Server and become a registered device. Once registered, a Client can send a request to the Authorization Server, to obtain an Access Token for a Resource Server. For a Client to access the Resource Server, the Client must present the issued Access Token at the Resource Server, which then validates it before storing it (see {{Section 5.10.1.1 of I-D.ietf-ace-oauth-authz}}).

Even though Access Tokens have expiration times, there are circumstances by which an Access Token may need to be revoked before its expiration time, such as: (1) a registered device has been compromised, or is suspected of being compromised; (2) a registered device is decommissioned; (3) there has been a change in the ACE profile for a registered device; (4) there has been a change in access policies for a registered device; and (5) there has been a change in the outcome of policy evaluation for a registered device (e.g., if policy assessment depends on dynamic conditions in the execution environment, the user context, or the resource utilization).

As discussed in {{Section 6.1 of I-D.ietf-ace-oauth-authz}}, only client-initiated revocation is currently specified {{RFC7009}} for OAuth 2.0 {{RFC6749}}, based on the assumption that Access Tokens in OAuth are issued with a relatively short lifetime. However, this is not expected to be the case for constrained, intermittently connected devices, that need Access Tokens with relatively long lifetimes.

This document specifies a method for allowing registered devices to access and possibly subscribe to a Token Revocation List (TRL) resource on the Authorization Server, in order to obtain an updated list of revoked, but yet not expired, pertaining Access Tokens. In particular, registered devices can subscribe to the TRL at the Authorization Server by using resource observation {{RFC7641}} for the Constrained Application Protocol (CoAP) {{RFC7252}}.

Unlike in the case of token introspection (see {{Section 5.9 of I-D.ietf-ace-oauth-authz}}), a registered device does not provide an owned Access Token to the Authorization Server for inquiring about its current state. Instead, registered devices simply obtain an updated list of revoked, but yet not expired, pertaining Access Tokens, as efficiently identified by corresponding hash values.

The benefits of this method are that it complements token introspection, and it does not require any additional endpoints on the registered devices. The only additional requirements for registered devices are a request/response interaction with the Authorization Server to access and possibly subscribe to the TRL (see {{sec-overview}}), and the lightweight computation of hash values to use as Token identifiers (see {{sec-token-name}}).

## Terminology ## {#terminology}

{::boilerplate bcp14}

Readers are expected to be familiar with the terms and concepts described in the ACE framework for Authentication and Authorization {{I-D.ietf-ace-oauth-authz}}, as well as with terms and concepts related to CBOR Web Tokens (CWTs) {{RFC8392}}, and JSON Web Tokens (JWTs) {{RFC7519}}. The terminology for entities in the considered architecture is defined in OAuth 2.0 {{RFC6749}}. In particular, this includes Client, Resource Server, and Authorization Server.

Readers are also expected to be familiar with the terms and concepts related to CBOR {{RFC8949}}, JSON {{RFC8259}}, the CoAP protocol {{RFC7252}}, CoAP Observe {{RFC7641}}, and the use of hash functions to name objects as defined in {{RFC6920}}.

Note that, unless otherwise indicated, the term "endpoint" is used here following its OAuth definition, aimed at denoting resources such as /token and /introspect at the Authorization Server, and /authz-info at the Resource Server. This document does not use the CoAP definition of "endpoint", which is "An entity participating in the CoAP protocol."

This specification also refers to the following terminology.

* Token hash: identifier of an Access Token, in binary format encoding. The token hash has no relation to other possibly used token identifiers, such as the "cti" (CWT ID) claim of CBOR Web Tokens (CWTs) {{RFC8392}}.

* Token Revocation List (TRL): a collection of token hashes such that the corresponding Access Tokens have been revoked but are not expired yet.

* TRL resource: a resource on the Authorization Server, with a TRL as its representation.

* TRL endpoint: an endpoint at the Authorization Server associated with the TRL resource. The default name of the TRL endpoint in a url-path is '/revoke/trl'. Implementations are not required to use this name, and can define their own instead.

* Registered device: a device registered at the Authorization Server, i.e., as a Client, or a Resource Server, or both. A registered device acts as a caller of the TRL endpoint.

* Administrator: entity authorized to get full access to the TRL at the Authorization Server, and acting as a caller of the TRL endpoint. An administrator is not necessarily a registered device as defined above, i.e., a Client requesting Access Tokens or a Resource Server consuming Access Tokens. How the administrator authorization is established and verified is out of the scope of this specification.

* Pertaining Access Token:

   - With reference to an administrator, an Access Token issued by the Authorization Server.

   - With reference to a registered device, an Access Token intended to be owned by that device. An Access Token pertains to a Client if the Authorization Server has issued the Access Token and provided it to that Client. An Access Token pertains to a Resource Server if the Authorization Server has issued the Access Token to be consumed by that Resource Server.

Examples throughout this document are expressed in CBOR diagnostic notation without the tag and value abbreviations.

# Protocol Overview # {#sec-overview}

This protocol defines how a CoAP-based Authorization Server informs Clients and Resource Servers, i.e., registered devices, about pertaining revoked Access Tokens. How the relationship between a registered device and the Authorization Server is established is out of the scope of this specification.

At a high level, the steps of this protocol are as follows.

* Upon startup, the Authorization Server creates a single TRL resource. At any point in time, the TRL resource represents the list of all revoked Access Tokens issued by the Authorization Server that are not expired yet.

* When a device registers at the Authorization Server, it also receives the url-path to the TRL resource.

   After the registration procedure is finished, the registered device can send an Observation Request to the TRL resource as described in {{RFC7641}}, i.e., a GET request including the CoAP Observe Option set to 0 (register). By doing so, the registered device effectively subscribes to the TRL resource, as interested to receive notifications about its update. Upon receiving the request, the Authorization Server adds the registered device to the list of observers of the TRL resource.

   At any time, the registered device can send a GET request to the TRL endpoint. When doing so, it can request for: the current list of pertaining revoked Access Tokens (see {{ssec-trl-full-query}}); or the most recent TRL updates occurred over the list of pertaining revoked Access Tokens (see {{ssec-trl-diff-query}}). In either case, the registered device may also rely on an Observation Request for subscribing to the TRL resource as discussed above.

* When an Access Token is revoked, the Authorization Server adds the corresponding token hash to the TRL. Also, when a revoked Access Token eventually expires, the Authorization Server removes the corresponding token hash from the TRL.

   In either case, after updating the TRL, the Authorization Server sends Observe Notifications as per {{RFC7641}}. That is, an Observe Notification is sent to each registered device subscribed to the TRL resource and to which the Access Token pertains.

   Depending on the specific subscription established through the observation request, the notification provides the current updated list of revoked Access Tokens in the portion of the TRL pertaining to that device (see {{ssec-trl-full-query}}), or rather the most recent TRL updates occurred over that list of pertaining revoked Access Tokens (see {{ssec-trl-diff-query}}).

   Further Observe Notifications may be sent, consistently with ongoing additional observations of the TRL resource.

* An administrator can access and subscribe to the TRL like a registered device, while getting the full updated representation of the TRL.

{{fig-protocol-overview}} shows a high-level overview of the service provided by this protocol. In particular, it shows the Observe Notifications sent by the Authorization Server to one administrator and four registered devices, upon revocation of the issued Access Tokens t1, t2 and t3, with token hash th1, th2 and th3, respectively. Each dotted line associated with a pair of registered devices indicates the Access Token that they both own.

~~~~~~~~~~~
                    +----------------------+
                    | Authorization Server |
                    +-----------o----------+
                    revoke/trl  |  TRL: {th1,th2,th3}
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

{{sec-RS-examples}} provides examples of the protocol flow and message exchange between the Authorization Server and a registered device.

# Token Hash # {#sec-token-name}

The token hash of an Access Token is computed as follows.

1. The Authorization Server defines ENCODED\_TOKEN, as the content of the 'access\_token' parameter in the Authorization Server response (see {{Section 5.8.2 of I-D.ietf-ace-oauth-authz}}), where the Access Token was included and provided to the requesting Client.

    Note that the content of the 'access\_token' parameter is either:

    * A CBOR byte string, if the Access Token was transported using CBOR. With reference to the example in {{fig-as-response-cbor}}, and assuming the string's length in bytes to be 119 (i.e., 0x77 in hexadecimal), then ENCODED\_TOKEN takes the bytes \{0x58 0x77 0xd0 0x83 0x44 0xa1 ...\}, i.e., the raw content of the parameter 'access\_token'.

   * A text string, if the Access Token was transported using JSON. With reference to the example in {{fig-as-response-json}}, ENCODED\_TOKEN takes "2YotnFZFEjr1zCsicMWpAA", i.e., the raw content of the parameter 'access\_token'.

2. The Authorization Server defines HASH\_INPUT as follows.

   * If CBOR was used to transport the Access Token (as a CWT or JWT), HASH\_INPUT takes the same value of ENCODED\_TOKEN.

   * If JSON was used to transport the Access Token (as a CWT or JWT), HASH\_INPUT takes the serialization of ENCODED\_TOKEN.

      In either case, HASH\_INPUT results in the binary representation of the content of the 'access\_token' parameter from the Authorization Server response.

3. The Authorization Server generates a hash value of HASH\_INPUT as per {{Section 6 of RFC6920}}. The resulting output in binary format is used as the token hash. Note that the used binary format embeds the identifier of the used hash function, in the first byte of the computed token hash.

   The specifically used hash function MUST be collision-resistant on byte-strings, and MUST be selected from the "Named Information Hash Algorithm" Registry {{Named.Information.Hash.Algorithm}}.

   The Authorization Server specifies the used hash function to registered devices during their registration procedure (see {{sec-registration}}).

~~~~~~~~~~~
2.01 Created
Content-Format: application/ace+cbor
Max-Age: 85800
Payload:
{
   "access_token" : h'd08344a1 ...
    (remainder of the Access Token omitted for brevity) ...',
   "token_type" : pop,
   "expires_in" : 86400,
   "profile" : coap_dtls,
   (remainder of the response omitted for brevity)
}
~~~~~~~~~~~
{: #fig-as-response-cbor title="Example of Authorization Server response using CBOR" artwork-align="left"}

~~~~~~~~~~~
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache
Payload:
{
   "access_token" : "2YotnFZFEjr1zCsicMWpAA ...
    (remainder of the Access Token omitted for brevity) ...",
   "token_type" : "pop",
   "expires_in" : 86400,
   "profile" : "coap_dtls",
   (remainder of the response omitted for brevity)
}
~~~~~~~~~~~
{: #fig-as-response-json title="Example of Authorization Server response using JSON" artwork-align="left"}

# The TRL Resource # {#sec-trl-resource}

Upon startup, the Authorization Server creates a single TRL resource, encoded as a CBOR array.

Each element of the array is a CBOR byte string, with value the token hash of an Access Token. The order of the token hashes in the CBOR array is irrelevant, and the CBOR array MUST be treated as a set in which the order of elements has no significant meaning.

The TRL is initialized as empty, i.e., the initial content of the TRL resource representation MUST be an empty CBOR array.

## Update of the TRL Resource ## {#ssec-trl-update}

The Authorization Server updates the TRL in the following two cases.

* When a non-expired Access Token is revoked, the token hash of the Access Token is added to the TRL resource representation. That is, a CBOR byte string with the token hash as its value is added to the CBOR array used as TRL resource representation.

* When a revoked Access Token expires, the token hash of the Access Token is removed from the TRL resource representation. That is, the CBOR byte string with the token hash as its value is removed from the CBOR array used as TRL resource representation.

# The TRL Endpoint # {#sec-trl-endpoint}

Consistent with {{Section 6.5 of I-D.ietf-ace-oauth-authz}}, all communications between a caller of the TRL endpoint and the Authorization Server MUST be encrypted, as well as integrity and replay protected. Furthermore, responses from the Authorization Server to the caller MUST be bound to the caller's request.

Following a request to the TRL endpoint, the messages defined in this document that the Authorization Server sends as response use Content-Format "application/ace-trl+cbor". Their payload is formatted as a CBOR map, and the CBOR values for the parameters included therein are defined in {{trl-registry-parameters}}.

The Authorization Server MUST implement measures to prevent access to the TRL endpoint by entities other than registered devices and authorized administrators.

The TRL endpoint supports only the GET method, and allows two types of query of the TRL.

* Full query: the Authorization Server returns the token hashes of the revoked Access Tokens currently in the TRL and pertaining to the requester.

   The Authorization Server MUST support this type of query. The processing of a full query and the related response format are defined in {{ssec-trl-full-query}}.

* Diff query: the Authorization Server returns a list of diff entries. Each diff entry is related to one of the most recent updates, in the portion of the TRL pertaining to the requester.

   The entry associated with one of such updates contains a list of token hashes, such that: i) the corresponding revoked Access Tokens pertain to the requester; and ii) they were added to or removed from the TRL at that update.

   The Authorization Server MAY support this type of query. In such a case, the Authorization Server maintains the history of updates to the TRL resource as defined in {{sec-trl-endpoint-supporting-diff-queries}}. The processing of a diff query and the related response format are defined in {{ssec-trl-diff-query}}.

If it supports diff queries, the Authorization Server MAY additionally support its "Cursor" extension, which has two benefits. First, the Authorization Server can avoid excessively big latencies when several diff entries have to be transferred, by delivering one adjacent subset at the time, in different diff query responses. Second, a requester can retrieve diff entries associated with TRL updates that, even if not the most recent ones, occurred after a TRL update indicated as reference point.

If it supports the "Cursor" extension, the Authorization Server stores additional information when maintaining the history of updates to the TRL resource, as defined in {{sec-trl-endpoint-supporting-cursor}}. Also, the processing of full query requests and diff query requests, as well as the related response format, are further extended as defined in {{sec-using-cursor}}.

## Supporting Diff Queries # {#sec-trl-endpoint-supporting-diff-queries}

If the Authorization Server supports diff queries, it is able to transfer a list of diff entries, as a series of TRL updates. That is, when replying to a diff query performed by a requester, the Authorization Server specifies the most recent updates to the portion of the TRL pertaining to that requester.

The following defines how the Authorization Server builds and maintains consistent histories of TRL updates for each registered device and administrator, hereafter referred to as requesters.

For each requester, the Authorization Server maintains an update collection of maximum N\_MAX series items, where N\_MAX is a pre-defined positive integer. The Authorization Server MUST keep track of the N\_MAX most recent updates to the portion of the TRL that pertains to each requester. The Authorization Server SHOULD provide requesters with the value of N\_MAX, upon their registration (see {{sec-registration}}).

The series items in the update collection MUST be strictly ordered in a chronological fashion. That is, at any point in time, the current first series item is the one least recently added to the update collection and still retained by the Authorization Server, while the current last series item is the one most recently added to the update collection. The particular method used to achieve this is implementation-specific.

Each time the TRL changes, the Authorization Server performs the following operations for each requester.

1. The Authorization Server considers the portion of the TRL pertaining to that requester. If the TRL portion is not affected by this TRL update, the Authorization Server stops the processing for that requester.

2. Otherwise, the Authorization Server creates two sets "trl_patch" of token hashes, i.e., one  "removed" set and one "added" set, as related to this TRL update.

3. The Authorization Server fills the two sets with the token hashes of the removed and added Access Tokens, respectively, from/to the TRL portion considered at step 1.

4. The Authorization Server creates a new series item, which includes the two sets from step 3.

5. If the update collection associated with the requester currently includes N\_MAX series items, the Authorization Server MUST delete the oldest series item in the update collection.

   This occurs when the number of TRL updates pertaining to the requester and currently stored at the Authorization Server is equal to N\_MAX.

6. The Authorization Server adds the series item to the update collection associated with the requester, as the most recent one.

### Supporting the "Cursor" Extension # {#sec-trl-endpoint-supporting-cursor}

If it supports the "Cursor" extension for diff queries, the Authorization Server performs also the following actions.

When maintaining the history of updates to the TRL resource, the following applies for each update collection.

* Each series item X in the update collection is also associated with an unsigned integer 'index'. The value of 'index' is the absolute counter of series items added to that update collection until and including X, minus 1.

   That is, the first series item ever added to that update collection has 'index' with value 0. Also, after having added V series item to the update collection, the last series item has 'index' with value (V - 1), independently of the maximum possible size of the update collection (i.e., N\_MAX).

* The unsigned integer LAST_INDEX is also defined. At every point in time, the value of LAST_INDEX is the value of 'index' currently associated with the latest added series item in the update collection.

   That is, the value of LAST_INDEX is the total number of series items added so far to the update collection, minus 1.

When processing a diff query using the "Cursor" extension, the values of 'index' are used as cursor information, as defined in {{sec-using-cursor-diff-query-response}}.

Furthermore, the Authorization Server defines an unsigned integer MAX_DIFF_BATCH <= N_MAX. The value of MAX_DIFF_BATCH specifies the maximum number of diff entries to be included in a single diff query response. If supporting the "Cursor" extension, the Authorization Server SHOULD provide registered devices and administrators with the value of MAX_DIFF_BATCH, upon their registration (see {{sec-registration}}).

## Query Parameters # {#sec-trl-endpoint-query-parameters}

The TRL endpoint allows the following query parameters to be present in a GET request. The Authorization Server MUST silently ignore unknown query parameters.

* 'pmax': if included, it follows the semantics defined in {{Section 3.2.2 of I-D.ietf-core-conditional-attributes}}. This query parameter is relevant only in case the GET request is specifically an Observation Request, i.e., if it includes the CoAP Observe Option set to 0 (register). In such a case, this parameter indicates the maximum time, in seconds, between two consecutive notifications for the observation in question, regardless whether the TRL resource has changed or not.

   If the Observation Request does not include the 'pmax' parameter, the maximum time to consider is up to the Authorization Server. If the Observation Request includes the 'pmax' parameter, its value MUST be greater than zero, otherwise the Authorization Server MUST return a 4.00 (Bad Request) response.

   If the GET request is not an Observation Request, the Authorization Server MUST ignore the 'pmax' parameter, in case this is included.

* 'diff': if included, it indicates to perform a diff query of the TRL (see {{ssec-trl-diff-query}}). Its value MUST be either:

   - the integer 0, indicating that a (notification) response should include as many diff entries as the Authorization Server can provide in the response; or

   - a positive integer greater than 0, indicating the maximum number of diff entries that a (notification) response should include.

   If the Authorization Server does not support diff queries, it ignores the query parameter 'diff' when present in the GET request and proceeds like when processing a full query of the TRL (see {{ssec-trl-full-query}}).

   Otherwise, the Authorization Server MUST return a 4.00 (Bad Request) response in case the query parameter 'diff' of the GET request specifies a value other than 0 or than a positive integer. The response MUST have Content-Format "application/ace-trl+cbor". The payload of the response is a CBOR map, which MUST include the 'error' field with value 0 ("Invalid parameter value") and MAY include the 'error_description' field to provide additional context.

* 'cursor': if included, it indicates to perform a diff query of the TRL together with the "Cursor" extension, as defined in {{sec-using-cursor-diff-query-response}}. Its value MUST be either 0 or a positive integer.

   If the Authorization Server does not support the "Cursor" extension, it ignores the query parameter 'cursor' when present in the GET request. In such a case, the Authorization Server proceeds: i) like when processing a diff query of the TRL (see {{ssec-trl-diff-query}}), if it supports diff queries and the query parameter 'diff' is present in the GET request, or ii) like when processing a full query of the TRL (see {{ssec-trl-full-query}}) otherwise.

   If the Authorization Server supports both diff queries and the "Cursor" extension, and the GET request specifies the query parameter 'cursor', then the Authorization Server MUST return a 4.00 (Bad Request) response in case any of the following conditions holds.

   * The GET request does not specify the query parameter 'diff'. The 'error' parameter within the CBOR map carried in the response payload MUST have value 1 ("Invalid set of parameters").

   * The query parameter 'cursor' has a value other than 0 or than a positive integer. The 'error' parameter within the CBOR map carried in the response payload MUST have value 0 ("Invalid parameter value").

   * The query parameter 'cursor' has a value strictly greater than the current LAST_INDEX for the update collection associated with the requester (see {{sec-trl-endpoint-supporting-cursor}}). The 'error' parameter within the CBOR map carried in the response payload MUST have value 2 ("Out of bound cursor value"). The CBOR map MUST also include the 'cursor' parameter, which MUST specify the current value of LAST_INDEX for the update collection associated with the requester.

   The 4.00 (Bad Request) response MUST have Content-Format "application/ace-trl+cbor". The payload of the response MUST be a CBOR map, which MUST include the 'error' parameter and MAY include the 'error_description' parameter to provide additional context.

# Full Query of the TRL ## {#ssec-trl-full-query}

In order to produce a (notification) response to a GET request asking for a full query of the TRL, the Authorization Server performs the following actions.

1. From the current TRL resource representation, the Authorization Server builds a set HASHES, such that:

    * If the requester is a registered device, HASHES specifies the token hashes of the Access Tokens pertaining to that registered device. The Authorization Server can use the authenticated identity of the registered device to perform the necessary filtering on the TRL resource representation.

    * If the requester is an administrator, HASHES specifies all the token hashes in the current TRL resource representation.

2. The Authorization Server sends a 2.05 (Content) response to the requester. The response MUST have Content-Format "application/ace-trl+cbor". The payload of the response is a CBOR map, which MUST be formatted as follows.

   * The 'full_set' parameter MUST be included and specifies a CBOR array 'full_set_value'. Each element of 'full_set_value' specifies one of the token hashes from the set HASHES, encoded as a CBOR byte string. If the set HASHES is empty, the 'full_set' parameter specifies the empty CBOR array.

      The order of the token hashes in the CBOR array is irrelevant, i.e., the CBOR array MUST be treated as a set in which the order of elements has no significant meaning.

   * The 'cursor' parameter MUST be included if the Authorization Server supports both the diff queries and the related "Cursor" extension (see {{sec-trl-endpoint-supporting-diff-queries}} and {{sec-trl-endpoint-supporting-cursor}}). Its value is specified according to what is defined in {{sec-using-cursor-full-query-response}}, and provides the requester with information for performing a follow-up diff query using the "Cursor" extension (see {{sec-using-cursor-diff-query-response}}).

      If the Authorization Server does not support both diff queries and the "Cursor" extension, this parameter MUST NOT be included. In case the requester does not support both diff queries and the "Cursor" extension, it MUST silently ignore the 'cursor' parameter if present.

{{cddl-full}} provides the CDDL definition {{RFC8610}} of the CBOR array 'full_set_value' specified in the response from the Authorization Server, as value of the 'full_set' parameter.

~~~~~~~~~~~ CDDL
token_hash = bytes
full_set_value = [* token_hash]
~~~~~~~~~~~
{: #cddl-full title="CDDL definition of 'full_set_value'" artwork-align="left"}

{{response-full}} shows an example of response from the Authorization Server, following a full query request to the TRL endpoint. Full token hashes are omitted for brevity.

~~~~~~~~~~~
2.05 Content
Content-Format: application/ace-trl+cbor
Payload:
{
   "full_set" : [
     h'01fa51cc ... ', h'01748190 ... '
   ]
}
~~~~~~~~~~~
{: #response-full title="Example of response following a Full Query request to the TRL endpoint" artwork-align="left"}

# Diff Query of the TRL ## {#ssec-trl-diff-query}

In order to produce a (notification) response to a GET request asking for a diff query of the TRL, the Authorization Server performs the following actions.

1. The Authorization Server defines the positive integer NUM as follows. If the value N specified in the query parameter 'diff' in the GET request is equal to 0 or greater than the pre-defined positive integer N\_MAX (see {{sec-trl-endpoint-supporting-diff-queries}}), then NUM takes the value of N_MAX. Otherwise, NUM takes N.

2. The Authorization Server determines U = min(NUM, SIZE), where SIZE <= N_MAX is the number of TRL updates pertaining to the requester and currently stored at the Authorization Server.

3. The Authorization Server prepares U diff entries. If U is equal to 0 (e.g., because SIZE is equal to 0 at step 2), then no diff entries are prepared.

    The prepared diff entries are related to the U most recent TRL updates pertaining to the requester, as maintained in the update collection for that requester (see {{sec-trl-endpoint-supporting-diff-queries}}). In particular, the first diff entry refers to the most recent of such updates, the second diff entry refers to the second from last of such updates, and so on.

    Each diff entry is a CBOR array 'diff_entry', which includes the following two elements.

    * The first element is a CBOR array 'removed'. Each element of the array is a CBOR byte string, with value the token hash of an Access Token such that: it pertained to the requester; and it was removed from the TRL during the update associated with the diff entry.

    * The second element is a CBOR array 'added'. Each element of the array is a CBOR byte string, with value the token hash of an Access Token such that: it pertains to the requester; and it was added to the TRL during the update associated with the diff entry.

    The order of the token hashes in the CBOR arrays 'removed' and 'added' is irrelevant. That is, the CBOR arrays 'removed' and 'added' MUST be treated as a set in which the order of elements has no significant meaning.

4. The Authorization Server prepares a 2.05 (Content) response for the requester. The response MUST have Content-Format "application/ace-trl+cbor". The payload of the response is a CBOR map, which MUST be formatted as follows.

   * The 'diff_set' parameter MUST be present and specifies a CBOR array 'diff_set_value' of U elements. Each element of 'diff_set_value' specifies one of the CBOR arrays 'diff_entry' prepared above as diff entry. Note that U might have value 0, in which case 'diff_set_value' is the empty CBOR array.

      Within 'diff_set_value', the CBOR arrays 'diff_entry' MUST be sorted to reflect the corresponding updates to the TRL in reverse chronological order. That is, the first 'diff_entry' element of 'diff_set_value' relates to the most recent update to the portion of the TRL pertaining to the requester. The second 'diff_entry' element relates to the second from last most recent update to that portion, and so on.

   * The 'cursor' parameter and the 'more' parameters MUST be included if the Authorization Server supports both the diff queries and the related "Cursor" extension (see {{sec-trl-endpoint-supporting-cursor}}). Their values are specified according to what is defined in {{sec-using-cursor-diff-query-response}}, and provide the requester with information for performing a follow-up query to the TRL endpoint (see {{sec-using-cursor-diff-query-response}}).

      If the Authorization Server does not support both diff queries and the "Cursor" extension, these parameters MUST NOT be included. In case the requester does not support both diff queries and the "Cursor" extension, it MUST silently ignore the 'cursor' parameter and the 'more' parameter if present.

{{cddl-diff}} provides the CDDL definition {{RFC8610}} of the CBOR array 'diff_set_value' specified in the response from the Authorization Server, as value of the 'diff_set' parameter.

~~~~~~~~~~~ CDDL
   token_hash = bytes
   trl_patch = [* token_hash]
   diff_entry = [removed: trl_patch, added: trl_patch]
   diff_set_value = [* diff_entry]
~~~~~~~~~~~
{: #cddl-diff title="CDDL definition of 'diff_set_value'" artwork-align="left"}

{{response-diff}} shows an example of response from the Authorization Server, following a Diff Query request to the TRL endpoint, where U = 3 diff entries are specified. Full token hashes are omitted for brevity.

~~~~~~~~~~~
2.05 Content
Content-Format: application/ace-trl+cbor
Payload:
{
   "diff_set" : [
     [
       [ h'01fa51cc ... ', h'01748190 ... '],
       [ h'01cdf1ca ... ', h'01be41a6 ... ']
     ],
     [
       [ h'0144dd12 ... ', h'01231fff ... '],
       []
     ],
     [
       [],
       [ h'01ca986f ... ', h'01fe1a2b ... ']
     ]
   ]
}
~~~~~~~~~~~
{: #response-diff title="Example of response following a Diff Query request to the TRL endpoint" artwork-align="left"}

{{sec-series-pattern}} discusses how performing a diff query of the TRL is in fact a usage example of the Series Transfer Pattern defined in {{I-D.bormann-t2trg-stp}}.

# Using the "Cursor" Extension ## {#sec-using-cursor}

If it supports both diff queries and the "Cursor" extension, the Authorization Server composes a response to a full query request or diff query request as defined in {{sec-using-cursor-full-query-response}} and {{sec-using-cursor-diff-query-response}}, respectively.

The exact format of the response depends on the request being a full query or diff query request, on the presence of the query parameter 'cursor' in the diff query request, and on the current status of the update collection associated with the requester.

## Response to Full Query {#sec-using-cursor-full-query-response}

When processing a full query request to the TRL endpoint, the Authorization Server composes a response as defined in {{ssec-trl-full-query}}.

In particular, the 'cursor' parameter included in the CBOR map carried in the response payload specifies either the CBOR simple value "null" (0xf6) or a CBOR unsigned integer.

The 'cursor' parameter MUST specify the CBOR simple value "null" in case there are currently no TRL updates pertinent to the requester, i.e., the update collection for that requester is empty. This is the case from when the requester registers at the Authorization Server until a first update pertaining to that requester occurs to the TRL.

Otherwise, the 'cursor' parameter MUST specify a CBOR unsigned integer. This MUST take the 'index' value of the last series item in the update collection associated with the request (see {{sec-trl-endpoint-supporting-cursor}}), as corresponding to the most recent update pertaining to the requester occurred to the TRL.

## Response to Diff Query {#sec-using-cursor-diff-query-response}

When processing a diff query request to the TRL endpoint, the Authorization Server composes a response as defined in the following.

### Empty Collection {#sec-using-cursor-diff-query-response-empty}

If the update collection associated with the requester has no elements, the Authorization Server returns a 2.05 (Content) response. The response MUST have Content-Format "application/ace-trl+cbor" and its payload MUST be a CBOR map formatted as follows.

* The 'diff_set' parameter MUST be included and specifies the empty CBOR array.

* The 'cursor' parameter MUST be included and specifies the CBOR simple value "null" (0xf6).

* The 'more' parameter MUST be included and specifies the CBOR simple value "false" (0xf4).

Note that the above applies when the update collection associated with the requester has no elements, regardeless whether the query parameter 'cursor' is included or not in the diff query request.

### Cursor Not Specified in the Diff Query Request {#sec-using-cursor-diff-query-response-no-cursor}

If the update collection associated with the requester is not empty and the diff query request does not include the query parameter 'cursor', the Authorization Server performs the same actions defined in {{ssec-trl-diff-query}}, with the following differences.

* At step 3, the Authorization Server considers the value MAX_DIFF_BATCH (see {{sec-trl-endpoint-supporting-cursor}}), and prepares L = min(U, MAX_DIFF_BATCH) diff entries. If L is equal to 0 (e.g., because U is equal to 0), then no diff entries are prepared.

   If U <= MAX_DIFF_BATCH, the prepared diff entries are the last series items in the update collection associated with the requester, corresponding to the L most recent TRL updates pertaining to the requester.

   If U > MAX_DIFF_BATCH, the prepared diff entries are the eldest of the last U series items in the update collection associated with the requester, as corresponding to the first L of the U most recent TRL updates pertaining to the requester.

* At step 4, the CBOR map to carry in the payload of the 2.05 (Content) response MUST be formatted as follows.

   * The 'diff_set' parameter MUST be present and specifies a CBOR array 'diff_set_value' of L elements. Each element of 'diff_set_value' specifies one of the CBOR arrays 'diff_entry' prepared as diff entry. Note that L might have value 0, in which case 'diff_set_value' is the empty CBOR array.

   * The 'cursor' parameter MUST be present and specifies a CBOR unsigned integer. This MUST take the 'index' value of the series item of the update collection included as first diff entry in the 'diff_set_value' CBOR array, which is specified by the 'diff_set' parameter. That is, the 'cursor' parameter takes the 'index' value of the series item in the update collection corresponding to the most recent update pertaining to the requester and returned in this diff query response.

      Note that the 'cursor' parameter takes the same 'index' value of the last series item in the update collection when U <= MAX_DIFF_BATCH.

   * The 'more' parameter MUST be present and MUST specify the CBOR simple value "false" (0xf4) if U <= MAX_DIFF_BATCH, or the CBOR simple value "true" (0xf5) otherwise.

      If the 'more' parameter has value "true", the requester can send a follow-up diff query request including the query parameter 'cursor', with the same value of the 'cursor' parameter specified in this diff query response. As defined in {{sec-using-cursor-diff-query-response-cursor}}, this would result in the Authorization Server transferring the following subset of series items as diff entries, thus resuming from where interrupted in the previous transfer.

### Cursor Specified in the Diff Query Request {#sec-using-cursor-diff-query-response-cursor}

If the update collection associated with the requester is not empty and the diff query request includes the query parameter 'cursor' with value P, the Authorization Server proceeds as follows, depending on which of the following two cases hold.

* Case A - The series item X with 'index' having value P and the series item Y with 'index' having value P+1 are both not found in the update collection associated with the requester. This occurs when the item Y (and possibly further ones after it) has been previously removed from the history of updates for that requester (see step 5 at {{sec-trl-endpoint-supporting-diff-queries}}).

   In this case, the Authorization Server returns a 2.05 (Content) response. The response MUST have Content-Format "application/ace-trl+cbor" and its payload MUST be a CBOR map formatted as follows.

    * The 'diff_set' parameter MUST be included and specifies the empty CBOR array.

    * The 'cursor' parameter MUST be included and specifies the CBOR simple value "null" (0xf6).

    * The 'more' parameter MUST be included and specifies the CBOR simple value "true" (0xf5).

   With the combination ('cursor', 'more') = ("null", "true"), the Authorization Server is signaling that the update collection is in fact not empty, but that one or more series items have been lost due to their removal. These include the item with 'index' value P+1, that the requester wished to obtain as the first one following the specified reference point with 'index' value P.

   When receiving this diff query response, the requester should send a new full query request to the Authorization Server, in order to fully retrieve the current pertaining portion of the TRL.

* Case B - The series item X with 'index' having value P is found in the update collection associated with the requester; or the series item X is not found and the series item Y with 'index' having value P+1 is found in the update collection associated with the requester.

   In this case, the Authorization Server performs the same actions defined in {{ssec-trl-diff-query}}, with the following differences.

   * At step 3, the Authorization Server considers the value MAX_DIFF_BATCH (see {{sec-trl-endpoint-supporting-cursor}}), and prepares L = min(SUB_U, MAX_DIFF_BATCH) diff entries, where SUB_U = min(NUM, SUB_SIZE), and SUB_SIZE is the number of series items in the update collection following the series item X. If L is equal to 0 (e.g., because SUB_U is equal to 0), then no diff entries are prepared.

      If SUB_U <= MAX_DIFF_BATCH, the prepared diff entries are the last series items in the update collection associated with the requester, corresponding to the L most recent TRL updates pertaining to the requester.

      If SUB_U > MAX_DIFF_BATCH, the prepared diff entries are the eldest of the last SUB_U series items in the update collection associated with the requester, corresponding to the first L of the SUB_U most recent TRL updates pertaining to the requester.

   * At step 4, the CBOR map to carry in the payload of the 2.05 (Content) response MUST be formatted as follows.

      * The 'diff_set' parameter MUST be present and specifies a CBOR array 'diff_set_value' of L elements. Each element of 'diff_set_value' specifies one of the CBOR arrays 'diff_entry' prepared as diff entry. Note that L might have value 0, in which case 'diff_set_value' is the empty CBOR array.

      * The 'cursor' parameter MUST be present and MUST specify a CBOR unsigned integer. In particular:

         - If L is equal to 0, i.e., the series item X is the last one in the update collection, then the 'cursor' parameter MUST take the same 'index' value of the last series item in the update collection.

         - If L is different than 0, then the 'cursor' parameter MUST take the 'index' value of the series element of the update collection included as first diff entry in the 'diff_set' CBOR array. That is, the 'cursor' parameter takes the 'index' value of the series item in the update collection corresponding to the most recent update pertaining to the requester and returned in this diff query response.

         Note that the 'cursor' parameter takes the same 'index' value of the last series item in the update collection when SUB_U <= MAX_DIFF_BATCH.

      * The 'more' parameter MUST be present and MUST specify the CBOR simple value "false" (0xf4) if SUB_U <= MAX_DIFF_BATCH, or the CBOR simple value "true" (0xf5) otherwise.

         If 'more' has value "true", the requester can send a follow-up diff query request including the query parameter 'cursor', with the same value of the 'cursor' parameter specified in this diff query response. This would result in the Authorization Server transferring the following subset of series items as diff entries, thus resuming from where interrupted in the previous transfer.

# Upon Registration # {#sec-registration}

During the registration process at the Authorization Server, an administrator or a registered device receives the following information as part of the registration response.

* The url-path to the TRL endpoint at the Authorization Server.

* The hash function used to compute token hashes. This is specified as an integer or a text string, taking value from the "ID" or "Hash Name String" column of the "Named Information Hash Algorithm" Registry {{Named.Information.Hash.Algorithm}}, respectively.

* Optionally, a positive integer N\_MAX, if the Authorization Server supports diff queries of the TRL resource (see {{sec-trl-endpoint-supporting-diff-queries}} and {{ssec-trl-diff-query}}).

* Optionally, a positive integer MAX\_DIFF\_BATCH, if the Authorization Server supports diff queries of the TRL resource as well as the related "Cursor" extension (see {{sec-trl-endpoint-supporting-cursor}} and {{sec-using-cursor}}).

After the registration procedure is finished, the administrator or registered device can send a GET request to the TRL resource, including the CoAP Observe Option set to 0 (register), in order to start an observation of the TRL resource at the Authorization Server as per {{Section 3.1 of RFC7641}}. The GET request can express the wish for a full query (see {{ssec-trl-full-query}}) or a diff query (see {{ssec-trl-diff-query}}) of the TRL.

In case the request is successfully processed, the Authorization Server replies with a response specifying the CoAP response code 2.05 (Content) and including the CoAP Observe Option. The payload of the response is formatted as defined in {{ssec-trl-full-query}} or in {{ssec-trl-diff-query}}, in case the GET request yielded the execution of a full query or a diff query of the TRL, respectively.

Further details about the registration process at the Authorization Server are out of scope for this specification. Note that the registration process is also out of the scope of the ACE framework for Authentication and Authorization (see {{Section 5.5 of I-D.ietf-ace-oauth-authz}}).

# Notification of Revoked Tokens # {#sec-notification}

When the TRL is updated (see {{ssec-trl-update}}), the Authorization Server sends Observe Notifications to the observers of the TRL resource. Observe Notifications are sent as per {{Section 4.2 of RFC7641}}.

If the 'pmax' query parameter was specified in the Observation Request starting an observation (see {{sec-trl-endpoint-query-parameters}}), the Authorization Server might accordingly send additional Observe Notifications to the associated observer. That is, the Authorization Server ensures that no more than pmax seconds elapse between two consecutive notifications sent to that observer, regardless whether the TRL resource has changed or not. If the 'pmax' query parameter was not specified in the Observation Request, a possible maximum time to consider is up to the Authorization Server.

The payload of each Observe Notification is formatted as defined in {{ssec-trl-full-query}} or in {{ssec-trl-diff-query}}, in case the original Observation Request yielded the execution of a full query or a diff query of the TRL, respectively.

Furthermore, an administrator or a registered device can send additional GET requests to the TRL endpoint at any time, in order to retrieve the token hashes of the pertaining revoked Access Tokens. When doing so, the caller of the TRL endpoint can perform a full query (see {{ssec-trl-full-query}}) or a diff query (see {{ssec-trl-diff-query}}) of the TRL.

When receiving a response from the TRL endpoint, a registered device MUST expunge every stored Access Token associated with a token hash specified in the response.

When a Resource Server RS receives a response from the TRL endpoint specifying the token hash th1 associated with a revoked Access Token t1, the RS might not have received and stored that Access Token yet. This occurs if the Access Token is revoked before it is successfully posted to the Authorization Information Endpoint at the RS (see {{Section 5.10.1 of I-D.ietf-ace-oauth-authz}}). Such a delay can be due, for example, to messages that get lost in transmission, or rather to the Client experiencing failures in sending the Access Token to the RS, or deliberately holding the Access Token back.

Thus, in order to ensure that no revoked Access Tokens are accepted and stored, the RS performs the following actions.

   * The RS MUST store the token hash th1, until gaining knowledge that the associated revoked Access Token t1 is also expired.

      This can happen when receiving a subsequent response from the TRL endpoint (i.e., indicating that the token hash th1 is not in the TRL portion pertaining to the RS anymore), or when the Access Token t1 is posted to the Authorization Information Endpoint and is found to be expired based on its 'exp' claim {{RFC7519}}, if included.

   * The RS MUST NOT accept as valid and store an Access Token t1 posted to the Authorization Information Endpoint, if the corresponding token hash th1 is among the stored ones.

# Interaction Examples # {#sec-RS-examples}

This section provides examples of interactions between a Resource Server RS as a registered device and an Authorization Server AS. The Authorization Server supports both full query and diff query of the TRL, as defined in {{ssec-trl-full-query}} and {{ssec-trl-diff-query}}, respectively.

The details of the registration process are omitted, but it is assumed that the Resource Server sends an unspecified payload to the Authorization Server, which replies with a 2.01 (Created) response.

The payload of the registration response is a CBOR map, which includes the following entries:

* a "trl_path" parameter, specifying the path of the TRL resource;

* a "trl_hash" parameter, specifying the hash function used to computed token hashes as defined in {{sec-token-name}};

* an "n_max" parameter, specifying the value of N_MAX, i.e., the maximum number of TRL updates pertaining to each registered device that the Authorization Server retains for that device (see {{ssec-trl-diff-query}});

* possible further parameters related to the registration process.

Furthermore, 'h(x)' refers to the hash function used to compute the token hashes, as defined in {{sec-token-name}} of this specification and according to {{RFC6920}}. Assuming the usage of CWTs transported in CBOR, 'bstr.h(t1)' and 'bstr.h(t2)' denote the byte-string representations of the token hashes for the Access Tokens t1 and t2, respectively.

## Full Query with Observation # {#sec-RS-example-1}

{{fig-RS-AS}} shows an interaction example considering a CoAP observation and a full query of the TRL.

~~~~~~~~~~~
RS                                                 AS
|                                                   |
| Registration: POST                                |
+-------------------------------------------------->|
|                                                   |
|<--------------------------------------------------+
|                   2.01 CREATED                    |
|                     Payload: {                    |
|                        ...                        |
|                        "trl_path" : "revoke/trl", |
|                        "trl_hash" : "sha-256",    |
|                        "n_max" : 10               |
|                     }                             |
|                                                   |
| GET Observe: 0                                    |
|  coap://as.example.com/revoke/trl/                |
+-------------------------------------------------->|
|                                                   |
|<--------------------------------------------------+
|        2.05 CONTENT Observe: 42                   |
|        Content-Format: "application/ace-trl+cbor" |
|         Payload: {                                |
|           "full_set" : []                         |
|         }                                         |
|                         .                         |
|                         .                         |
|                         .                         |
|                                                   |
|          (Access Tokens t1 and t2 issued          |
|          and successfully submitted to RS)        |
|                         .                         |
|                         .                         |
|                         .                         |
|                                                   |
|                                                   |
|            (Access Token t1 is revoked)           |
|                                                   |
|<--------------------------------------------------+
|        2.05 CONTENT Observe: 53                   |
|        Content-Format: "application/ace-trl+cbor" |
|         Payload: {                                |
|           "full_set" : [bstr.h(t1)]               |
|         }                                         |
|                         .                         |
|                         .                         |
|                         .                         |
|                                                   |
|            (Access Token t2 is revoked)           |
|                                                   |
|<--------------------------------------------------+
|        2.05 CONTENT Observe: 64                   |
|        Content-Format: "application/ace-trl+cbor" |
|         Payload: {                                |
|           "full_set" : [bstr.h(t1), bstr.h(t2)]   |
|        }                                          |
|                                                   |
|                         .                         |
|                         .                         |
|                         .                         |
|                                                   |
|             (Access Token t1 expires)             |
|                                                   |
|<--------------------------------------------------+
|        2.05 CONTENT Observe: 75                   |
|        Content-Format: "application/ace-trl+cbor" |
|         Payload: {                                |
|           "full_set" : [bstr.h(t2)]               |
|         }                                         |
|                         .                         |
|                         .                         |
|                         .                         |
|                                                   |
|            (Access Token t2 expires)              |
|                                                   |
|<--------------------------------------------------+
|        2.05 CONTENT Observe: 86                   |
|        Content-Format: "application/ace-trl+cbor" |
|         Payload: {                                |
|           "full_set" : []                         |
|         }                                         |
|                                                   |
~~~~~~~~~~~
{: #fig-RS-AS title="Interaction for Full Query with Observation" artwork-align="center"}

## Diff Query with Observation # {#sec-RS-example-2}

{{fig-RS-AS-2}} shows an interaction example considering a CoAP observation and a diff query of the TRL.

The Resource Server indicates N=3 as value of the query parameter 'diff', i.e., as the maximum number of diff entries to be specified in a response from the Authorization Server.

~~~~~~~~~~~
RS                                                 AS
|                                                   |
| Registration: POST                                |
+-------------------------------------------------->|
|                                                   |
|<--------------------------------------------------+
|                    2.01 CREATED                   |
|                     Payload: {                    |
|                        ...                        |
|                        "trl_path" : "revoke/trl", |
|                        "trl_hash" : "sha-256",    |
|                        "n_max" : 10               |
|                     }                             |
|                                                   |
| GET Observe: 0                                    |
|  coap://as.example.com/revoke/trl?diff=3          |
+-------------------------------------------------->|
|                                                   |
|<--------------------------------------------------+
|        2.05 CONTENT Observe: 42                   |
|        Content-Format: "application/ace-trl+cbor" |
|         Payload: {                                |
|           "diff_set" : []                         |
|         }                                         |
|                         .                         |
|                         .                         |
|                         .                         |
|                                                   |
|          (Access Tokens t1 and t2 issued          |
|          and successfully submitted to RS)        |
|                         .                         |
|                         .                         |
|                         .                         |
|                                                   |
|            (Access Token t1 is revoked)           |
|                                                   |
|<--------------------------------------------------+
|        2.05 CONTENT Observe: 53                   |
|        Content-Format: "application/ace-trl+cbor" |
|         Payload: {                                |
|           "diff_set" : [                          |
|                          [ [], [bstr.h(t1)] ]     |
|                        ]                          |
|         }                                         |
|                         .                         |
|                         .                         |
|                         .                         |
|                                                   |
|            (Access Token t2 is revoked)           |
|                                                   |
|<--------------------------------------------------+
|        2.05 CONTENT Observe: 64                   |
|        Content-Format: "application/ace-trl+cbor" |
|         Payload: {                                |
|           "diff_set" : [                          |
|                          [ [], [bstr.h(t2)] ],    |
|                          [ [], [bstr.h(t1)] ]     |
|                        ]                          |
|         }                                         |
|                         .                         |
|                         .                         |
|                         .                         |
|                                                   |
|              (Access Token t1 expires)            |
|                                                   |
|<--------------------------------------------------+
|        2.05 CONTENT Observe: 75                   |
|        Content-Format: "application/ace-trl+cbor" |
|         Payload: {                                |
|           "diff_set" : [                          |
|                          [ [bstr.h(t1)], [] ],    |
|                          [ [], [bstr.h(t2)] ],    |
|                          [ [], [bstr.h(t1)] ]     |
|                        ]                          |
|         }                                         |
|                         .                         |
|                         .                         |
|                         .                         |
|                                                   |
|              (Access Token t2 expires)            |
|                                                   |
|<--------------------------------------------------+
|        2.05 CONTENT Observe: 86                   |
|        Content-Format: "application/ace-trl+cbor" |
|         Payload: {                                |
|           "diff_set" : [                          |
|                          [ [bstr.h(t2)], [] ],    |
|                          [ [bstr.h(t1)], [] ],    |
|                          [ [], [bstr.h(t2)] ]     |
|                        ]                          |
|         }                                         |
|                                                   |
~~~~~~~~~~~
{: #fig-RS-AS-2 title="Interaction for Diff Query with Observation" artwork-align="center"}

## Full Query with Observation and Additional Diff Query # {#sec-RS-example-3}

{{fig-RS-AS-3}} shows an interaction example considering a CoAP observation and a full query of the TRL.

The example also considers one of the notifications from the Authorization Server to get lost in transmission, and thus not reaching the Resource Server.

When this happens, and after a waiting time defined by the application has elapsed, the Resource Server sends a GET request with no Observe Option to the Authorization Server, to perform a diff query of the TRL. The Resource Server indicates N=8 as value of the query parameter 'diff', i.e., as the maximum number of diff entries to be specified in a response from the Authorization Server.

~~~~~~~~~~~
RS                                                 AS
|                                                   |
| Registration: POST                                |
+-------------------------------------------------->|
|                                                   |
|<--------------------------------------------------+
|                    2.01 CREATED                   |
|                     Payload: {                    |
|                        ...                        |
|                        "trl_path" : "revoke/trl", |
|                        "trl_hash" : "sha-256",    |
|                        "n_max" : 10               |
|                     }                             |
|                                                   |
| GET Observe: 0                                    |
|  coap://as.example.com/revoke/trl/                |
+-------------------------------------------------->|
|                                                   |
|<--------------------------------------------------+
|        2.05 CONTENT Observe: 42                   |
|        Content-Format: "application/ace-trl+cbor" |
|         Payload: {                                |
|           "full_set" : []                         |
|         }                                         |
|                         .                         |
|                         .                         |
|                         .                         |
|                                                   |
|          (Access Tokens t1 and t2 issued          |
|          and successfully submitted to RS)        |
|                         .                         |
|                         .                         |
|                         .                         |
|                                                   |
|            (Access Token t1 is revoked)           |
|                                                   |
|<--------------------------------------------------+
|        2.05 CONTENT Observe: 53                   |
|        Content-Format: "application/ace-trl+cbor" |
|         Payload: {                                |
|           "full_set" : [bstr.h(t1)]               |
|         }                                         |
|                         .                         |
|                         .                         |
|                         .                         |
|                                                   |
|            (Access Token t2 is revoked)           |
|                                                   |
|<--------------------------------------------------+
|        2.05 CONTENT Observe: 64                   |
|        Content-Format: "application/ace-trl+cbor" |
|         Payload: {                                |
|           "full_set" : [bstr.h(t1), bstr.h(t2)]   |
|         }                                         |
|                         .                         |
|                         .                         |
|                         .                         |
|                                                   |
|             (Access Token t1 expires)             |
|                                                   |
|<--------------------------------------------------+
|        2.05 CONTENT Observe: 75                   |
|        Content-Format: "application/ace-trl+cbor" |
|         Payload: {                                |
|           "full_set" : [bstr.h(t2)]               |
|         }                                         |
|                         .                         |
|                         .                         |
|                         .                         |
|                                                   |
|             (Access Token t2 expires)             |
|                                                   |
|    X<---------------------------------------------+
|        2.05 CONTENT Observe: 86                   |
|        Content-Format: "application/ace-trl+cbor" |
|         Payload: {                                |
|           "full_set" : []                         |
|         }                                         |
|                         .                         |
|                         .                         |
|                         .                         |
|                                                   |
|           (Enough time has passed since           |
|         the latest received notification)         |
|                                                   |
| GET                                               |
|  coap://as.example.com/revoke/trl?diff=8          |
+-------------------------------------------------->|
|                                                   |
|<--------------------------------------------------+
|        2.05 CONTENT                               |
|        Content-Format: "application/ace-trl+cbor" |
|         Payload: {                                |
|           "diff_set" : [                          |
|                          [ [bstr.h(t2)], [] ],    |
|                          [ [bstr.h(t1)], [] ],    |
|                          [ [], [bstr.h(t2)] ],    |
|                          [ [], [bstr.h(t1)] ]     |
|                        ]                          |
|         }                                         |
|                                                   |
~~~~~~~~~~~
{: #fig-RS-AS-3 title="Interaction for Full Query with Observation and Diff Query" artwork-align="center"}


# ACE Token Revocation List Parameters # {#trl-registry-parameters}

This specification defines a number of parameters that can be transported in the response from the TRL endpoint, when the response payload is a CBOR map. Note that such a response MUST use the Content-Format "application/ace-trl+cbor" defined in {{iana-content-type}} of this specification.

The table below summarizes them, and specifies the CBOR value to use as abbreviation instead of the full descriptive name.

~~~~~~~~~~~
+-------------------+------------+------------------------+
| Name              | CBOR Value | CBOR Type              |
+-------------------+------------+------------------------+
| full_set          |  0         | array                  |
+-------------------+------------+------------------------+
| diff_set          |  1         | array                  |
+-------------------+------------+------------------------+
| cursor            |  2         | unsigned integer /     |
|                   |            | simple value "null"    |
+-------------------+------------+------------------------+
| more              |  3         | simple value "false" / |
|                   |            | simple value "true"    |
+-------------------+------------+------------------------+
| error             | -1         | int                    |
+-------------------+------------+------------------------+
| error_description | -2         | tstr                   |
+-------------------+------------+------------------------+
~~~~~~~~~~~
{: #fig-cbor-trl-params title="CBOR abbreviations for the ACE Token Revocation List parameters" artwork-align="center"}


# ACE Token Revocation List Error Identifiers {#error-types}

This specification defines a number of values that the Authorization Server can include as error identifiers, in the 'error' field of an error response from the TRL endpoint. This applies to error responses whose payload is a CBOR map and whose Content-Format is "application/ace-trl+cbor".

~~~~~~~~~~~
+-------+---------------------------+
| Value | Description               |
+-------+---------------------------+
|   0   | Invalid parameter value   |
+-------+---------------------------+
|   1   | Invalid set of parameters |
+-------+---------------------------+
|   2   | Out of bound cursor value |
+-------+---------------------------+
~~~~~~~~~~~
{: #fig-ACE-TRL-Error Identifiers title="ACE Token Revocation List Error Identifiers" artwork-align="center"}


# Security Considerations # {#sec-security-considerations}

Security considerations are inherited from the ACE framework for Authentication and Authorization {{I-D.ietf-ace-oauth-authz}}, from {{RFC8392}} as to the usage of CWTs, from {{RFC7519}} as to the usage of JWTs, from {{RFC7641}} as to the usage of CoAP Observe, and from {{RFC6920}} with regard to resource naming through hashes. The following considerations also apply.

The Authorization Server MUST ensure that each registered device can access and retrieve only its pertaining portion of the TRL. To this end, the Authorization Server can perform the required filtering based on the authenticated identity of the registered device, i.e., a (non-public) identifier that the Authorization Server can securely relate to the registered device and the secure association that they use to communicate.

Disclosing any information about revoked Access Tokens to entities other than the intended registered devices may result in privacy concerns. Therefore, the Authorization Server MUST ensure that, other than registered devices accessing their own pertaining portion of the TRL, only authorized and authenticated administrators can retrieve the full TRL. To this end, the Authorization Server may rely on an access control list or similar.

If a registered device has many non-expired Access Tokens associated with itself that are revoked, the pertaining portion of the TRL could grow to a size bigger than what the registered device is prepared to handle upon reception, especially if relying on a full query of the TRL resource (see {{ssec-trl-full-query}}). This could be exploited by attackers to negatively affect the behavior of a registered device. Issuing Access Tokens with not too long expiration time could help reduce the size of a TRL, but an Authorization Server SHOULD take measures to limit this size.

Most of the communication about revoked Access Tokens presented in this specification relies on CoAP Observe Notifications sent from the Authorization Server to a registered device. The suppression of those notifications by an external attacker that has access to the network would prevent registered devices from ever knowing that their pertaining Access Tokens have been revoked. In order to avoid this, a registered device SHOULD NOT rely solely on the CoAP Observe notifications. In particular, a registered device SHOULD also regularly poll the Authorization Server for the most current information about revoked Access Tokens, by sending GET requests to the TRL endpoint according to a related application policy.

# IANA Considerations # {#iana}

This document has the following actions for IANA.

Note to RFC Editor: Please replace all occurrences of "{{&SELF}}" with
the RFC number of this specification and delete this paragraph.

## Media Type Registrations {#iana-media-type}

IANA is asked to register the media type "application/ace-trl+cbor" for messages of the protocols defined in this document encoded in CBOR. This registration follows the procedures specified in {{RFC6838}}.

Type name: application

Subtype name: ace-trl+cbor

Required parameters: N/A

Optional parameters: N/A

Encoding considerations: Must be encoded as CBOR map containing the protocol parameters defined in {{&SELF}}.

Security considerations: See {{sec-security-considerations}} of this document.

Interoperability considerations: N/A

Published specification: {{&SELF}}

Applications that use this media type: The type is used by Authorization Servers, Clients and Resource Servers that support the notification of revoked Access Tokens, according to a Token Revocation List maintained by the Authorization Server as specified in {{&SELF}}.

Fragment identifier considerations: N/A

Additional information: N/A

Person & email address to contact for further information: \<iesg@ietf.org\>

Intended usage: COMMON

Restrictions on usage: None

Author: Marco Tiloca \<marco.tiloca@ri.se>

Change controller: IESG

## CoAP Content-Formats Registry {#iana-content-type}

IANA is asked to add the following entry to the "CoAP Content-Formats" registry within the "CoRE Parameters" registry group.

Media Type: application/ace-trl+cbor

Encoding: -

ID: TBD

Reference: {{&SELF}}

## ACE Token Revocation List Parameters Registry ## {#iana-token-revocation-list}

This specification establishes the "ACE Token Revocation List Parameters" IANA registry. The
registry has been created to use the "Expert Review" registration procedure {{RFC8126}}. Expert Review guidelines are provided in {{review}}. It should be noted that, in addition to the Expert Review, some portions of the registry require a specification, potentially a Standards Track RFC, to be supplied as well.

The columns of this registry are:

* Name: This is a descriptive name that enables easier reference to the item. The name MUST be unique. It is not used in the encoding.

* CBOR Value: This is the value used as CBOR abbreviation of the item. These values MUST be unique. The value can be a positive integer or a negative integer. Different ranges of values use different registration policies {{RFC8126}}. Integer values from -256 to 255 are designated as Standards Action. Integer values from -65536 to -257 and from 256 to 65535 are designated as Specification Required. Integer values greater than 65535 are designated as Expert Review. Integer values less than -65536 are marked as Private Use.

* CBOR Type: This contains the CBOR type of the item, or a pointer to the registry that defines its type, when that depends on another item.

* Reference: This contains a pointer to the public specification for the item.

This registry has been initially populated by the values in {{trl-registry-parameters}}. The "Reference" column for all of these entries refers to this document.

## ACE Token Revocation List Errors {#iana-token-revocation-list-errors}

This specification establishes the "ACE Token Revocation List Errors" IANA registry. The registry has been created to use the "Expert Review" registration procedure {{RFC8126}}. Expert Review guidelines are provided in {{review}}. It should be noted that, in addition to the Expert Review, some portions of the registry require a specification, potentially a Standards Track RFC, to be supplied as well.

The columns of this registry are:

* Value: The value to be used to identify the error. The value MUST be unique. The value can be a positive integer or a negative integer. Integer values between 0 and 255 are designated as Standards Track Document required. Integer values from 256 to 65535 are designated as Specification Required. Integer values greater than 65535 are designated as Expert Review. Integer values less than -65536 are marked as private use.

* Description: This field contains a brief description of the error.

* Reference: This field contains a pointer to the public specification defining the error, if one exists.

This registry has been initially populated by the values in {{error-types}}. The "Reference" column for all of these entries refers to this document.

## Expert Review Instructions {#review}

The IANA registries established in this document is defined as Expert Review. This section gives some general guidelines for what the experts should be looking for, but they are being designated as experts for a reason so they should be given substantial latitude.

Expert reviewers should take into consideration the following points:

* Point squatting should be discouraged. Reviewers are encouraged to get sufficient information for registration requests to ensure that the usage is not going to duplicate one that is already registered and that the point is likely to be used in deployments. The zones tagged as private use are intended for testing purposes and closed environments. Code points in other ranges should not be assigned for testing.

* Specifications are required for the Standards Track range of point assignment. Specifications should exist for Specification Required ranges, but early assignment before a specification is available is considered to be permissible. Specifications are needed for the Expert Review range if they are expected to be used outside of closed environments in an interoperable way. When specifications are not provided, the description provided needs to have sufficient information to identify what the point is being used for.

* Experts should take into account the expected usage of fields when approving point assignment. The fact that there is a range for Standards Track documents does not mean that a Standards Track document cannot have points assigned outside of that range. The length of the encoded value should be weighed against how many code points of that length are left, the size of device it will be used on, and the number of code points left that encode to that size.

--- back

# On using the Series Transfer Pattern # {#sec-series-pattern}

Performing a diff query of the TRL as specified in {{ssec-trl-diff-query}} is in fact a usage example of the Series Transfer Pattern defined in {{I-D.bormann-t2trg-stp}}.

That is, a diff query enables the transfer of a series of TRL updates, with the Authorization Server specifying U <= N_MAX diff entries as the U most recent updates to the portion of the TRL pertaining to a requester, i.e., a registered device or an administrator.

When responding to a diff query request from a requester (see {{ssec-trl-diff-query}}), 'diff_set' is a subset of the update collection associated with the requester, where each 'diff_entry' record is a series item from that update collection. Note that 'diff_set' specifies the whole current update collection when the value of U is equal to SIZE, i.e., the current number of series items in the update collection.

The value N of the query parameter 'diff' in the GET request allows the requester and the Authorization Server to trade the amount of provided information with the latency of the information transfer.

Since the update collection associated with each requester includes up to N_MAX series item, the Authorization Server deletes the oldest series item when a new one is generated and added to the end of the update collection, due to a new TRL update pertaining to that requester (see {{sec-trl-endpoint-supporting-diff-queries}}). This addresses the question "When can the server decide to no longer retain older items?" raised in {{Section 3.2 of I-D.bormann-t2trg-stp}}.

Furthermore, performing a diff query of the TRL together with the "Cursor" extension as specified in {{sec-using-cursor}} in fact relies on the "Cursor" pattern of the Series Transfer Pattern (see {{Section 3.3 of I-D.bormann-t2trg-stp}}).

# Document Updates # {#sec-document-updates}

RFC EDITOR: Please remove this section.

## Version -01 to -02 ## {#sec-01-02}

* Earlier mentioning of error cases.

* Clearer distinction between maintaining the history of TRL updates and preparing the response to a diff query.

* Defined the use of "cursor" in the document body, as an extension of diff queries.

* Both success and error responses have a CBOR map as payload.

* Corner cases of message processing explained more explcitly.

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

The authors sincerely thank {{{Christian Ams??ss}}}, {{{Carsten Bormann}}}, {{{Benjamin Kaduk}}}, {{{David Navarro}}}, {{{Marco Rasori}}}, {{{Michael Richardson}}}, {{{Jim Schaad}}}, {{{G??ran Selander}}} and {{{Travis Spencer}}} for their comments and feedback.

The work on this document has been partly supported by VINNOVA and the Celtic-Next project CRITISEC; and by the H2020 project SIFIS-Home (Grant agreement 952652).
