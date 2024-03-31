# Docpatch
[.zshrc](..%2F.zshrc)
## Overview

Docpatch is a stateless document compression scheme.
A client, holding version 0 of a document, can retrieve a diff,
and use the diff to transform version 0 into the latest version of the document.

Key terms:
- A *docpatch file* holds a data file and splits it up into segments.
- A *docpatch header* is a set of hashes of segments from a docpatch file.
- A *docpatch diff file* is a docpatch file that uses hash references to include segments from a "base" docpatch file.

## Docpatch File Format

A docpatch file holds a data file and splits it up into segments.

To create a docpatch file, take a data file and insert
`^_` [0x1F ASCII Unit Separator](https://en.wikipedia.org/wiki/C0_and_C1_control_codes#Field_separators)
bytes around portions of the document that are likely to change.

To convert a docpatch file back into the data file, remove the separator bytes.

## HTTP Header

The `Docpatch: ` header contains a sequence of hashes of the segments of the base docpatch file.

To make the header value:
1. For each segment in the base docpatch file:
    - Get the bytes of the segment, not including start and end separator bytes.
    - Calculate the [XXH3_64bits](https://cyan4973.github.io/xxHash/) hash of the bytes.
    - Convert the hash to a sequence of 8 bytes in [little-endian](https://en.wikipedia.org/wiki/Endianness) order.
2. Concatenate the hash byte sequences in any order.
3. Encode the sequence in [Base64](https://en.wikipedia.org/wiki/Base64),
   using the "standard" alphabet (from [RFC-4648](https://datatracker.ietf.org/doc/html/rfc4648#section-4)),
   without padding.

To prevent [431 Request Header Fields Too Large](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/431) errors,
applications should limit the number of segments when generating docpatch files.
A good default limit is 94 segments, which limits the HTTP/1.1 docpatch header line to 1014 bytes.

## Docpatch Diff Format
A docpatch diff file represents the differences between two docpatch files: the *base* file and *target* file.
A machine holding the base file and diff file can quickly produce the target file.

A docpatch diff file is a sequence of segments and inclusions:
* The `^P` ([0x10 ASCII Data Link Escape](https://en.wikipedia.org/wiki/C0_and_C1_control_codes#DLE)) byte signals
  an inclusion.
  It must be followed by 12 bytes which are the XXH3_64bits hash of a base file segment,
  in little-endian order, encoded in Base64 without padding.
* A segment is any other sequence of bytes, delimited by `^_` or an inclusion.

## Content-Type
Denote docpatch files with MIME type: `application/docpatch`.
Denote docpatch diff files with MIME type: `application/docpatch-diff`.

When requesting a docpatch diff, send the `Accept: application/docpatch-diff` header.

Why use `Accept` and not `Accept-Encoding`?
At the time of writing, browsers do not allow JavaScript to set the `Accept-Encoding` header.
If browsers add support for docpatch, applications can switch to `Accept-Encoding`.

If your application must support various response content-types, send these types in a `Docpatch-accept: ` header.

## Caching

HTTP servers returning a docpatch diff file SHOULD include `Cache-Control: no-store` header
to disallow caching of the response.  
If the HTTP server allows caching, it MUST include the 'Vary: Docpatch' header in the response.

## Example
Example docpatch file:
```
<html>
<head>
    <title>User5 - Chats</title>
    <link rel="stylesheet" type="text/css" href="/style.css">
    <script src="/update-23.js"></script>
    <script src="/ticks-11.js"></script>
    <subscribe ticks="/chats/5">
</head>
<body>
<ul id="top_menu">
    <li><a href="/chats">Chats</a></li>
    <li><a href="/logout">Logout</a></li>
</ul>
<div id="chat">
    <h1>User5</h2>
    <ul id="messages">
^_
        <li class="received">message1</li>
^_
        <li class="sent">message2</li>
^_
    </ul>
    <form id="message_form" method="post" update>
        <input type="text" name="message" required>
        <input type="submit" value="Send">
    </form>
</div>
</body>
</html>
```

The file has four segments:
- HTML `head`, menu, and everything up to the first message
- List item with message1
- List item with message2
- Everything after the last message

TODO: Add example header
TODO: Add example diff

# Escaping
To use docpatch with files that may contain the `^_` or `^P` symbols, you must escape these symbols.
The recommended scheme is to use the
`^[` ([0x1B ASCII Escape symbol](https://en.wikipedia.org/wiki/C0_and_C1_control_codes#ESC))
followed by the byte number in hexadecimal format:
- `^[ -> ^[ 1 B`
- `^_ -> ^[ 1 F`
- `^P -> ^[ 1 0`
