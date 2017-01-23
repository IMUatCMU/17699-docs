# Requirement Notes

## Keywords

MUST/REQUIRED/SHALL: The definition is an absolute requirement of the specification
MUST NOT/SHALL NOT: The definition is an absolute prohibitation of the specification
SHOULD/RECOMMENDED: May exist valid reasons to ignore item, but full implications must be understood.
SHOULD NOT/NOT RECOMMENDED: May exist valid reasons to accept item, but full implications must be understood.
MAY/OPTIONAL: Optional

## Naming rules

Attribute names must conform to ABNF rules
ATTRNAME   = ALPHA *(nameChar)
nameChar   = "$" / "-" / "_" / DIGIT / ALPHA

Rule is case-insensitive
Character set is US-ASCII

## Attribute Characteristics Default Values

- `required`: false
- `canonicalValues`: none assigned
- `caseExact`: false
- `mutability`: readWrite
- `returned`: "default"
- `uniqueness`: "none"
- `type`: "string"

## Data Value Memo

- Boolean values `true` and `false` are case insensitive
- Decimal values have at least one digit before and after the decimal point.
- Implementations may impose precision ranges on decimal numbers.
- DateTime are in the format of "2008-01-23T04:56:22Z"
- Binary is base64 encoded, case exact and no uniqueness
- Reference is case exact, MAY enforce referential integrity
- `primary` attribute, if absent, shall be assumed `false`
- sub elements should not return items with same `type` & `value` combo
- `null` and empty array should be considered unassigned, unassigned attributes can be ommitted in JSON response

## SCIM Resources

SCIM resources contain the following attributes:

- `Resource Type`: `meta.resourceType`
- `Schemas`: `schemas`, REQUIRED, array of URI, order is unspecified.
- `Common Attributes`: attributes that is present in every resource regardless of the value of `schemas`
- `Core Attributes`: top level attributes along with common attributes.
- `Extended Attributes`

## Extensions

- Schema extensions SHOULD avoid re-defining the attributes in this specification
- Schema extensions SHOULD follow convensions defined in this specification

## Protocol (General)

- SHALL indicate supported HTTP authentication via "WWW-Authenticate" header
- MUST map the authenticated client to access control policy to determine access rights
- SHOULD consider whether the requested action is appropriate for the security subject
- MAY accept anonymous requests, for instance, self registration via create /User
- SCIM uses a media type of `application/scim+json`

## Mutability Rules

- Attributes whose mutability is "readOnly" SHALL be ignored
- Service provider may assign a deafult value to the "readWrite" attributes that are omitted in the body
- Clients may specify `null` or `[]` to clear server defaulted values

## Protocol (Resource Creation)

- Successful resource creation SHALL return `201` as status
- Successful resource creation SHOULD contain resource representation in the body
- `Location` header SHALL reflect `meta.location` value
- In case of resource conflict, return `409` as status with `scimType` errorCode of `uniqueness` (See error)
- When adding resource, the `meta.resourceType` SHALL be set accordingly

## Protocol (Resource Retrieval)

- By default, attributes with `returnability` of `always` or `default` are returned
- If resource exists, reply with `200` as status and resource representation in the body

## Protocol (Resource Query)

- Responses must be identified by URN `urn:ietf:params:scim:api:messages:2.0:ListResponse`
- `totalResults` (REQUIRED)
- `Resources` (REQUIRED if `totalResults` is not-zero)
- `startIndex` (1-based, REQUIRED when partial results are returned due to pagination)
- `itemsPerPage` (REQUIRED when partial results are returned due to pagination)
- Query that does not return any match SHALL return `200` as status and `totalResults` of `0`
- Query can be performed against resource objecct or server root (server root example: `meta.resourceType eq User or meta.resourceType eq Group`).
- If query resulted in too many results (i.e. query against resource directly), server SHALL return `400` with error message of `scimType` set to `tooMany` (See error)
- When query includes more than one resource type, service provider SHALL treat missing attributes as no attribute value (i.e. a presence or equality filter on undefined attribute results in false)

## Protocol (Resource Filter)

- Attribute names and operators are case insensitive
- `pr`: `true` if attribute has non-empty / non-null value. Or contains non-empty node for complex attributes
- Comparison operators like `gt`, `ge`, `lt`, `le` performs lexicographical comparison for strings, chronological comparison for dateTime and numeric comparison for numbers. For boolean and binary attributes, return `400` status with `scimType` of `invalidFilter`
- precedence: grouping > not > and > or > relational (attribute operators)
- for complex attributes, a fully qualified sub-attribute MUST be specified
- Unrecognized operator SHALL result in `400` and `scimType` of `invalidFilter`
- `caseExact` characteristics MUST be determined when comparing string attributes
- `filter=urn:ietf:params:scim:schemas:core:2.0:User:userName sw "J"` is valid
- `filter=schemas eq "urn:ietf:params:scim:schemas:extension:enterprise:2.0:User"` is valid

### Sort

- `sortBy`: if points to multivalued attributes, resources are sorted by the value of the primary attribute, if any, or else the first value in the list. Must point to existing, non-complex attribute.
- `sortOrder`: allowed values are `ascending` and `descending`, defaulted to `ascending`. String type is case insensitive by default unless attribute `caseExact=true`

### Pagination

Requests:

- `startIndex`: 1-based, value less than 1 SHALL be interpreted as 1
- `count`: Non-negative integer, value less than 0 SHALL be interpeted as 0. 0 means no results will be returned except for `totalResults`. Maximum number is set by service provider

Response:

- `itemsPerPage`: Non-negative integer.
- `totalResults`: Non-negative integer.
- `startIndex`: 1-based index of the first result.

### Attributes

- `attributes`: multi-valued list of resource attributes, overriding the default return list
- `excludedAttributes`: multi-valued list of resource attributes not to be returned, SHALL have no effect on attributes whose `returned` is `always`

## Query with HTTP POST

- put `/.search` extension behind the endpoint
- MUST identify with URN `urn:ietf:params:scim:api:messages:2.0:SearchRequest`
- JSON fields to include: `attributes`, `excludedAttributes`, `filter`, `sortBy`, `sortOrder`, `startIndex` and `count`.

## Modify Resource (PUT)

- Client MAY send all attributes on `PUT` request. Service provider will perform attribute-by-attribute replacement
- If `mutability=readWrite` and absent, service provider may clear the value or assign default. Clients that wish to override server default MAY provide `null` and/or `[]`
- If `mutability=immutable`, value must match, otherwise should return `400` status code and `scimType=mutability`. If there is no existing value, new value SHOULD be applied.
- If `mutability=readOnly`, provided values SHALL be ignored.
- If an attribute is required, clients MUST specify in PUT request.
- Successful response returns `200` code and entire resource in body

## Modify Resource (PATCH)

- body of each request MUST contain `schemas` attribute of value `urn:ietf:params:scim:api:messages:2.0:PatchOp`
- body MUST contain `Operations` attribute, which is an array of one or more PATCH operations.
- each PATCH operation MUST have exactly one `op` member, value MAY be one of `add`, `remove`, or `replace`.
- operation MUST be compatible with `mutability` and schema.
- client MAY `add` to `immutable` if there is no previous value
- a PATCH operation that sets a value's `primary` sub-attribute to `true` SHALL cause the server to automatically set `primary` to `false` for any other values in the array.
- PATCH request SHALL be treated as atomic
- return `200` on success

### Add operation

- operation MUST contain `value` member
- if `path` is ommitted, assumed the target is resource itself
- if target is complex, a set of sub-attributes SHALL be specified in `value`
- if target is single-valued, existing value is replaced
- if already contains the same value, nothing SHOULD be done, do not modify update timestamp

### Remove operation

- return `400` with `scimType=noTarget` when `path` is not specified
- if target location is a complex filter matching all elements, attribute should be considered unassigned after removal.
- check schema attribute after operation. If attribute becomes unassigned and it's required or read-only, return `scimType` error.

### Replace operation

- if `path` is ommitted, assumed the target is resource itself
- if attribute does not exist, treat as `add`
- if filter matches more than one, all matches shall be replaced
- if filter results in no target, return `400` with `scimType=noTarget`

## Deleting Resources

- successful delete should return `204`

## Bulk Operations

- requests are identified by URI `urn:ietf:params:scim:api:messages:2.0:BulkRequest`
- responses are identified by URI `urn:ietf:params:scim:api:messages:2.0:BulkResponse`
- successful requests return `200`
- server must continue processing (ignore partial errors), or if `failOnError` is specified, continue processing until error count exceeds
- MUST define maximum operations and payload size limit, return `413` (Payload Too Large) if exceeds

## Misc

- `{urn}:{Attribute name}.{Sub-Attribute name}`
- if does not support `/Me` endpoint, respond with `501` (Not implemented)