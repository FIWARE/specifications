# NGSI v2 Context Source Forwarding Specification

## Editor

José Manuel Cantera Fonseca, FIWARE Foundation e.V.

## Contributors

Jason Fox, FIWARE Foundation e.V. 

## Copyright

Copyright (c) 2018, FIWARE Foundation e.V.

## License

This specification is licensed under the
[FIWARE Open Specification License (implicit patent license)](https://forge.fiware.org/plugins/mediawiki/wiki/fiware/index.php/Implicit_Patents_License).

## Normative References

+ [FIWARE NGSIv2](http://fiware.github.io/specifications/ngsiv2/latest/)

## Introduction

Distributed Context Information Management architectures revolve around multiple sources of Context Information, Context Sources (usually known as Context Providers in the
traditional FIWARE NGSI jargon), 
which are capable of providing specific Attributes, or even all content of particular Entity Types, according to different scoping criteria such as location or time.

A Context Source is usually
[registered](https://swagger.lab.fiware.org/?url=https://raw.githubusercontent.com/Fiware/specifications/master/OpenAPI/ngsiv2/ngsiv2-openapi.json#/Registrations/Create_Registration)
with an NGSI Registry so that it can announce what kind of information it can
provide, when requested, to Context Consumers and Brokers.

When registering a new NGSI Context Source it can be specified whether a (capable) Broker shall forward operations to such Context Source when needed, i.e. when
requests involving the Context Source-provided information are ongoing, so that
all information from different Context Sources is mashed-up, hidding information distribution complexities to the caller.

This specification is intended to define

* the behaviour of NGSI Brokers capable of forwarding requests to NGSI Context Sources.
* the interface (specified as an HTTP binding) that shall be supported by NGSI Context Sources.

This specification defines forwarding within
[![NGSI v2](https://img.shields.io/badge/NGSI-v2-blue.svg)](http://fiware.github.io/specifications/ngsiv2/latest/)


## Definitions

An NGSI Context Source is an NGSI endpoint supporting the NGSI operations mandated by this specification.

An NGSI Forwarder is an NGSI Broker (NGSI implementation) capable of forwarding Context Information Management operations to matching Context Sources.

A matching Context Source is a registered Context Source capable of providing Context Information in the scope of a particular NGSI operation. 

A Forwarded NGSI Context Source is the target of an NGSI Forwarding operation (request) issued by an NGSI Forwarder. 

An NGSI Forwarding operation (request) is an NGSI (Context Information Management) operation (request) issued by an NGSI Forwarder (against a Forwarded NGSI Context Source).

## Context Source Interface

Compliant Context Sources shall support the interface (HTTP binding) defined by the operations described below. 

### Query Entity(ies)

It allows to obtain Context Information represented as Entities and Attributes. 

`GET ${endPoint}/entities/`

For [![NGSI v2](https://img.shields.io/badge/NGSI-v2-blue.svg)](https://swagger.lab.fiware.org/?url=https://raw.githubusercontent.com/Fiware/specifications/master/OpenAPI/ngsiv2/ngsiv2-openapi.json):

-  ${endPoint} shall be equal to `/v2`
-  It shall support the same HTTP interface as described by the
[NGSIv2 Specification](https://swagger.lab.fiware.org/?url=https://raw.githubusercontent.com/Fiware/specifications/master/OpenAPI/ngsiv2/ngsiv2-openapi.json#/Entities/List_Entities)

If the Context Source is not capable of resolving a filter query or a geoquery
then it shall result in an HTTP `501` (`Not implemented`) error code. 

### Update Entity(ies)

It allows to update Context Information, Entity's Attributes, with the following restrictions:

-  It shall not be allowed the mutation of the structure of Entities (adding or removing attributes) through Context Forwarding operations. 
-  It shall not be allowed updating GeoProperties (`geo:json` in NGSIv2) through Context Forwarding operations.

The assumption is that those operations would eventually be executed directly against the original Context Source and not through Context Forwarding.

-  Any prohibited action shall result in an HTTP `422` error code. Error description shall include the string `Operation not Supported`. 

For [![NGSI v2](https://img.shields.io/badge/NGSI-v2-blue.svg)](https://swagger.lab.fiware.org/?url=https://raw.githubusercontent.com/Fiware/specifications/master/OpenAPI/ngsiv2/ngsiv2-openapi.json):

`POST /${endPoint}/op/update`

-  ${endPoint} shall be equal to `/v2`
-  It shall support the same interface as described by
[NGSIv2 Specification](https://swagger.lab.fiware.org/?url=https://raw.githubusercontent.com/Fiware/specifications/master/OpenAPI/ngsiv2/ngsiv2-openapi.json#/Batch_Operations/Update)
but only the `UPDATE` action type shall be accepted.

### Detecting Forwarding NGSI operations

NGSI Context Source endpoint implementations shall detect when the origin of NGSI operations are NGSI Forwarders. To that aim,
they shall check the content of the HTTP [Via](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Via) header, as mandated by this specification.

NGSI Context Source endpoint implementations shall use the HTTP `Via` Header
to detect possible loops while processing incoming operations. If a loop is detected they should return an `508` `Loop detected` HTTP error code. 

### Other NGSI operations

Any other **Forwarding** NGSI Operation (i.e. generated by a Context Forwarder) issued against a Context Source interface shall result in an HTTP `422` error code.
Error description shall include the string `Operation not Supported`. 

## Forwarding behaviour

### General Behaviour

For each Context Source endpoint the logic implemented by an NGSI Forwarder should group (aggregate) as many Forwarding Operations as possible i.e. minimizing the number of HTTP requests against Forwarded Context Sources.

For each Context Source endpoint an NGSI Forwarder shall establish a timeout period that once elapsed it will result in a communication error with the Forwarded Context Source (see below). 

The HTTP headers present in the original request shall be preserved when issuing requests to the Forwarded Context Sources.

The only NGSI operations that are subject to be forwarded are:

-  Query/List, Get Entity Data operations (involving one or multiple Entities)
-  Entity's Attribute update operations. 

The rest of NGSI operations are not suitable to be forwarded and Brokers,
instead they shall respond with an HTTP `422` error. Error description shall include the string `Operation not Supported`. 

### Maximum Number of Context Forwarding Requests

When it comes to distributed Context Sources, NGSI client applications can hint Brokers when resolving an NGSI request. For doing so, they can use the **`Fiware-Csource-Max-Forwards`** HTTP header. The value of such header shall be a positive number, including `0`. Using such header, it can be limited the total number of Context Forwarding requests performed when resolving an NGSI operation. This header, as any other, shall be propagated while resolving an NGSI Forwarding Operation, so that Forwarded Context sources can also apply this policy individually. 

Implementations should have a default maximum limit for Context Forwarding requests so that resources are not exhausted. When resolving an NGSI operation, if not all the matching Context Sources were contacted due to these limits, the HTTP response shall flag this situation to the client, by using an appropriate HTTP response status code. See the specification below.  

### Tracking Context Forwarding

In order to allow tracking and/or debugging the forwarding chain, implementations shall use the
**`Fiware-Csource-Response`** HTTP response header. There shall be as many fields values (comma separated) for `Fiware-Csource-Response` as the number of involved Context Sources that were contacted, when processing the original request. Each field-value is a semicolon-separated list; this sublist consists of parameter-identifier pairs. Parameter-identifier pairs are grouped together by an equals sign. The parameters defined are:

-  `id`: the id of the matching Context Source registration. Ex. `id=urn:ngsi-ld:ContextSourceRegistration:A4567`. Mandatory
-  `endpoint`: For instance when debugging activated, it can include the URI of the endpoint associated to the Forwarded Context Source endpoint. Ex. `host=mycsource.example.org`. Optional. 
-  `status`: The HTTP status code resulting from the Forwarding request made. If the endpoint was reachable. Optional. 
-  `description`: Any human-readable string that can help to understand what happened. Optional. 

Implementations shall consider all the trade-offs (security vs development friendliness) when disclosing information through the `Fiware-Csource-Response` header.

The forwarding component (usually a Broker) shall add to the outbound forwarding request the
`Via` HTTP header, so that Forwarded Context Sources can be aware that a Context Forwarding operation is ongoing. 
The content of the **`Via`** HTTP header is described below:

-  The received protocol shall be `ngsi/v2` for NGSIv2 and `ngsi/ld` for NGSI-LD.
-  The received by could be just a pseudonym for the originator of the request or could include the endpoint's URI. In order to detect possible loops, if a pseudonym is used then it shall be easily recognizable during subsequent requests. 

### Matching Context Sources

A matching Context Source is a Context Source which shall meet all of the following criteria:

For query operations:

-  If it involves an Entity Type, the Context Source is registered as provider of information for such Entity Type.
-  If it involves an Entity Id the Context Source is registered as provider of information for such Entity Id
-  If it involves an `idPattern`, the Context Source is registered as provider of information for entities which id matches an `idPattern` that denotes an intersecting L(R) with the query's `idPattern`.
-  If it involves Entity Attributes, the Context Source is registered as provider of information for one of the Attributes involved. 
-  If it involves a geoquery, the Context Source is registered as capable of providing data in the geographical scope determined by such geoquery.

For update operations:

-  If it involves an Entity Type, the Context Source is registered as provider of information for such Entity Type.
-  If it involves an Entity Id, the Context Source is registered as provider of information for such Entity Id
-  If it involves Entity Attributes, the Context Source is registered as provider of information for one of the Attributes involved. 

### Forwarding Scenarios

When resolving NGSI operations implementations shall deal with three different scenarios:

-  *Scenario 1*. There are no registered Context Sources matching the concerned Entities, Attributes or scopes.
-  *Scenario 2*. There is no local information about the concerned Entities but there are matching Context Sources.
-  *Scenario 3*. There are both local information and matching Context Sources.

*Scenario 1* shall be resolved as any other NGSI operation, using the locally stored information.

*Scenario 2* shall be resolved by forwarding the corresponding operation to each of the matching, registered Context Sources (`Fiware-Csource-Max-Forwards` permitting), provided that the concerned operation
is a query-related operation or an Entity Attribute Update operation (see above). If the concerned operation is not allowed to be forwarded, a response with HTTP error `422` shall be generated. 

For query-related operations, the implementation shall take care of mashing-up all the the information provided by the matching Context Sources. If more than one matching Context Source is providing the information same Entity attribute, the resulting Entity content returned can be implementation-dependant, as in principle there is no "priority" between the different Context Sources. 

For update operations matching Context Sources shall be contacted forwarding to them the corresponding update operations over Entities. 

*Scenario 3* shall be always resolved locally, firstly. After resolving the operation locally, if and only if there are outstanding information elements (i.e. not present locally) that can be potentially provided by
some of the matching Context Sources, request forwarding shall proceed. Only those matching Context Sources capable of providing attributes not available locally shall be contacted. 

For query operations, after getting a response from the Forwarded Context Sources, only the Entity Attributes not present locally shall be mashed-up, rest of them shall be ignored, as local data acts always as a shadow for remote, Context Source provided data. 

For update operations, only the updates that cannot be performed locally, shall be forwarded to the matching Context Sources. 

### HTTP Response Codes

If all the matching Context Sources responded with an HTTP status code `2xx` (Successful) i.e. the request was successfully received, understood, and accepted, then a final HTTP response with `2xx` status code shall be generated. 

If all matching Context Sources responded with an HTTP status code in the ranges `4xx` or `5xx` or were not reachable at all, then an HTTP response with status code `502` `Bad Gateway` shall be generated. If all matching Context Sources raised timeout conditions, then an HTTP response with status code `503` `Gateway Timeout` shall be generated.

If some matching Context Sources responded with a HTTP code in the ranges `4xx` or `5xx`, they were not reachable (Connection Errors, timeouts, etc.), or they were not contacted due to Context Forwarding request limits (posed by `Fiware-Csource-Max-Forwards` or the implementation itself), then the HTTP response code shall be `207` (`Partial Success`). Clients should inspect the content of the `Fiware-Csource-Response` header to understand the real motivation of the partial success. 

### Forwarding query-related operations

For [![NGSI v2](https://img.shields.io/badge/NGSI-v2-blue.svg)](https://swagger.lab.fiware.org/?url=https://raw.githubusercontent.com/Fiware/specifications/master/OpenAPI/ngsiv2/ngsiv2-openapi.json) the following table outlines how NGSI query-related operations should be forwarded to Context Sources. 

A [Query Entities Operation](https://swagger.lab.fiware.org/?url=https://raw.githubusercontent.com/Fiware/specifications/master/OpenAPI/ngsiv2/ngsiv2-openapi.json#/Entities/List_Entities) shall be forwarded to a Context Source as a Query Entities GET request (adapting it
to the attributes / scope that the Context Source announced it provided information for). 

Individual Entity query-related operations shall be forwarded to a Context Source as follows (adapting it to the 
attributes that the Context Source announce it provided information for): 

```text
GET ${endPoint}/entities/{entityId}?attrs={attrList}  --> ${endPoint}/entities/?id={entityId}&attrs={attrList}
GET ${endPoint}/entities/{entityId}/attrs/  -->  ${endPoint}/entities/?id={entityId} 
GET ${endPoint}/entities/{entityId}/attrs/{attrId}/value  --> ${endPoint}/entities/?id={entityId}&attrs={attrId}
```

### Forwarding Entity Attribute update operations

For [![NGSI v2](https://img.shields.io/badge/NGSI-v2-blue.svg)](https://swagger.lab.fiware.org/?url=https://raw.githubusercontent.com/Fiware/specifications/master/OpenAPI/ngsiv2/ngsiv2-openapi.json):

```text
PATCH ${endPoint}/entities/{entityId}/attrs/ 
PUT ${endPoint}/entities/{entityId}/attrs/{attrName}
PUT ${endPoint}/entities/{entityId}/attrs/{attrName}/value
```

shall be mapped to

```POST /v2/op/update``` with a payload with `actionType` equals to `UPDATE` and an array of affected entities which 
shall include an entity with id equals to {entityId} and all the attributes that are to be modified. 

Other update operations shall not be forwarded and in case they can only be resolved remotely they shall result in
an HTTP response with status code `422`. 

## Examples

### Example 1. Forwarding a query operation

* Registered Context Source: `urn:ngsi-ld:ContextSourceRegistration:A23456` for Entity Type: `Room`. 

![NGSI v2](https://img.shields.io/badge/NGSI-v2-blue.svg) original operation:

```text
GET /v2/entities/?type=Room&q=capacity>20
```

![NGSI v2](https://img.shields.io/badge/NGSI-v2-blue.svg) Forwarded Request:

```text
GET /v2/entities/?type=Room&q=capacity>20
Via: ngsi/v2 mycity.example.org:1026
```

![NGSI v2](https://img.shields.io/badge/NGSI-v2-blue.svg) Forwarded Response:

```
200 OK
Content-Type: application/json
Content-Length: 52
```text
```json
[
   {
      "id": "urn:ngsi-ld:Room:B234",
      "type": "Room",
      "capacity": {
        "value": 34
      }
   }
]
```

![NGSI v2](https://img.shields.io/badge/NGSI-v2-blue.svg) Final Response:

```text
200 OK
Content-Type: application/json
Content-Length: 52
Fiware-Csource-Response: id=urn:ngsi-ld:ContextSourceRegistration:A23456; status=200; 
```
```json
[
   {
      "id": "urn:ngsi-ld:Room:B234",
      "type": "Room",
      "capacity": {
        "value": 34
      }
   }
]
```

### Example 2. Forwarding an update operation

![NGSI v2](https://img.shields.io/badge/NGSI-v2-blue.svg) original operation:

```text
PATCH /v2/entities/urn:ngsi-ld:Room:A234/attrs/
Content-Type: application/json
Content-Length: 35
```

```json
{
   "capacity": {
     "value": 34
   }
}
```

![NGSI v2](https://img.shields.io/badge/NGSI-v2-blue.svg) Forwarded Request:

```text
POST /v2/op/update
Content-Type: application/json
Content-Length: 42
Via: ngsi/v2 mycity.example.org:1026
```

```json
{
  "actionType": "UPDATE",
  "entities": [{
     "id": "urn:ngsi-ld:Room:A234",
     "capacity": {
       "value": 34
     }
  }]
}
```

```text
204 No Content
```

![NGSI v2](https://img.shields.io/badge/NGSI-v2-blue.svg) Final Response:

```
204 No Content
Fiware-Csource-Response: id=urn:ngsi-ld:ContextSourceRegistration:A23456; status=200; 
```
 
