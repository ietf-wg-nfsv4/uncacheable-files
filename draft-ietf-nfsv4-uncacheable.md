---
title: Adding an Uncacheable Attribute to NFSv4.2
abbrev: Uncacheable
docname: draft-ietf-nfsv4-uncacheable-latest
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
  RFC7204:
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
  MS-ABE:
    title: Access-Based Enumeration (ABE) Concepts
    author:
      org: Microsoft
    target: https://techcommunity.microsoft.com/blog/askds/access-based-enumeration-abe-concepts-part-1-of-2/400435
    date: May 2009
  MS-SMB2:
    title: Server Message Block (SMB) Protocol Versions 2 and 3
    author:
      org: Microsoft Corporation
    seriesinfo:
      Microsoft: MS-SMB2
    target: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-smb2/
  OPEN-O_DIRECT:
    title: open(2) - Linux system call for opening files (O_DIRECT)
    target: https://man7.org/linux/man-pages/man2/open.2.html
    author:
    - org: Linux man-pages project
    date: 2024
  POSIX.1:
    title: The Open Group Base Specifications Issue 7
    seriesinfo: IEEE Std 1003.1, 2013 Edition
    author:
      org: IEEE 
    date:  2013
  RFC1813:
  RFC4949:
  RFC9754:
  Samba: 
    title: Samba Project
    author:
      org: Samba Team
    target: https://www.samba.org/
  SOLARIS-FORCEDIRECTIO:
    title: mount -o forcedirectio - Solaris forcedirectio mount option
    target: https://docs.oracle.com/en/operating-systems/solaris/oracle-solaris/11.4/manage-nfs/mount-options-for-nfs-file-systems.html
    author:
    - org: Oracle Solaris Documentation
    date: 2023
    seriesinfo:
      Solaris: "Administration Guide"

--- abstract

Network File System version 4.2 (NFSv4.2) clients commonly cache
file data and directory-entry metadata to improve performance. In
some environments, such caching can be unsuitable due to requirements
for predictable data visibility, correct metadata observation, or
server-controlled directory-entry presentation.  This document
defines an uncacheable attribute for NFSv4.2 that allows a server
to advise clients that client-side caching of file data and
directory-entry metadata is unsuitable. When supported by both the
client and server, the attribute affects client caching behavior
for files and directories on which it is set. This document extends
NFSv4.2 (see RFC7862).

--- note_Note_to_Readers

Discussion of this draft takes place
on the NFSv4 working group mailing list (nfsv4@ietf.org),
which is archived at
[](https://mailarchive.ietf.org/arch/search/?email_list=nfsv4). Source
code and issues list for this draft can be found at
[](https://github.com/ietf-wg-nfsv4/uncacheable).

Working Group information can be found at [](https://github.com/ietf-wg-nfsv4).

--- middle

# Introduction

Clients of remote filesystems commonly perform client-side caching
of file data in order to improve performance. Such caching may
include retaining data read from the server to satisfy subsequent
READ requests, as well as retaining data written by applications
in order to delay or combine WRITE requests before transmitting
them to the server.  While these techniques are effective for many
workloads, they may be unsuitable for environments that require
predictable data visibility or that involve concurrent modification
of shared files by multiple clients.

In some cases, Network File System version 4.2 (NFSv4.2) (see
{{RFC7862}}) mechanisms such as file delegations can reduce the
impact of concurrent access. However, delegations are not always
available or effective, particularly for workloads with frequent
concurrent writers or rapidly changing access patterns.

There have been prior efforts to bypass client-side file data caching
to address these issues. In High-Performance Computing (HPC)
environments, file data caching is often bypassed to improve
predictability and to avoid read-modify-write hazards when multiple
clients write disjoint byte ranges of the same file.  Applications
on some systems can request bypass of the client data cache by
opening files with the O_DIRECT flag (see {{OPEN-O_DIRECT}}).
However, this approach has limitations, including the requirement
that each application be explicitly modified and the lack of a
standardized mechanism for communicating this intent between servers
and clients.

This document defines an uncacheable attribute for NFSv4.2 that
allows a server to advise clients that client-side caching is
unsuitable for certain files and directories. When supported by
both the client and server, the attribute affects client caching
behavior in accordance with the semantics defined in this document.

The uncacheable attribute is read-write, has a data type of boolean,
and applies on a per-object basis. Support for the attribute is
specific to the exported filesystem and may differ between filesystems
served by the same server. A client can determine whether the
attribute is supported for a given object by issuing a GETATTR
request and examining the returned attribute list.

In addition to file data caching, clients of remote filesystems
commonly cache directory entries (dirents) and their associated
metadata to improve performance. Directory-entry metadata typically
includes names, file types, sizes, timestamps, and other attributes
returned by READDIR or related operations. This caching is commonly
shared across users on a client and assumes that directory contents
and access permissions are uniform across users.

In environments where directory-entry visibility or attributes vary
by user, this assumption does not hold. Access Based Enumeration
(ABE) {{MS-ABE}}, as implemented in the Server Message Block (SMB)
protocol {{MS-SMB2}} and deployed in implementations such as Samba
{{Samba}}, restricts directory visibility based on the access
permissions of the requesting user. Implementing similar behavior
in NFSv4.2 requires server involvement, as clients may not have
sufficient information to evaluate permissions based on identity
mappings, Access Control Lists (ACLs), or server-local policy.

While effective in SMB environments, the ABE model relies on per-user
directory views that are not safely cacheable across users. This
approach does not generalize well to NFS, where directory contents
and metadata have traditionally been shared and cached on the client.

Even in the absence of ABE, caching of directory-entry metadata can
lead to reuse of file-associated metadata that no longer reflects
current server state when files are modified concurrently. While
existing NFSv4.2 cache consistency mechanisms (such as change-attribute–based
validation) remain sufficient to ensure correctness, reliance on
previously observed metadata without revalidation can reduce the
predictability benefits that suppression of file data caching is
intended to provide.

NFSv4.2 enforces ACLs on the server rather than the client. As a
result, correct enforcement of access permissions may require the
client to bypass cached directory-entry metadata when evaluating
access for different users. Since cached dirents are shared across
users on a client, the client cannot safely determine per-user
visibility or attributes without consulting the server.

To address these issues, the uncacheable attribute defined in this
document also governs caching of directory-entry metadata. When the
attribute is set, the client is advised not to cache directory-entry
metadata for the affected files or directories, allowing the server
to provide directory-entry metadata that reflects current state and
user-specific access permissions.

Using the process described in {{RFC8178}}, the revisions in this
document extend NFSv4.2 {{RFC7862}}. They are built on top of the
external data representation (XDR) {{RFC4506}} generated from
{{RFC7863}}.

## Unified Attribute Semantics {#sec_uas}

This document defines a single uncacheable attribute whose effects
depend on the type of object on which it is set.

When set on a regular file (NF4REG), the uncacheable attribute
advises the client that client-side caching of file data is unsuitable
for that file. In particular, file data retrieved or written on
behalf of one user SHOULD NOT be reused to satisfy file access
requests made on behalf of another user, even when access permissions
would otherwise permit such reuse.

The attribute does not alter the semantics of the NFSv4 change
attribute.  Clients MAY continue to cache and reuse file-associated
metadata provided that such metadata is validated using the change
attribute as defined in {{RFC8881}}. However, when honoring the
uncacheable advice, clients SHOULD take care not to rely on previously
observed file-associated metadata without appropriate revalidation,
as doing so may reduce the predictability benefits that motivate
suppression of file data caching.

When set on a directory (NF4DIR), the uncacheable attribute advises
the client that caching of directory-entry metadata returned by
READDIR and related operations is unsuitable, including reuse of
such metadata across users.

In both cases, the attribute provides advisory guidance intended
to prevent clients from observing fresh file data through stale or
inappropriately shared metadata.

The uncacheable attribute does not require servers to omit attributes
from protocol responses, nor does it depend on servers hiding file
or directory metadata to enforce client behavior.

The uncacheable attribute does not mandate a particular client
implementation strategy. It provides advisory information intended to
align client behavior with server knowledge or workload requirements.
Clients that do not support or do not honor the attribute continue to
operate correctly according to existing NFSv4.2 semantics.

## Definitions

client-side caching of file data

: The retention of file data by a client in a local data cache,
commonly referred to as the page cache, for the purpose of satisfying
subsequent READ requests or delaying transmission of WRITE data to
the server.

write-behind caching

: A form of file data caching in which WRITE data is retained by
the client and transmission of the data to the server is delayed
in order to combine multiple WRITE operations or improve efficiency.

direct I/O

: An access mode in which file data is transferred between application
buffers and the underlying storage without populating or consulting
the client's file data cache. Direct I/O suppresses both read caching
and write-behind caching of file data.

write hole

: An instance of data corruption that arises when multiple clients
modify disjoint byte ranges within the same encoded data block
without having a consistent view of the existing contents. This can
result in stale data overwriting newer updates, particularly in
environments that use erasure encoding or striped storage.  (Adapted
from {{I-D.haynes-nfsv4-flexfiles-v2}}.)

dirent

: A directory entry representing a file or subdirectory and its
associated metadata.

dirent caching

: A client-side cache of directory entry names and associated
directory-entry metadata, used to avoid repeated directory lookup and
attribute retrieval.

Access Based Enumeration (ABE)

: A directory enumeration model in which the server presents directory
entries based on the access permissions of the user making the
request, such that directory-entry visibility may differ across users.

uncacheable attribute

: An NFSv4.2 attribute that advises clients that client-side caching of
file data and directory-entry metadata associated with an object is
unsuitable.

This document assumes familiarity with NFSv4 protocol operations,
error codes, object types, and attributes as defined in {{RFC8881}} and
{{RFC7862}}.

## Requirements Language

{::boilerplate bcp14-tagged}

# Client-Side Caching of File Data

The uncacheable attribute advises the client to bypass its page
cache for file data in certain cases. These include forms of
client-side caching such as write-behind caching, in which multiple
pending WRITEs are combined and transmitted to the server at a later
time for efficiency. When applied to file objects, the uncacheable
attribute inhibits such behavior with an effect similar to that of
using the O_DIRECT flag with the open call ({{OPEN-O_DIRECT}}).

On multi-user clients, file data caching is commonly shared across
users once access checks have succeeded. When the uncacheable
attribute is set on a file, such sharing is unsuitable.  Clients
are advised that reuse of file data retrieved on behalf of one user
to satisfy file access on behalf of another user may reduce the
predictability benefits associated with suppressing file data
caching, even when both users are authorized to access the file.

The intent of this attribute is to allow a server or administrator
to indicate that client-side caching of file data for a particular
file is unsuitable. The server is often in a better position than
individual clients to determine sharing patterns, access behavior,
or correctness requirements associated with a file. By exposing
this information via an attribute, the server can advise clients
to suppress file data caching in a consistent manner.

One important use case for this attribute arises in connection with
High-Performance Computing (HPC) workloads. These workloads often
involve large data transfers and concurrent access by multiple
clients. In such environments, client-side caching of file data can
introduce unpredictable latency or correctness hazards when data
is buffered and flushed at a later time.

Another aspect of such workloads is the need to support concurrent
writers to shared files. When application data spans a data block
in a client cache, delayed transmission of WRITE data can result
in clients modifying stale data and overwriting updates written by
others. Prompt transmission of WRITE data enables earlier detection
of write holes and reduces the risk of data corruption.

## Non-Goals

The uncacheable attribute does not require clients to provide strict
coherency, does not replace existing NFSv4.2 cache consistency mechanisms,
and does not mandate any specific client implementation strategy.
It provides advisory guidance intended to reduce latency and
correctness risks in selected workloads.

Nothing in this document alters the requirements for correct client
behavior defined in {{RFC8881}}. In particular, this document does
not prohibit caching of file-associated metadata, nor does it
invalidate change-attribute–based validation. Clients that cache
metadata and rely on the change attribute to detect modification
continue to behave correctly, whether or not the uncacheable attribute
is present.

## Uncacheable File Data {#sec_files}

When the uncacheable attribute is set on a regular file, the attribute
advises the client that client-side caching of file data for that
file is unsuitable. In particular, the client is advised to transmit
modifications to the file promptly rather than retaining them in a
local data cache. A client that does not query this attribute cannot
be expected to observe the behavior described in this section.

For files marked uncacheable, the client is advised not to retain
file data in its local data cache for the purpose of satisfying
subsequent READ requests or delaying transmission of WRITE data.
In such cases, READ operations bypass the client data cache, and
WRITE data is not retained for read-after-write satisfaction or for
the purpose of combining multiple WRITE requests.

Caching of unstably written data used to reissue WRITEs lost due
to server failure prior to COMMIT is not affected by the advice
provided by the uncacheable attribute. This is because the server
is made aware of the WRITE operation without the delays introduced
by write-behind caching.

Suppressing read caching in addition to suppressing write-behind
caching reduces the risk of stale-data overwrite in multi-writer
workloads. If a client retains cached READ data while other clients
concurrently modify disjoint byte ranges of the same file, the
client may perform a read-modify-write operation using stale data
and overwrite updates written by others. This risk exists even when
WRITE operations are transmitted promptly.

Disabling READ caching allows clients to observe the most recent
data prior to modification and reduces read-modify-write hazards
for shared files. This behavior is consistent with direct I/O
semantics such as those provided by the O_DIRECT flag in Linux and
the directio/forcedirectio mechanisms in Solaris.

If the uncacheable attribute is not set when a file is opened and
is changed while the file is open, the client is not expected to
retroactively alter its caching behavior. A client MAY choose to
flush cached data and apply the advice to subsequent I/O, but such
behavior is not required until the file is closed and reopened.

The presence of the uncacheable attribute does not invalidate file
delegations. A server that wishes to ensure prompt client I/O MAY
choose not to issue write delegations for files marked uncacheable,
but clients are not required to suppress delegations solely due to
the presence of this attribute.

# Setting the Uncacheable Attribute {#sec_setting}

The uncacheable attribute provides a mechanism by which applications
that do not support O_DIRECT can obtain direct-I/O-like semantics
for file access. In particular, when applied to a file, the attribute
allows a server to advise clients that client-side caching of file
data is unsuitable, including both read caching and write-behind
caching.

Suppressing read caching in addition to suppressing write-behind
caching is necessary to avoid read-modify-write hazards in multi-writer
workloads. If clients retain cached READ data while other clients
concurrently modify disjoint byte ranges of the same file, stale
cached data may be merged with new WRITE data and overwrite updates
written by others. This risk exists even when WRITE data is transmitted
promptly and is not addressed by suppressing write-behind caching
alone.

One possible deployment model is for a server or administrator to
configure a mount option (see {{MOUNT}}) such that newly created
files under a given export are marked uncacheable. In such a
configuration, an NFSv4.2 client could use SETATTR to set the
uncacheable attribute at file creation time.

This approach is conceptually similar in intent to the Solaris
forcedirectio mount option (see {{SOLARIS-FORCEDIRECTIO}}), but
differs in scope and visibility in that it allows direct-I/O-like
behavior to be applied without requiring changes to individual
applications. Unlike the Solaris option, the NFSv4.2 attribute is
visible to all clients accessing the file and is intended to convey
server-side knowledge or policy in a distributed environment.

# Caching of Directory-Entry Metadata

The uncacheable attribute is a read-write attribute with a data
type of boolean. When applied to a directory object, the attribute
governs client-side caching of directory-entry metadata returned
by READDIR and related operations on that directory.

The uncacheable attribute enables correct presentation of directory-entry
visibility and attributes, including but not limited to Access Based
Enumeration (ABE).  If both the client and the server support this
attribute, and it is set on a directory, the client is advised to
suppress caching of directory-entry metadata for that directory.

This document specifies the required externally observable behavior
rather than mandating a particular internal implementation strategy.
Clients MAY employ more sophisticated mechanisms, such as per-user
directory-entry caching, provided that the externally visible
behavior is equivalent to not caching directory-entry metadata
across users.

Allowing clients to set this attribute provides a portable mechanism
to request that directory-entry metadata not be cached, without
requiring changes to application behavior or out-of-band administrative
configuration.

A client can determine whether the uncacheable attribute is supported
for a given directory by issuing a GETATTR request and examining
the returned attribute list. The only way a server can determine
that a client supports the attribute is if the client sends a GETATTR
or SETATTR request that includes the attribute.

When applied to a directory, the uncacheable attribute governs
caching behavior of directory-entry metadata returned by READDIR
and related operations. It does not define behavior for positive
or negative name caching, nor for caching of LOOKUP results outside
the scope of directory-entry metadata.

Because the uncacheable attribute provides advisory guidance for both
file data caching and directory-entry metadata caching, clients that
honor the attribute are less likely to rely on previously observed
metadata without revalidation when accessing current file data.
This improves predictability in selected workloads but does not
replace existing NFSv4.2 cache consistency mechanisms.

The uncacheable attribute therefore governs caching of both file
data and directory-entry metadata, ensuring that clients do not
observe fresh file data through stale metadata.  Directory delegations
do not address per-user directory-entry metadata visibility and
therefore cannot replace the semantics defined by the uncacheable
attribute.

## Uncacheable Directory-Entry Metadata Semantics {#sec_dirents}

When the uncacheable attribute is set on a directory object, the
client is advised not to cache directory-entry metadata returned
for that directory. In such cases, the client retrieves directory-entry
metadata from the server as needed, allowing the server to evaluate
access permissions and visibility based on the requesting user.
Clients are advised not to share directory-entry metadata retrieved
on behalf of one user to satisfy requests made on behalf of another
user.

The uncacheable attribute does not modify the semantics of the
NFSv4.2 change attribute. Clients MUST continue to use the change
attribute to detect directory modifications and to determine when
directory contents may have changed, even when directory-entry
metadata caching is suppressed. Suppressing caching of directory-entry
metadata does not remove the need for change-based validation.

Servers SHOULD assume that clients which do not query or set this
attribute may cache directory-entry metadata, and therefore SHOULD
NOT rely on this attribute for correctness unless client support
has been confirmed.

Authorization to query, set, or modify the uncacheable attribute
is governed by existing NFSv4.2 authorization mechanisms.

If a client holds a directory delegation for a directory that becomes
marked with the uncacheable attribute, the server is expected to
ensure that the client observes the updated attribute value. A
server MAY recall an existing directory delegation in order to
enforce the semantics of this attribute. Clients that observe the
attribute set while holding a directory delegation MUST ensure that
directory-entry metadata is not cached inconsistently with the
attribute semantics.

Because this attribute provides advisory guidance rather than
mandatory access control, servers cannot rely on client compliance
for security enforcement in adversarial environments.

As described in {{sec_uas}}, the uncacheable attribute governs
directory-entry metadata both when set directly on a directory and
when set on a regular file whose metadata is observed via directory
enumeration.

# Example: Directory Enumeration With and Without Directory-Entry Metadata Caching

This example illustrates the difference in client-visible behavior
when directory-entry metadata caching is enabled versus when the
uncacheable attribute is set on a directory.

## Classic Directory Enumeration (Directory-Entry Metadata Cached)

In this scenario, the client caches directory-entry metadata obtained
from the server and reuses it for subsequent users.

~~~
User A Process          NFSv4.2 Client        NFSv4.2 Server
-------------           --------------        --------------
readdir("/dir")
   |
   |                     READDIR
   |-------------------->------------------------>
   |                     entries: {a,b,c}
   |<--------------------<------------------------
   |
(entries cached in client)

User B Process
-------------
readdir("/dir")
   |
   |                     (no network traffic)
   |                     entries returned from
   |                     client cache: {a,b,c}
~~~
{: #fig-cached-dirents title="Directory-Entry Metadata Cached"}

In this case, {{fig-cached-dirents}} shows directory-entry metadata
retrieved on behalf of User A reused to satisfy a directory read
for User B. This behavior is typical of legacy NFSv4.2 clients and
maximizes performance, but it can result in incorrect or unauthorized
directory views in multi-user or multi-protocol environments.

## Directory Enumeration With Uncacheable Attribute

In this scenario, the directory has the uncacheable attribute set.
The client does not retain directory-entry metadata across directory
reads for different users.

~~~
User A Process          NFSv4.2 Client        NFSv4.2 Server
-------------           --------------        --------------
readdir("/dir")
   |
   |                     READDIR
   |-------------------->------------------------>
   |                     entries visible to A:
   |                     {a,b}
   |<--------------------<------------------------
   |
(no directory-entry metadata retained)

User B Process
-------------
readdir("/dir")
   |
   |                     READDIR
   |-------------------->------------------------>
   |                     entries visible to B:
   |                     {b,c}
   |<--------------------<------------------------
~~~
{: #fig-uncached-dirents title="Directory-Entry Metadata Not Cached"}

In this case, {{fig-uncached-dirents}} shows each directory read
resulting in a READDIR operation sent to the server, ensuring that
directory-entry metadata reflects the visibility and attributes
appropriate to the requesting user. The client may still cache other
information, provided the externally observable behavior is equivalent
to not caching directory-entry metadata.

## Discussion

This example demonstrates that the uncacheable attribute does not
mandate a particular client implementation, but it does require
that directory-entry metadata retrieved for one user MUST NOT be
reused to satisfy directory reads for another user. The attribute
ensures correctness and interoperability in environments where
directory contents or visibility may differ across users, clients,
or protocols.

# Implementation Status

There is a prototype Hammerspace server that implements the uncacheable
attribute and a prototype Linux client that treats the attribute
as an indication to use O_DIRECT-like behavior for file access. In
the prototype, all files created under the configured mount point
are marked uncacheable.

Experience with the prototype indicates that use of the uncacheable
attribute can provide many of the practical benefits associated
with direct I/O without requiring application modification. For
applications that issue well-formed I/O requests, this approach has
been observed to improve performance in some cases, while also
reducing memory pressure and CPU utilization in the NFSv4.2 client.

# XDR for Uncacheable Attribute

~~~ xdr
///
/// typedef bool            fattr4_uncacheable;
///
/// const FATTR4_UNCACHEABLE              = 87;
///
~~~

# Extraction of XDR

This document contains the external data representation (XDR)
{{RFC4506}} description of the uncacheable attribute.  The XDR
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

The uncacheable attribute defined in this document is not intended
to provide a security boundary or to replace server-enforced access
control. Its primary purpose is to improve correctness and
interoperability in environments where client-side caching of file
data or directory-entry metadata may lead to inconsistent or incorrect
behavior. Servers MUST NOT rely on this mechanism alone to prevent
unauthorized access to files or directory entries.

Authorization to query, set, or modify the uncacheable attribute
is governed by existing NFSv4.2 authorization mechanisms. Servers
MAY restrict modification of this attribute based on local policy,
file or directory ownership, or access control rules. This document
does not define a new authorization model.

The discussion of users in this section is independent of the
specific user identity representation employed by the client or
server. This document does not distinguish between users identified
via NFSv4.2 user@domain strings, RPC authentication identities, or
local operating system user identifiers. The uncacheable attribute
does not alter NFSv4.2 authentication or authorization semantics
and does not depend on any particular user identity model.

When the uncacheable attribute is used to suppress caching of
directory-entry metadata, a client MUST NOT make access or visibility
decisions for one user based on directory-entry metadata retrieved
on behalf of another user. Such decisions MUST be made by the server.
If the client is Labeled NFS aware ({{RFC7204}}), the client MUST
locally enforce applicable mandatory access control (MAC) policies.

When the uncacheable attribute is set on a file, a client MUST NOT make
file data access decisions for one user based on file data retrieved on
behalf of another user.

The concerns described above primarily apply to multi-user clients
that cache directory-entry metadata on behalf of multiple users.
Single-user clients may not be subject to these risks; however, the
attribute semantics remain the same regardless of client usage
model.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

Trond Myklebust, Mike Snitzer, Jon Flynn, Keith Mannthey, and Thomas
Haynes all worked on the prototype at Hammerspace.

Rick Macklem, Chuck Lever, and Dave Noveck reviewed the document.

Chris Inacio, Brian Pawlowski, and Gorry Fairhurst helped guide
this process.
