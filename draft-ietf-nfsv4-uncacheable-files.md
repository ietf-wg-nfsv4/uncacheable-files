---
title: Adding an Uncacheable File Data Attribute to NFSv4.2
abbrev: Uncacheable File
docname: draft-ietf-nfsv4-uncacheable-files-latest
category: std
date: {DATE}
consensus: true
ipr: trust200902
area: General
workgroup: Network File System Version 4
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping, comments]

author:
 -
    ins: T. Haynes
    name: Thomas Haynes
    organization: Hammerspace
    email: loghyr@gmail.com

normative:
  RFC2119:
  RFC4506:
  RFC4949:
  RFC7862:
  RFC7863:
  RFC8174:
  RFC8178:
  RFC8881:

informative:
  open:
    title: open and create files.
    seriesinfo: Linux Programmer's Manual
  POSIX.1:
    title: The Open Group Base Specifications Issue 7
    seriesinfo: IEEE Std 1003.1, 2013 Edition
    author:
      org: IEEE
    date:  2013
  RFC1813:

--- abstract

The Network File System version 4.2 (NFSv4.2) allows a client to
cache data for file objects.  Caching file data can lead to performance
issues if the cache hit rate is low.  This document introduces a
new uncacheable file data attribute for NFSv4.2.  Files marked as
uncacheable file data SHOULD NOT have their data be stored in
client-side caches.  This document extends NFSv4.2 (see RFC7862).

--- note_Note_to_Readers

Discussion of this draft takes place
on the NFSv4 working group mailing list (nfsv4@ietf.org),
which is archived at
[](https://mailarchive.ietf.org/arch/search/?email_list=nfsv4). Source
code and issues list for this draft can be found at
[](https://github.com/ietf-wg-nfsv4/uncacheable-files).

Working Group information can be found at [](https://github.com/ietf-wg-nfsv4).

--- middle

# Introduction

With a remote filesystem, the client typically caches file contents
in order to improve performance.  Several assumptions are made about
the rate of change in the number of clients trying to concurrently
access a file.  With NFSv4.2, this could be mitigated by file
delegations for the file contents.

There are prior efforts to bypass file caching.  In Highly Parallel
Computing (HPC) workloads, file caching is bypassed in order to
achieve consistent work flows.

This document introduces the uncacheable file data attribute to
NFSv4.2 to bypass file caching on the client. As such, it is an
OPTIONAL attribute to implement for NFSv4.2. However, if both the
client and the server support this attribute, then the client MUST
follow the semantics of the uncacheable file data attribute.

The uncacheable file data attribute is read-only and per file. The
data type is bool.

A client can easily determine whether or not a server supports the
uncacheable file data attribute with a simple GETATTR on any file.
If the server does not support the uncacheable file data attribute,
it will return an error of NFS4ERR_ATTRNOTSUPP.

The only way that the server can determine that the client supports
the attribute is if the client sends either a GETATTR with the
uncacheable file data attribute.

As bypassing file caching is file based, it is only applicable for
dirents which are of type attribute value of NF4REG.

Using the process detailed in {{RFC8178}}, the revisions in this document
become an extension of NFSv4.2 {{RFC7862}}. They are built on top of the
external data representation (XDR) {{RFC4506}} generated from
{{RFC7863}}.

## Definitions

file caching

: A client cache, normally called the page cache, which caches the
contents of a regular file. Typical usage would be to accumulate
changes to be bunched together for writing to the server.

Further, the definitions of the following terms are referenced as follows:

- file delegations ({{Section 10.2 of RFC8881}})
- GETATTR ({{Section 18.7 of RFC8881}})
- NF4REG ({{Section 5.8.1.2 of RFC8881}})
- NFS4ERR_ATTRNOTSUPP ({{Section 15.1.15.1 of RFC8881}})
- SETATTR ({{Section 18.30 of RFC8881}})
- system ({{Section 5.8.2.36 of RFC8881}})

## Requirements Language

{::boilerplate bcp14-tagged}

# Caching of File Data

The uncacheable file data attribute instructs the client to bypass
its page cache for the file. This behavior is similar to using the
O_DIRECT flag with the open call ({{open}}). This can be beneficial
for files that are not shared or do not exhibit access patterns
suitable for caching.

However, the real need for bypassing write caching is evident in
HPC workloads. In general, these involve massive data transfers and
require extremely low latency.  Write caching can introduce
unpredictable latency, as data is buffered and flushed later.

## Uncacheable File Data {#sec_files}

If a file object is marked as uncacheable file data, all modifications
to the file SHOULD be immediately sent from the client to the server.
I.e., if a NFSv4.2 client fails to query this attribute, then it
can not meet the requirements of the attribute.

If the client has a OPEN_DELEGATE_WRITE delegation on the file
(see Section 10.4 of {{RFC8881}}), then the uncacheable file data
attribute takes precedence over the caching of file data. The
server can control this by not issuing file delegations for
files with this attribute.

# XDR for Uncacheable Attribute

~~~ xdr
///
/// typedef bool            fattr4_uncacheable_file_data;
///
/// const FATTR4_UNCACHEABLE_FILE_DATA       = 87;
///
~~~

# Extraction of XDR

This document contains the external data representation (XDR)
{{RFC4506}} description of the uncacheable file attribute.  The XDR
description is presented in a manner that facilitates easy extraction
into a ready-to-compile format. To extract the machine-readable XDR
description, use the following shell script:

~~~ shell
#!/bin/sh
grep '^ *///' $* | sed 's?^ */// ??' | sed 's?^ *///$??'
~~~

For example, if the script is named 'extract.sh' and this document is
named 'spec.txt', execute the following command:

~~~ shell
sh extract.sh < spec.txt > uncacheable_prot.x
~~~

This script removes leading blank spaces and the sentinel sequence '///'
from each line. XDR descriptions with the sentinel sequence are embedded
throughout the document.

Note that the XDR code contained in this document depends on types from
the NFSv4.2 nfs4_prot.x file (generated from {{RFC7863}}).  This includes
both nfs types that end with a 4, such as offset4, length4, etc., as
well as more generic types such as uint32_t and uint64_t.

While the XDR can be appended to that from {{RFC7863}}, the code snippets
should be placed in their appropriate sections within the existing XDR.

# Security Considerations

This document imposes no new security considerations to NFSv4.2.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

Trond Myklebust, Mike Snitzer, and Thomas Haynes all worked on the
prototype at Hammerspace.

Rick Macklem, Chuck Lever, and Dave Noveck reviewed the document.

Chris Inacio, Brian Pawlowski, and Gorry Fairhurst helped guide
this process.
