# HUSST Tickethub API
## Overview

The HUSST Tickethub API provides secure management of tickets associated with transit tokens. It enables transit operators and partners to:

- **Store and update tickets** linked to transit tokens
- **Retrieve ticket information** for inspection and validation
- **Manage the ticket lifecycle** including terminating, deleting, or reassigning tickets

## Transit Token

The API supports different transit token schemes. Each transit token is always handled together with an explicit `tokenType`.

The default and currently supported token type is `iso24851`. Support for additional token types must be requested and approved via this repository.

A transit token is represented as a **typed transit token**, consisting of:
- `tokenType`: Identifies the token scheme
- `transitToken`: The token value encoded as Base64url

### Matching rules

Ticket lookup and validation apply token-type-specific matching rules.

For `iso24851`, matching rules depend on the tokenization method defined by the ISO 24851 standard. These rules are applied internally by the Tickethub when resolving tickets for a given transit token.


## Ticket

A ticket is a composite of structured data that is available for interpretation for the ticket hub and a ticket container which is not interpreted by the ticket hub. It allows to filter the tickets for relevance when an inspection requests tickets for inspection.
The ticket container's format can be freely chosen by the different parties (CCP, SO).

### Structured data

Ticket data is separated conceptually into **write-time data** (ticket creation) and **read-time data** (retrieved tickets).

### Ticket creation (write)

When creating a ticket, the client must provide:
- `status`
- `expirationTime`
- `entitlement` (container data)

If not specified explicitly:
- `effectiveTime` defaults to the creation time

### Retrieved tickets (read)

When tickets are retrieved from the API, the structured data additionally contains:
- `ticketId` (including `ccpOrgId`)
- `status`
- `effectiveTime`
- `expirationTime`
- `entitlement` (container data)

All fields except `status` are immutable.

### Container

The container is the part of the ticket that allows the CCP to store entitlement data in any previously agreed format.
It may contain information such as spatial validity or complex temporal rules that are not interpreted by the Tickethub itself.

The container format SHOULD be indicated using the `contentType` field.
It is recommended to use well-known ticket formats to ensure compatibility with inspection devices.

## API Endpoints

The API endpoints provide access to the ticket storage, which is essentially a highly available database augmented by a lightweight, versatile, domain-specific layer that models tickets and tokens and enforces client-specific read and write permissions.

### Tickets

| Method | Endpoint | Operation | Required Scope |
|--------|----------|-----------|----------------|
| `POST` | `/tickets` | Create new ticket | `create:ticket` |
| `GET` | `/tickets` | Retrieve tickets for a transit token | `view:token` or `validate:token` |
| `PATCH` | `/tickets/{ticketRef}` | Update a single ticket | `update:ticket` |
| `DELETE` | `/tickets/{ticketRef}` | Delete a single ticket | `delete:ticket` |

### Tokens

| Method | Endpoint | Operation | Required Scope |
|--------|----------|-----------|----------------|
| `POST` | `/tokens` | Replace transit token | `replace:token` |


### General Authentication Flow

This flow illustrates the authentication and authorization process that precedes all API operations. If either step fails, the corresponding error response is returned immediately. The endpoint-specific flows below assume successful authentication.

```mermaid
sequenceDiagram
    autonumber
    participant Client as CCP/SO
    participant Tickethub as Ticket hub

    Client->>+Tickethub: API Request [Bearer Token]

    Tickethub->>Tickethub: Validate JWT signature
    Tickethub->>Tickethub: If mTLS present, verify certificate binding

    alt Authentication failed
        Tickethub-->>Client: 401 Unauthorized
    else Authentication succeeded
        Tickethub->>Tickethub: Check scope for requested operation

        alt Authorization failed
            Tickethub-->>Client: 403 Forbidden
        else Authorization succeeded
            note over Client,Tickethub: Processing continues, see below
            Tickethub-->>Client: Return message
        end
    end

```

## Data Models

### Ticket

```json
{
  "ticketId": {
    "ccpOrgId": 35000,
    "ticketRef": "987654321"
  },
  "depositedAt": "2025-12-01T10:00:00Z",
  "effectiveTime": "2025-12-08T12:00:00Z",
  "expirationTime": "2025-12-31T23:59:59Z",
  "status": "ok",
  "entitlement": {
    "contentType": "etiCore",
    "content": "d2F0Y2g/dj14dkZaam81UGdHMA=="
  }
}
```

### Entitlement Content Types

| Type | Description |
|------|-------------|
| `KA` | VDV-KA format |
| `etiCore` | etiCore format |
| `UIC` | UIC format |
| `unspecified` | Unspecified format (default) |

### OrgId

Organization IDs range from `0` to `2147483647`.

## Response Codes

| Code | Description |
|------|-------------|
| `200` | Success (with response body) |
| `204` | Success (no content - operation successful) |
| `400` | Bad Request - Invalid input data |
| `401` | Unauthorized - Authentication failed |
| `403` | Forbidden - Authorization failed (e.g., OrgId mismatch) |
| `404` | Not Found - Resource does not exist |
| `500` | Internal Server Error |


## Error Handling

All error responses follow RFC 7807 (Problem Details):

```json
{
  "type": "https://api.example.com/problems/invalid-request",
  "title": "Invalid request",
  "status": 400,
  "detail": "Descriptive error message",
  "instance": "/tickets"
}
```

### Common Error Scenarios

| Scenario | Status | Description/Example |
|----------|--------|---------------------|
| Missing auth token | 401 | Authentication required. |
| Invalid OrgId | 400 | Invalid OrgId. |
| Certificate mismatch | 403 | Insufficient permissions: OrgId does not match certificate. |
| Missing scope | 403 | Insufficient permissions for the requested operation. |
| Ticket not found | 404 | Ticket reference not found. |


## Encoding

### Base64url

Transit tokens use URL-safe Base64 encoding as defined in [RFC 4648 Section 5](https://datatracker.ietf.org/doc/html/rfc4648#section-5):
- Character set: `A-Z`, `a-z`, `0-9`, `-`, `_`
- No padding characters (`=`)
