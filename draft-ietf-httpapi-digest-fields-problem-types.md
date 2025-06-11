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
  STRUCTURED-FIELDS: RFC9651
  HTTP: RFC9110
  RFC8792:
  JSON: RFC8259

informative:


--- abstract

This document specifies problem types that servers can use in responses to problems encountered while dealing with a request carrying integrity fields and integrity preference fields.

--- middle

# Introduction

{{DIGEST}} defines HTTP fields for exchanging integrity digests and preferences, but does not specify, require or recommend any specific behavior for error handling relating to integrity by design. The responsibility is instead delegated to applications. This draft defines a set of problem types ({{PROBLEM}}) that can be used by server applications to indicate that a problem was encountered while dealing with a request carrying integrity fields and integrity preference fields.

For example, a request message may include content alongside `Content-Digest` and `Repr-Digest` fields that use a digest algorithm the server does not support. An application could decide to reject this request because it cannot validate the integrity. Using a problem type, the server can provide machine-readable error details to aid debugging or error reporting, as shown in the following example.

~~~ http-message
# NOTE: '\' line wrapping per RFC 8792

HTTP/1.1 400 Bad Request
Content-Type: application/problem+json
Want-Content-Digest: sha-512=3, sha-256=10

{
  "type": "https://iana.org/assignments/http-problem-types#\
    digest-unsupported-algorithms",
  "title": "Unsupported hashing algorithms",
  "unsupported-algorithms": [
    {
      "algorithm": "foo",
      "header": "Want-Content-Digest"
    }
  ]
}
~~~

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Some examples in this document contain long lines that may be folded, as described in {{RFC8792}}.

The terms "integrity fields" and "integrity preference fields" in this document are to be
interpreted as described in {{DIGEST}}.

The term "problem type" in this document is to be
interpreted as described in {{PROBLEM}}.

The terms "request", "response", "intermediary", "sender", "client", and "server" are from {{HTTP}}.

The problem types in this document are defined using JSON {{JSON}}. They can be serialized into an equivalent XML format as outlined in {{Appendix B of PROBLEM}}.

# Problem Types

The following section defines three problem types to express common problems that occur when handling integrity or integrity preference fields on the server. These problem types use the `digest-` prefix in their type URI. Other problem types that are defined outside this document, yet specific to digest related problems, may reuse this prefix.

Requests can include multiple integrity or integrity preference fields. For example, they may use the `Content-Digest` and `Repr-Digest` fields simultaneously or express preferences for content and representation digests at the same time. Therefore, similar problems can appear multiple times for one request. The problem types defined in this document allow expressing multiple appearances, while each time identifying the corresponding header that contained the problematic value.

## Unsupported Hashing Algorithms

This section defines the "https://iana.org/assignments/http-problem-types#digest-unsupported-algorithms" problem type.
A server can use this problem type to communicate to the client that
one or more of the hashing algorithms referenced in the integrity or integrity preference fields present in the request
are not supported.

For this problem type, the `unsupported-algorithms` extension member is defined, whose value is a JSON {{JSON}} array of entries identifying each unsupported algorithm.
Each entry in the array is a JSON object with the following members:

- The `algorithm` member is a JSON string containing the algorithm key of the unsupported algorithm.
- The `header` member is a JSON string containing the name of the integrity or integrity preference field that referenced the unsupported algorithm.

The response can include the corresponding integrity preference field to indicate the server's algorithm support and preference.

This problem type is a hint to the client about algorithm support, which the client could use to retry the request with different, supported, algorithms.

Example:

~~~ http-message
POST /books HTTP/1.1
Host: foo.example
Content-Type: application/json
Accept: application/json
Accept-Encoding: identity
Repr-Digest: sha-256=:mEkdbO7Srd9LIOegftO0aBX+VPTVz7/CSHes2Z27gc4=:
Content-Digest: sha-256=:mEkdbO7Srd9LIOegftO0aBX+VPTVz7/CSHes2Z27gc4=:

{"title": "New Title"}
~~~
{: title="A request with sha-256 integrity fields, which are not supported by the server"}

~~~ http-message
# NOTE: '\' line wrapping per RFC 8792

HTTP/1.1 400 Bad Request
Content-Type: application/problem+json
Want-Repr-Digest: sha-512=10, sha-256=0
Want-Content-Digest: sha-512=10, sha-256=0

{
  "type": "https://iana.org/assignments/http-problem-types#\
    digest-unsupported-algorithms",
  "title": "Unsupported hashing algorithms",
  "unsupported-algorithms": [
    {
      "algorithm": "sha-256",
      "header": "Repr-Digest"
    },
    {
      "algorithm": "sha-256",
      "header": "Content-Digest"
    }
  ]
}
~~~
{: title="Response indicating the problem and advertising the supported algorithms"}

This problem type can also be used when a request contains an integrity preference field with an unsupported algorithm. For example:

~~~ http-message
GET /items/123 HTTP/1.1
Host: foo.example
Want-Repr-Digest: sha=10

~~~
{: title="A request with a sha-256 integrity preference field, which is not supported by the server"}

~~~ http-message
# NOTE: '\' line wrapping per RFC 8792

HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://iana.org/assignments/http-problem-types#\
    digest-unsupported-algorithms",
  "title": "Unsupported hashing algorithms",
  "unsupported-algorithms": [
    {
      "algorithm": "sha",
      "header": "Want-Repr-Digest"
    }
  ]
}
~~~
{: title="Response indicating the problem and advertising the supported algorithms"}

## Invalid Digest Values

This section defines the "https://iana.org/assignments/http-problem-types#digest-invalid-values" problem type. A server can use this problem type when responding to a request, whose integrity fields include a digest value, that cannot be generated by the corresponding hashing algorithm. For example, if the digest value of the `sha-512` hashing algorithm is not 64 bytes long, it cannot be a valid SHA-512 digest value and the server can skip computing the digest value. This problem type MUST NOT be used if the server is not able to parse the integrity fields according to {{Section 4.5 of STRUCTURED-FIELDS}}, for example because of a syntax error in the field value.

For this problem type, the `invalid-digests` extension member is defined, whose value is a JSON {{JSON}} array of entries identifying each invalid digest.
Each entry in the array is a JSON object with the following members:

- The `algorithm` member is a JSON string containing the algorithm key.
- The `header` member is a JSON string containing the name of the integrity field that contained the invalid digest value.
- The `reason` member is a JSON string containing a human-readable description why the value is considered invalid.

This problem type indicates a fault in the sender's calculation or encoding of the digest value. A retry of the same request without modification will likely not yield a successful response.

The following example shows a request with the content `{"hello": "world"}` (plus LF), but the digest has been truncated. The subsequent response indicates the invalid SHA-512 digest.

~~~ http-message
PUT /items/123 HTTP/1.1
Host: foo.example
Content-Type: application/json
Repr-Digest: sha-512=:YMAam51Jz/jOATT6/zvHrLVgOYTGFy1d6GJiOHTohq4:

{"hello": "world"}
~~~
{: title="A request with a sha-512 integrity field, whose digest has been truncated to 32 bytes"}

~~~ http-message
# NOTE: '\' line wrapping per RFC 8792

HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://iana.org/assignments/http-problem-types#\
    digest-invalid-values",
  "title": "Invalid digest values",
  "invalid-digests": [
    {
      "algorithm": "sha-512",
      "header": "Repr-Digest",
      "reason": "digest value is not 64 bytes long"
    }
  ]
}
~~~
{: title="Response indicating that the provided digest is too short"}

## Mismatching Digest Values

This section defines the "https://iana.org/assignments/http-problem-types#digest-mismatching-values" problem type. A server can use this problem type when responding to a request, whose integrity fields include a digest value that does not match the digest value that the server calculated for the request content or representation.

For this problem type, the `mismatching-digests` extension member is defined, whose value is a JSON {{JSON}} array of entries identifying each mismatching digest.
Each entry in the array is a JSON object with the following members:

- The `algorithm` member is a JSON string containing the algorithm key of the hashing algorithm.
- The `provided-digest` member is a JSON string containing the digest value taken from the request's integrity fields. The digest value is serialized as a byte sequence as described in {{Section 4.1.8 of STRUCTURED-FIELDS}}.
- The `header` member is a JSON string containing the name of the integrity field that contained the mismatching digest value.

The problem type intentionally does not include the digest value calculated by the server to avoid attackers abusing this information for oracle attacks.

If the sender receives this problem type, the request might be modified unintentionally by an intermediary. The sender could use this information to retry the request without modification to address temporary transmission issues.

The following example shows a request with the content `{"hello": "woXYZ"}` (plus LF), but the representation digest for `{"hello": "world"}` (plus LF). The subsequent response indicates the mismatching SHA-256 digest value.

~~~ http-message
PUT /items/123 HTTP/1.1
Host: foo.example
Content-Type: application/json
Repr-Digest: sha-256=:RK/0qy18MlBSVnWgjwz6lZEWjP/lF5HF9bvEF8FabDg=:

{"hello": "woXYZ"}
~~~
{: title="A request with a sha-256 integrity field, which does not belong to the representation"}

~~~ http-message
# NOTE: '\' line wrapping per RFC 8792

HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://iana.org/assignments/http-problem-types#\
    digest-mismatching-values",
  "title": "Mismatching digest values",
  "mismatching-digests": [
    {
      "algorithm": "sha-256",
      "provided-digest": ":RK/0qy18MlBSVnWgjwz6lZEWjP/lF5HF9bvEF8FabDg=:",
      "header": "Repr-Digest"
    }
  ]
}
~~~
{: title="Response indicating the mismatching digests"}

# Security Considerations

Disclosing error details could leak information
such as the presence of intermediaries or the server's implementation details.
Moreover, they can be used to fingerprint the server.

To mitigate these risks, a server could assess the risk of disclosing error details
and prefer a general problem type over a more specific one.

When a server informs the client about mismatching digest values, it should not expose
the calculated digest to avoid exposing information that can be abused for oracle attacks.

# IANA Considerations

IANA is asked to register the following entries in the "HTTP Problem Types" registry at <https://www.iana.org/assignments/http-problem-types>.

## Registration of "digest-unsupported-algorithms" Problem Type

Type URI:
: https://iana.org/assignments/http-problem-types#digest-unsupported-algorithms

Title:
: Unsupported Hashing Algorithms

Recommended HTTP status code:
: 400

Reference:
: {{unsupported-hashing-algorithms}} of this document

## Registration of "digest-invalid-values" Problem Type

Type URI:
: https://iana.org/assignments/http-problem-types#digest-invalid-values

Title:
: Invalid Digest Values

Recommended HTTP status code:
: 400

Reference:
: {{invalid-digest-values}} of this document

## Registration of "digest-mismatching-values" Problem Type

Type URI:
: https://iana.org/assignments/http-problem-types#digest-mismatching-values

Title:
: Mismatching Digest Values

Recommended HTTP status code:
: 400

Reference:
: {{mismatching-digest-values}} of this document

--- back
