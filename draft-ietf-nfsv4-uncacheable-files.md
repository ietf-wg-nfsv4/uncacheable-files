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

Network File System version 4.2 (NFSv4.2) clients commonly perform
client-side caching of file data in order to improve performance.
On some systems, applications may influence client data caching
behavior, but there is no standardized mechanism for a server or
administrator to indicate that particular file data should not be
cached by clients for reasons of performance or correctness. This
document introduces a new file data caching attribute for NFSv4.2.
Files marked with this attribute are intended to be accessed with
client-side caching of file data suppressed, in order to support
workloads that require predictable data visibility. This document
extends NFSv4.2 (see RFC7862).

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

Clients of remote filesystems commonly perform client-side caching
of file data in order to improve performance.  Such caching may
include retaining data read from the server to satisfy subsequent
READ requests, as well as retaining data written by applications
in order to delay or combine WRITE requests before transmitting
them to the server.  While these techniques are effective for many
workloads, they may be unsuitable for workloads that require
predictable data visibility or involve concurrent modification of
shared files by multiple clients.

In some cases, Network File System version 4.2 (NFSv4.2) (see
{{RFC7862}}) mechanisms such as file delegations can reduce the
impact of concurrent access.  However, delegations are not always
available or effective, particularly for workloads with frequent
concurrent writers or rapidly changing access patterns.

There have been prior efforts to bypass file data caching in order to
address these issues.  In High-Performance Computing (HPC) workloads,
file data caching is often bypassed to improve predictability and to
avoid read-modify-write hazards when multiple clients write disjoint
byte ranges of the same file.

Applications on some systems can request bypass of the client data
cache by opening files with the O_DIRECT flag (see {{OPEN-O_DIRECT}}).
However, this approach has limitations, including the requirement
that each application be explicitly modified and the lack of a
standardized mechanism for communicating this intent between servers
and clients.

This document introduces the uncacheable file data attribute to
NFSv4.2.  This OPTIONAL attribute allows a server to indicate that
client-side caching of file data for a particular file is unsuitable.
When both the client and the server support this attribute, the
client is advised to suppress client-side caching of file data for
that file, in accordance with the semantics defined in this document.

The uncacheable file data attribute is read-write, applies on a
per-file basis, and has a data type of boolean.

Support for the uncacheable file data attribute is specific to the
exported filesystem and may differ between filesystems served by the
same server.  A client can determine whether the attribute is
supported for a given file by examining the supported_attrs attribute
for that file's filesystem or by probing support using the procedures
described in {{RFC8178}}.

The uncacheable file data attribute applies only to regular files
(NF4REG).  Attempts to query or set this attribute on objects of
other types MUST result in an error of NFS4ERR_INVAL. Since the
uncacheable file data attribute applies only to regular files,
attempts to apply it to other object types represent an invalid use
of the attribute.

Using the process described in {{RFC8178}}, the revisions in this
document extend NFSv4.2 {{RFC7862}}.  They are built on top of the
external data representation (XDR) {{RFC4506}} generated from
{{RFC7863}}.

## Definitions

client-side caching of file data

: The retention of file data by a client in a local data cache, commonly
  referred to as the page cache, for the purpose of satisfying subsequent
  READ requests or delaying transmission of WRITE data to the server.

write-behind caching

: A form of file data caching in which WRITE data is retained by the
  client and transmission of the data to the server is delayed in order
  to combine multiple WRITE operations or improve efficiency.

direct I/O

: An access mode in which file data is transferred between application
  buffers and the underlying storage without populating or consulting
  the client's file data cache.  Direct I/O suppresses both read caching
  and write-behind caching of file data.

write hole

: A write hole is an instance of data corruption that arises when
  multiple clients modify disjoint byte ranges within the same encoded
  data block without having a consistent view of the existing contents.
  This can result in stale data overwriting newer updates, particularly
  in environments that use erasure encoding or striped storage.
  (Adapted from {{I-D.haynes-nfsv4-flexfiles-v2}}.)

This document assumes familiarity with the NFSv4 protocol operations,
error codes, object types, and attributes as defined in {{RFC8881}}.

## Requirements Language

{::boilerplate bcp14-tagged}


# Client-Side Caching of File Data

The uncacheable file data attribute advises the client to limit the
use of client-side caching of file data for a file. This includes
both write-behind caching and read caching, which are addressed
separately below.

The intent of this attribute is to allow a server or administrator
to indicate that client-side caching of file data for a particular
file is unsuitable. The server is often in a better position than
individual clients to determine sharing patterns, access behavior,
or correctness requirements associated with a file. By exposing
this information via an attribute, the server can advise clients
to limit file data caching in a consistent manner.

## Write-Behind Caching

The uncacheable file data attribute inhibits write-behind caching,
in which multiple pending WRITEs are combined and transmitted to
the server at a later time for efficiency.

When honoring the uncacheable file data attribute, clients SHOULD
NOT delay transmission of WRITE data for the purpose of combining
multiple WRITE operations or improving efficiency.

One important use case for this attribute arises in connection with
High-Performance Computing (HPC) workloads. These workloads often
involve concurrent writers modifying disjoint byte ranges of shared
files.

When application data spans a data block in a client cache, delayed
transmission of WRITE data can result in clients modifying stale
data and overwriting updates written by others. Prompt transmission
of WRITE data enables the prompt detection of write holes and reduces
the risk of data corruption.

## Read Caching

The uncacheable file data attribute may also influence the use of
read caching. Retaining cached READ data while other clients
concurrently modify disjoint byte ranges of the same file can result
in read-modify-write operations based on stale data.

Clients SHOULD ensure that cached file data is not reused without
first validating that the file has not changed.

At a minimum, clients MUST revalidate metadata necessary to ensure
correctness of cached file data, including the change attribute and
file size. These attributes provide the primary mechanism for
detecting modification of file contents.

Clients MAY revalidate additional attributes (e.g., modification
time or change time) as required by their local semantics or
application requirements.

Failure to perform such revalidation can result in the client
presenting stale or inconsistent file state (e.g., incorrect size
or timestamps) to the application.

Suppressing read caching in addition to suppressing write-behind
caching can further reduce the risk of stale-data overwrite in
multi-writer workloads. However, in some cases read caching may
remain appropriate when another NFSv4.2 mechanism ensures a
consistent view of the file, such as a delegation.

## Relationship to Direct I/O

While similar in intent to O_DIRECT ({{OPEN-O_DIRECT}}) and
forcedirectio ({{SOLARIS-FORCEDIRECTIO}}), the uncacheable file
data attribute operates at the protocol level and is advisory.
Clients retain flexibility in how they satisfy the requirements
described above.

# Setting the Uncacheable File Data Attribute {#sec_setting}

The uncacheable file data attribute provides a mechanism by which
a server or administrator can indicate that client-side caching of
file data for a file is unsuitable.

In some deployments, applications or administrative tools may request
that this attribute be set on a file in order to influence client
behavior. For example, applications that require predictable data
visibility or that would otherwise rely on mechanisms such as
O_DIRECT may use this attribute as a protocol-visible hint to the
server.

However, the setting of this attribute is subject to server policy.
The server is responsible for determining whether a request to set
or clear the attribute is permitted. This may depend on factors
such as administrative configuration, export policy, or access
control mechanisms.

Requests that are not permitted MUST be rejected using existing
NFSv4 error codes (e.g., NFS4ERR_INVAL or NFS4ERR_PERM).

One possible deployment model is for a server or administrator to
configure a mount (see {{MOUNT}}) option such that newly created
files under a given export are marked as uncacheable file data. In
such a configuration, a client may request setting of the attribute
at file creation time (e.g., via CREATE or OPEN createattrs).

This approach is conceptually similar in intent to the Solaris
forcedirectio mount option (see {{SOLARIS-FORCEDIRECTIO}}), but
differs in scope and visibility in that it allows DIRECT-I/O-like
behavior to be applied without requiring changes to individual
applications.

Unlike local mechanisms such as forcedirectio, the NFSv4.2 attribute
is visible to all clients accessing the file and is intended to
convey server-side knowledge or policy in a distributed environment.

Changes to the uncacheable file data attribute while a file is
actively in use may not be immediately reflected in client behavior.
A client that has already opened a file MAY continue to operate
based on its existing caching behavior and is not required to
immediately alter its behavior in response to a change in the
attribute.

Clients are expected to observe attribute changes through normal
NFSv4 mechanisms (e.g., GETATTR or revalidation) and apply updated
behavior as appropriate for subsequent operations.

# Implementation Status

Note to RFC Editor: please remove this section prior to publication.

There is a prototype Hammerspace server which implements the
uncacheable file data attribute and a prototype Linux client which
treats the attribute as an indication to use O_DIRECT-like behavior
for file access and to revalidate file-associated metadata before
exposing cached state.

For the prototype, all files created under the mount
point have the fattr4_uncacheable_file_data set to be true.

Experience with the prototype indicates that the uncacheable file
data attribute can provide many of the practical benefits of O_DIRECT
without requiring application modification. For applications that
issue well-formed I/O requests, this approach has been observed to
improve performance in many cases, while also reducing memory
pressure and CPU utilization in the NFS client.

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

The uncacheable file data attribute does not introduce new
authentication or authorization mechanisms and does not alter
existing NFSv4.2 access control semantics. All operations that set
or clear the attribute are subject to existing access control and
server policy.

In particular, a server MUST enforce appropriate authorization
checks for SETATTR operations that modify the fattr4_uncacheable_file_data
attribute. The ability to set or clear the attribute may be restricted
based on administrative configuration, export policy, or other
server-defined criteria.

Because the attribute is visible to and may affect the behavior of
multiple clients, servers SHOULD consider the implications of
allowing unprivileged users to modify it. Inappropriate use of the
attribute could impact performance or data access patterns for other
clients accessing the same file.

The uncacheable file data attribute is advisory and does not provide
a security boundary. Clients MUST NOT rely on the presence or absence
of this attribute to make access control decisions.

Use of this attribute does not replace or modify existing cache
consistency mechanisms or data integrity protections provided by
NFSv4.2.

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
