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
  RFC7862:
  RFC7863:
  RFC8174:
  RFC8178:
  RFC8881:

informative:
  I-D.haynes-nfsv4-flexfiles-v2:
  MOUNT:
    title: mount(2) - mount filesystem
    target: https://man7.org/linux/man-pages/man2/mount.2.html
    author:
    - org: Linux man-pages project
    date: 2024
    seriesinfo:
      Linux: "Programmer's Manual"
  OPEN:
    title: open(2) - open and possibly create a file
    target: https://man7.org/linux/man-pages/man2/open.2.html
    author:
    - org: Linux man-pages project
    date: 2024
    seriesinfo:
      Linux: "Programmer's Manual"
  OPEN-O_DIRECT:
    title: open(2) - Linux system call for opening files (O_DIRECT)
    target: https://man7.org/linux/man-pages/man2/open.2.html
    author:
    - org: Linux man-pages project
    date: 2024
  SOLARIS-FORCEDIRECTIO:
    title: mount -o forcedirectio - Solaris forcedirectio mount option
    target: https://docs.oracle.com/en/operating-systems/solaris/oracle-solaris/11.4/manage-nfs/mount-options-for-nfs-file-systems.html
    author:
    - org: Oracle Solaris Documentation
    date: 2023
    seriesinfo:
      Solaris: "Administration Guide"

--- abstract

The Network File System version 4.2 (NFSv4.2) allows a client to
cache data for file objects.  Applications on some clients can
control the caching of data, but there is no way to achieve this
at a system level.  This document introduces a new uncacheable file
data attribute for NFSv4.2.  Files marked as uncacheable file data
SHOULD NOT have their data stored in client-side caches.  This
document extends NFSv4.2 (see RFC7862).

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
achieve consistent work flows and to allow concurrent access from
many writers.

Applications can use O_DIRECT on open (see {{OPEN-O_DIRECT}}) to
force the client to bypass the page cache, but the limitation
is that each application must be modified to use this flag.

This document introduces the uncacheable file data attribute to
NFSv4.2 to bypass file caching on the client. As such, it is an
OPTIONAL attribute to implement for NFSv4.2. However, if both the
client and the server support this attribute, then the client SHOULD
follow the semantics of the uncacheable file data attribute.

The uncacheable file data attribute is read-write and per file. The
data type is bool.

A client can easily determine whether or not a server supports the
uncacheable file data attribute with a simple GETATTR on any file.
If the server does not support the uncacheable file data attribute,
it will return an error of NFS4ERR_ATTRNOTSUPP.

The only way that the server can determine that the client supports
the attribute is if the client sends either a GETATTR or SETATTR
with the uncacheable file data attribute.

As bypassing file caching is file based, it is only applicable for
objects which are of type attribute value of NF4REG.

Using the process detailed in {{RFC8178}}, the revisions in this document
become an extension of NFSv4.2 {{RFC7862}}. They are built on top of the
external data representation (XDR) {{RFC4506}} generated from
{{RFC7863}}.

## Definitions

file caching

: A client cache, normally called the page cache, which caches the
contents of a regular file. Typical usage would be to accumulate
changes to be bunched together for writing to the server.

write hole

: A write hole is a data corruption scenario where either two clients
are trying to write to the same erasure encoded block of dataor one
client is overwriting an existing erasure encoded block of data.
(Adapted from {{I-D.haynes-nfsv4-flexfiles-v2}}.) Note the hole
occurs when multiple data servers are not consistent with the
encoded block.

Further, the definitions of the following terms are referenced as follows:

- COMMIT (({{Section 18.3 of RFC8881}})
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
its page cache for the file.  This often includes write-behind
caching used to combine multiple pending WRITEs into a single
operation for more efficient remote processing. The uncacheable
file data attribute, inhibits this behavior with an effect similar
to the user using the O_DIRECT flag with the open call ({{OPEN}}).
Under some conditions clients might be able to determine that files
are not shared or do not exhibit access patterns suitable for
write-behind caching. In such situations, setting this attribute
provides a way to inform other clients of this judgment.

The most pressing need for this feature is in connection with HPC
workloads. These often involve massive data transfers and require
extremely low latency. Write-behind caching can introduce unpredictable
latency, as data is buffered and flushed later.

Another aspect of such workloads is the need to share data between
multiple writers. As the application data may span a data block
in a page cache, data needs to be flushed immediately for the
detection of write holes.

## Uncacheable File Data {#sec_files}

If a file object is marked as uncacheable file data, all modifications
to the file SHOULD be immediately sent from the client to the server.
I.e., if a NFSv4.2 client fails to query this attribute, then it
can not meet the requirements of the attribute.

For uncacheable data, the client MUST NOT retain file data in its
local data cache for the purpose of satisfying subsequent READ
requests or delaying transmission of WRITE data. Reads MUST bypass
the client data cache, and WRITE data MUST NOT be retained for
read-after-write satisfaction or for the purpose of combining
multiple WRITE requests.

Caching of UNSTABLE WRITE data required to support the NFSv4.2
COMMIT operation is permitted and unaffected by this requirement.

Suppression of read caching is required in addition to suppression
of write caching to prevent stale-data overwrite in multi-writer
workloads. If a client retains cached READ data for a file while
other clients are concurrently modifying disjoint byte ranges, the
client may perform a read-modify-write operation using stale data,
thereby overwriting updates written by other clients. This hazard
exists even when WRITE operations are transmitted immediately to
the server and no write-behind caching is performed.

Disabling READ caching ensures that clients observe the most recent
data prior to modification and avoids read-modify-write hazards for
shared files. This behavior is consistent with direct I/O semantics
such as those provided by the O_DIRECT flag in Linux and the
directio/forcedirectio mechanisms in Solaris.

If the fattr4_uncacheable_file_data is not set when a client opens
a file and is changed whilst the file is open, the client is not
responsible for bypassing the page cache. It could flush the page
cache and make all subsequent IO be direct, but until the
client closes the file and reopens it, it is not required to
meet the requirements of the attribute.

If the client has a OPEN_DELEGATE_WRITE delegation on the file
(see Section 10.4 of {{RFC8881}}), then the uncacheable file data
attribute takes precedence over the caching of file data. The
server can control this by not issuing file delegations for
files with this attribute.

# Setting the Uncacheable File Data Attribute {#sec_setting}

The uncacheable file data attribute can allow for applications which
do not support O_DIRECT to be able to use O_DIRECT semantics.  One
approach to support this would be to add a new parameter to mount
{{MOUNT}} that specifies all newly created files under that mount
point would have their data not cacheable. Then the NFSv4.2 client
would use a SETATTR to set fattr4_uncacheable_file_data. This
approach is similar to the Solaris forcedirectio (See
{{SOLARIS-FORCEDIRECTIO}}) mount option.

# Implementation Status

There is a prototype Hammerspace server which implements the
uncacheable file data attribute and a prototype Linux client which
treats the uncacheable file data attribute as meaning use O_DIRECT.
For the prototype, all files created under the mount point have the
fattr4_uncacheable_file_data set to be true.

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
<CODE BEGINS>
#!/bin/sh
grep '^ *///' $* | sed 's?^ */// ??' | sed 's?^ *///$??'
<CODE ENDS>
~~~

For example, if the script is named 'extract.sh' and this document is
named 'spec.txt', execute the following command:

~~~ shell
<CODE BEGINS>
sh extract.sh < spec.txt > uncacheable_prot.x
<CODE ENDS>
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
