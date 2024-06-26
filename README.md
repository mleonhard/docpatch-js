# docpatch-js
`docpatch.js` is a JavaScript library implementing the `<form docpatch>` attribute.

# Overview

Docpatch is a stateless document compression scheme.
A client, holding version 0 of a document, can request the latest version of the document
and receive a diff to transform version 0 into that latest version.

Key terms:
- A *docpatch file* holds a data file and splits it up into segments.
- A *docpatch header* is a list of 64-bit XXH3 hashes of segments from a docpatch file.
- A *docpatch diff* is a docpatch file that includes segments from a "base" docpatch file.

# Docpatch File Format

A docpatch file holds a data file and splits it up into segments.

To create a docpatch file, take a data file and insert
`^_` [0x1F ASCII Unit Separator](https://en.wikipedia.org/wiki/C0_and_C1_control_codes#Field_separators)
bytes around portions of the document that are likely to change.

To convert a docpatch file back into the data file, remove the separator bytes.

Mime type: `application/docpatch`

# HTTP Header

The `Docpatch: ` header contains a sequence of hashes of the segments of the base docpatch file.

To make the header value:
1. For each segment in the base docpatch file:
   - Get the bytes of the segment, not including start and end separator bytes.
   - Calculate the [XXH3_64bits](https://cyan4973.github.io/xxHash/) hash of bytes.
   - Convert the hash to a sequence of 8 bytes in [little-endian](https://en.wikipedia.org/wiki/Endianness) order.
2. Concatenate the hash byte sequences in any order.
3. Encode the sequence in [Base64](https://en.wikipedia.org/wiki/Base64),
   using the "standard" alphabet (the one from [RFC-4648](https://datatracker.ietf.org/doc/html/rfc4648#section-4)),
   without padding.

To prevent [431 Request Header Fields Too Large](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/431) errors,
applications should limit the number of segments when generating docpatch files.
A good default limit is 94 segments, which limits the HTTP/1.1 docpatch header line to 1014 bytes.

# Docpatch Diff Format
A docpatch diff file represents the differences between two docpatch files: the *base* file and *target* file.
A computer program holding the base file and diff file can quickly produce the target file.

A docpatch diff file is a sequence of segments and inclusions:
* A segment is a sequence of bytes separated by the `^_` symbol.

A docpatch diff file holds a compressed docpatch file.
The data file is split up into segments.
Some segments are elided and replaced with hashes referring to segments in the "base" docpatch file.

To create a docpatch file, take a data file and 

TODO:
- Include hashed segments with `^P`
  ([0x10 ASCII Data Link Escape](https://en.wikipedia.org/wiki/C0_and_C1_control_codes#DLE))
  byte followed by the hash of the segment
- Use [XXH3](https://cyan4973.github.io/xxHash/) to hash the segments
- See the References segment of the [`flit`](https://crates.io/crates/flit) Rust library.

# Example
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

# Escaping
To use docpatch with files that may contain the `^_` or `^P` symbols, you must escape these symbols.
The recommended scheme is to use the
`^[` ([0x1B ASCII Escape symbol](https://en.wikipedia.org/wiki/C0_and_C1_control_codes#ESC))
followed by the byte number in hexadecimal format:
- `^[ -> ^[ 1 B`
- `^_ -> ^[ 1 F`
- `^P -> ^[ 1 0`

# TO DO
- Host on Github and a CDN
- Make a release script that updates the hash in this readme.
- Support `application/docpatch` content type.

# Findings
- JavaScript in a webpage can get HTML of the current page, with edits.
  Lots of JavaScript libraries store their data in the DOM.
  There is no way to get the file that the script was loaded from.
- JavaScript can use the fetch API to re-request the current HTML page.
  It can use an etag so the server can cache.
