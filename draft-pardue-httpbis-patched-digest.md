---
title: "HTTP Patched Digest"
category: std

docname: draft-pardue-httpbis-patched-digest-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "HTTP"
keyword:
 - next generation
 - unicorn
 - AI-native
venue:
  group: "HTTP"
  type: "Working Group"
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "LPardue/patched-digest"
  latest: "https://LPardue.github.io/patched-digest/draft-pardue-httpbis-patched-digest.html"

author:
 -
    fullname: "Lucas Pardue"
    email: "lucas@lucaspardue.com"

normative:
  UNENC-DIGEST: I-D.draft-ietf-httpbis-unencoded-digest

informative:

...

--- abstract

The PATCH method can be used to apply partial modifications to a resource. This
document defines the Patched-Digest request field, which allows a client to
indicate the integrity digest of the modified resource once a PATCH operation is
applied. A server can use the integrity digest to detect an operation failure
and return an error. The Want-Patched-Digest response field is also defined to
signal server integrity preferences.


--- middle

# Introduction

The PATCH method {{!PATCH=RFC5789}} can be used to apply partial modifications
to a resource. A client can indicate the desired precondition of the resource
prior to a PATCH operation being applied ({{Section 2 of PATCH}}). For example,
sending a PATCH request containing an If-Match ({{Section 13.1.1 of
!HTTP=RFC9110}}) with a strong ETag ({{Section 8.8.3 of HTTP}}).

The following example illustrates a hypothetical PATCH request to apply a patch
document to an existing resource with the ETag "e0023aa4e":

~~~ http-message
PATCH /file.txt HTTP/1.1
Host: www.example.com
Content-Type: application/example
If-Match: "e0023aa4e"
Content-Length: 100

[description of changes]
~~~

The PATCH operation results in a successful response that indicates the resource
has been modified and the new ETag is "e0023aa4f.

~~~ http-message
HTTP/1.1 204 No Content
Content-Location: /file.txt
Content-Type: text/plain
ETag: "e0023aa4f"
~~~

To date there have been no means to communicate the client's expectation of the
result of a PATCH operation, delaying the ability for both client and server to
detect an error.

While the integrity fields defined in {{!DIGEST-FIELDS=RFC9530}} and
{{UNENC-DIGEST}} can be used by a client to obtain information related to the
resource before and/or after a PATCH operation, that cannot be used to
communicate an a priori expectation for the result. For instance, the example
request could be augmented to include integrity fields but all of them refer to
the patch document itself, not any state of the resource:

~~~ http-message
PATCH /file.txt HTTP/1.1
Host: www.example.com
Content-Type: application/example
If-Match: "e0023aa4e"
Content-Length: 100
Content-Digest: [something]
Repr-Digest: [something]
Unencoded-Digest: [something]

[description of changes]
~~~

The earlier example response can be similarly augmented with integrity fields
that indicate the state of the resource after modification:

~~~ http-message
HTTP/1.1 204 No Content
Content-Location: /file.txt
Content-Type: text/plain
ETag: "e0023aa4f"
Repr-Digest: [something]
Unencoded-Digest: [something]
~~~

The client might be able to use either `Repr-Digest` or `Unencoded-Digest` to
detect a problem that occurred during the PATCH operation. However, the server
would remain oblivious unless further action were taken by the client to
communicate a failure.

This document defines the `Patched-Digest` request field, which allows
a client to indicate the integrity digest ({{!DIGEST-FIELDS=RFC9530}}) of the
modified resource once a PATCH operation is applied. A server can use the
integrity digest to detect an operation failure and return an error response.
`Patched-Digest` complements other integrity fields but has a much narrower
usage scope.

As is common for integrity fields, the `Want-Patched-Digest` response field is
also defined to signal server integrity preferences.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the following terminology from {{Section 3 of
!STRUCTURED-FIELDS=RFC9651}} to specify syntax and parsing: Byte Sequence,
Dictionary, and Integer.

The definitions "representation", "selected representation", "representation
data", "representation metadata", and "content" in this document are to be
interpreted as described in {{!HTTP=RFC9110}}.

This document uses the line folding strategies described in {{?FOLDING=RFC8792}}.

The term "digest" is to be interpreted as described in {{DIGEST-FIELDS}}.

# The Patched-Digest Field {#patched-digest}

The `Patched-Digest` HTTP field can be used in requests to communicate an
expected digest for the result of applying a PATCH document to a resource. The
digest is calculated using a hashing algorithm applied to the resource's entire
selected representation data with no content codings applied ({{Section 8.4.1 of
HTTP}}).

In the following examples of `Patched-Digest` fields, the representation data
with no content codings applied is: "An unexceptional string" followed by a
line feed character (0xA).

~~~ http-message
NOTE: '\' line wrapping per RFC 8792

Patched-Digest: \
  sha-512=:WjyMuMD9EI/v0RoJchcevbo6lF498VyE9564OgXf+98iJptoSvb1Czo9\
  uVJu2bVU/tOv90huiMG3+YaMX1kipw==:
~~~

The `Dictionary` type can be used, for example, to attach multiple digests
calculated using different hashing algorithms in order to support a population
of endpoints with different or evolving capabilities. Such an approach could
support transitions away from weaker algorithms (see
{{Section 6.6 of DIGEST-FIELDS}}).

~~~ http-message
NOTE: '\' line wrapping per RFC 8792

Patched-Digest: \
  sha-256=:5Bv3NIx05BPnh0jMph6v1RJ5Q7kl9LKMtQxmvc9+Z7Y=:,\
  sha-512=:WjyMuMD9EI/v0RoJchcevbo6lF498VyE9564OgXf+98iJptoSvb1Czo9\
  uVJu2bVU/tOv90huiMG3+YaMX1kipw==:
~~~

A server MAY ignore any or all digests. Application-specific behavior or
local policy MAY set additional constraints on the processing and validation
practices of the conveyed digests. Security considerations related to ignoring
digests or validating multiple digests are presented in {{Sections 6.6 and
6.7 of DIGEST-FIELDS}} respectively. TODO: should we be more strict and
recommend returning an error (and maybe point to RFC 5789 Section 2.2)?

A client MAY send a digest without knowing whether the server supports a
given hashing algorithm. A sender MAY send a digest if it knows the recipient
will ignore it. An example is depicted in {{Appendix C.2 of DIGEST-FIELDS}}.

`Patched-Digest` can be sent in a trailer section. In this case,
`Patched-Digest` MAY be merged into the header section; see {{Section 6.5.1 of
HTTP}}.

# The Want-Patched-Digest Field {#want-patched-digest}

`Want-Patched-Digest` is an integrity preference field; see {{Section 4 of
DIGEST-FIELDS}}. It indicates that the server would like to receive a
`Patched-Digest` with PATCH requests.

`Want-Patched-Digest` is only a hint. Clients can ignore it and send an
`Patched-Digest` field using any algorithm or omit the field entirely. It is not
a protocol error if preferences are ignored. Applications that use
`Patched-Digest` and `Want-Patched-Digest` can define expectations or
constraints that operate in addition to this specification.  Ignored preferences
are an application-specific concern.

`Want-Patched-Digest` is of type `Dictionary` where each:

* key conveys the hashing algorithm;
* value is an `Integer` ({{Section 3.3.1 of STRUCTURED-FIELDS}}) that conveys an
  ascending, relative, weighted preference. It must be in the range 0 to 10
  inclusive. 1 is the least preferred, 10 is the most preferred, and a value of
  0 means "not acceptable".

  Each Dictionary value can have zero or more Parameters ({{Section 3.1.2 of
  STRUCTURED-FIELDS}}). This specification does not define any Parameters;
  future extensions may do so. Unknown Parameters MUST be ignored.

Examples:

~~~ http-message
Want-Patched-Digest: sha-256=1
Want-Patched-Digest: sha-512=3, sha-256=10, unixsum=0
~~~

# Security Considerations

All the same considerations documented in {{DIGEST-FIELDS}} apply.


# IANA Considerations

IANA is asked to update the "Hypertext Transfer Protocol (HTTP) Field Name
Registry" {{!HTTP=RFC9110}} as shown in the table below:

|-----------------------|-----------|-----------------|--------------------------------------------|
| Field Name            | Status    | Structured Type | Reference                                  |
|-----------------------|-----------|-----------------|--------------------------------------------|
| Patched-Digest        | permanent | Dictionary      | {{patched-digest}} of this document      |
| Want-Patched-Digest   | permanent | Dictionary      | {{want-patched-digest}} of this document |
|-----------------------|-----------|-----------------|--------------------------------------------|
{: #iana-field-name-table title="Hypertext Transfer Protocol (HTTP) Field Name Registry Update"}

--- back

# Acknowledgments
{:numbered="false"}

The PATCH integrity capability gap was identified by a discussion with Grant
Gryczan. Roberto Polli provided a reminder that the topic was touched on during
RFC 9530 development, and provided some early technical input related to the
design in this document.
