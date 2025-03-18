---
title: "HTTP Problem Types for Digest Fields"
category: info

docname: draft-ietf-httpapi-digest-fields-problem-types-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Building Blocks for HTTP APIs"
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: "Building Blocks for HTTP APIs"
  type: "Working Group"
  mail: "httpapi@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/httpapi/"
  github: "ietf-wg-httpapi/digest-fields-problem-types"
  latest: "https://ietf-wg-httpapi.github.io/digest-fields-problem-types/draft-ietf-httpapi-digest-fields-problem-types.html"

author:
 -
    fullname: "Marius Kleidl"
    organization: Transloadit
    email: "marius@transloadit.com"
 -
    fullname: "Lucas Pardue"
    organization: Cloudflare
    email: "lucas@lucaspardue.com"
 -
    ins: R. Polli
    fullname: "Roberto Polli"
    organization: Par-Tec
    email: "robipolli@gmail.com"
    country: Italy

normative:
  DIGEST: RFC9530
  PROBLEM: RFC9457
  STRUCTURED-FIELDS: RFC8941
  HTTP: RFC9110

informative:


--- abstract

This document specifies problem types that servers can use in responses to problems encountered while dealing with a request carrying integrity fields and integrity preference fields.

--- middle

# Introduction

{{DIGEST}} by design does not define, require or recommend any specific behavior for error handling relating to integrity. The responsibility is instead delegated to applications. This draft defines a set of problem types ({{PROBLEM}}) that can be used by server applications to indicate that a problem was encountered while dealing with a request carrying integrity fields and integrity preference fields.

For example, a request message may include content alongside `Content-Digest` and `Repr-Digest` fields that use a digest algorithm the server does not support. An application could decide to reject this request because it cannot validate the integrity. Using a problem type, the server can provide machine-readable error details to aid debugging or error reporting, as shown in the following example.

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json
Want-Content-Digest: sha-512=3, sha-256=10

{
  "type": "https://iana.org/assignments/http-problem-types#digest-unsupported-algorithm",
  "title": "hashing algorithm is not supported",
  "unsupported-algorithm": "foo"
}
~~~

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The terms "integrity fields" and "integrity preference fields" in this document are to be
interpreted as described in {{DIGEST}}.

The term "problem type" in this document is to be
interpreted as described in {{PROBLEM}}.

The term "request", "response", "intermediary", "sender", and "server" are from {{HTTP}}.

# Problem Types

## Unsupported Hashing Algorithm

This section defines the "https://iana.org/assignments/http-problem-types#digest-unsupported-algorithm" problem type.
A server MAY use this problem type if it wants to communicate to the client that
one of the hashing algorithms referenced in the integrity or integrity preference fields present in the request
is not supported.

For this problem type, `unsupported-algorithm` is defined as the only extension member.
It SHOULD be populated in a response using this problem type, with its value being the algorithm key of the unsupported algorithm from the request.
The response can include the corresponding integrity preference field to indicate the server's algorithm support and preference.

Example:

~~~ http-message
POST /books HTTP/1.1
Host: foo.example
Content-Type: application/json
Accept: application/json
Accept-Encoding: identity
Repr-Digest: sha-256=:mEkdbO7Srd9LIOegftO0aBX+VPTVz7/CSHes2Z27gc4=:

{"title": "New Title"}
~~~
{: title="A request with a sha-256 integrity field, which is not supported by the server"}

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json
Want-Repr-Digest: sha-512=10, sha-256=0

{
  "type": "https://iana.org/assignments/http-problem-types#digest-unsupported-algorithm",
  "title": "Unsupported hashing algorithm",
  "unsupported-algorithm": "sha-256"
}
~~~
{: title="Response Advertising the Supported Algorithms"}


This problem type is a hint to the client about algorithm support, which the client could use to retry the request with a different, supported, algorithm.
Note that a request may contain more than one integrity field.

This problem type can also be used when a request contains an integrity preference field. For example:

~~~ http-message
GET /items/123 HTTP/1.1
Host: foo.example
Want-Repr-Digest: sha=10

~~~
{: title="GET Request with Want-Repr-Digest"}

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://iana.org/assignments/http-problem-types#digest-unsupported-algorithm",
  "title": "Unsupported hashing algorithm",
  "unsupported-algorithm": "sha"
}
~~~
{: title="Response Advertising the Supported Algorithms"}

## Invalid Digest Value

This section defines the "https://iana.org/assignments/http-problem-types#digest-invalid-value" problem type. A server MAY use this problem type when responding to a request, whose integrity fields include a digest value, that cannot be generated by the corresponding hashing algorithm. For example, if the digest value of the `sha-512` hashing algorithm is not 64 bytes long, it cannot be a valid digest value and the server can skip computing the digest value. This problem type MUST NOT be used if the server is not able to parse the integrity fields according to {{Section 4.5 of STRUCTURED-FIELDS}}, for example because of a syntax error in the field value.

The server SHOULD include a human-readable description why the value is considered invalid in the `title` member.

The following example shows a request with the content `{"hello": "world"}` (plus LF), but the digest has been truncated. The subsequent response indicates the invalid SHA-512 digest.

~~~ http-message
PUT /items/123 HTTP/1.1
Host: foo.example
Content-Type: application/json
Repr-Digest: sha-512=:YMAam51Jz/jOATT6/zvHrLVgOYTGFy1d6GJiOHTohq4:

{"hello": "world"}
~~~
{: title="A request with a sha-512 integrity field, whose digest has been truncated"}

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://iana.org/assignments/http-problem-types#digest-invalid-value",
  "title": "digest value for sha-512 is not 64 bytes long"
}
~~~
{: title="Response indicating that the provided digest is too short"}

This problem type indicates a fault in the sender's calculation or encoding of the digest value. A retry of the same request without modification will likely not yield a successful response.

## Mismatching Digest Value

This section defines the "https://iana.org/assignments/http-problem-types#digest-mismatching-value" problem type. A server MAY use this problem type when responding to a request, whose integrity fields include a digest value that does not match the digest value that the server calculated for the request content or representation.

Two problem type extension members are defined: the `algorithm` and `provided-digest` members. A response using this problem type SHOULD populate all members, with the value of `algorithm` being the algorithm key of the used hashing algorithm and the value of `provided-digest` being the digest value taken from the request's integrity fields. The digest value MUST be serialized as byte sequences as described in {{Section 4.1.8 of STRUCTURED-FIELDS}}.

The following example shows a request with the content `{"hello": "woXYZ"}` (plus LF), but the representation digest for `{"hello": "world"}` (plus LF). The subsequent response indicates the mismatching SHA-256 digest values.

~~~ http-message
PUT /items/123 HTTP/1.1
Host: foo.example
Content-Type: application/json
Repr-Digest: sha-256=:RK/0qy18MlBSVnWgjwz6lZEWjP/lF5HF9bvEF8FabDg=:

{"hello": "woXYZ"}
~~~
{: title="A request with a sha-256 integrity field, which does not match the representation"}

~~~ http-message
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://iana.org/assignments/http-problem-types#digest-mismatching-value",
  "title": "digest value from request does not match expected value",
  "algorithm": "sha-256",
  "provided-digest": ":RK/0qy18MlBSVnWgjwz6lZEWjP/lF5HF9bvEF8FabDg=:"
}
~~~
{: title="Response indicating the mismatching digests"}

If the sender receives this problem type, the request might be modified unintentionally by an intermediary. The sender could use this information to retry the request without modification to address temporary transmission issues.

# Security Considerations

Disclosing error details could leak information
such as the presence of intermediaries or the server's implementation details.
Moreover, they can be used to fingerprint the server.

To mitigate these risks, a server could assess the risk of disclosing error details
and prefer a general problem type over a more specific one.

# IANA Considerations

IANA is asked to register the following entries in the "HTTP Problem Types" registry at <https://www.iana.org/assignments/http-problem-types>.

## Registration of "digest-unsupported-algorithm" Problem Type

Type URI:
: https://iana.org/assignments/http-problem-types#digest-unsupported-algorithm

Title:
: Unsupported Hashing Algorithm

Recommended HTTP status code:
: 400

Reference:
: {{unsupported-hashing-algorithm}} of this document

## Registration of "digest-invalid-value" Problem Type

Type URI:
: https://iana.org/assignments/http-problem-types#digest-invalid-value

Title:
: Invalid Digest Value

Recommended HTTP status code:
: 400

Reference:
: {{invalid-digest-value}} of this document

## Registration of "digest-mismatching-value" Problem Type

Type URI:
: https://iana.org/assignments/http-problem-types#digest-mismatching-value

Title:
: Mismatching Digest Value

Recommended HTTP status code:
: 400

Reference:
: {{mismatching-digest-value}} of this document

--- back
