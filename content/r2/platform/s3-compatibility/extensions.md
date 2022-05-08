---
title: Extensions
pcx-content-type: reference
meta:
  title: Extensions
---

# Extensions

R2 implements some extensions on top of the stock S3 API. This page outlines those additional features that are available:

## Extended Metadata Using Unicode

The [Cloudflare Workers R2 API](/workers/runtime-apis/r2) supports Unicode in keys and values natively without any additional
encoding/decoding needed for the `customMetadata` field. These fields map 1:1 to the headers that are prefixed with `x-amz-meta-`
in the R2 S3-compatible API endpoint.

To bridge the gap that HTTP headers can only have ASCII keys and values, all `x-amz-meta-` prefixed header values are RFC2047 decoded
before being stored in R2. Values with unicode are RFC2047 encoded before rendering back responses. The length limit on metadata
values applies on the decoded Unicode value.

Be mindful of this when using both Workers and S3 API endpoints to access the same data. If the R2 metadata keys contain Unicode then
they are stripped when returning responses through S3 and the `x-amz-missing-meta` header is set to the number of keys that were omitted.

Unicode is not allowed in normal HTTP headers and those headers must always follow standard rules for HTTP. These headers map to the
`httpMetadata` field in the R2 bindings:
* `Content-Encoding`
* `Content-Type`
* `Content-Language`
* `Content-Disposition`
* `Cache-Control`
* `Expires`

If using Unicode in object key names, it may be helpful to review the [technical notes](/r2/platform/unicode-interoperability) regarding
that.

## CopyObject

### MERGE metadata directive

The `x-amz-metadata-directive` can be given a value of `MERGE` (in addition to the standard `COPY`/`REPLACE` versions). This is
a combination of `COPY` + `REPLACE` which will `COPY` any metadata keys from the source object and `REPLACE` any that are specified
in the request with the new value. This cannot be used currently to merge in deletion of existing metadata keys. For that you still
need to use `REPLACE`.
