Title: Surviving a Compromise of PyPI: The Maximum Security Model
Version: $Revision$
Last-Modified: $Date$
Author: Trishank Karthik Kuppusamy <trishank@nyu.edu>,
Donald Stufft <donald@stufft.io>, Justin Cappos <jcappos@nyu.edu>,
Vladimir Diaz <vladimir.v.diaz@gmail.com>

BDFL-Delegate: Nick Coghlan <ncoghlan@gmail.com>
Discussions-To: DistUtils mailing list <distutils-sig@python.org>
Status: Draft
Type: Standards Track
Content-Type: text/x-rst
Created: 8-Oct-2014
Replaces:  458 


Abstract
========

Proposed is an extension to PEP 458 that adds support for end-to-end signing
and the maximum security model.  End-to-end signing allows both PyPI and
developers to sign for the distributions that are downloaded by clients.  The
minimum security model proposed by PEP 458 supports continuous delivery of
distributions (because they are signed by online keys), but that model does not
protect distributions in the event that PyPI is compromised.  The maximum
security model retains many of the benefits of PEP 458 (e.g., immediate
availability of distributions that are uploaded to PyPI) and additionally aims
to ensure that PyPI can recover from a key compromise.

This PEP includes the changes made to PEP 458 but excludes its informational
elements to primarily focus on the maximum security model.  For example, an
overview of The Update Framework is not covered here.  The changes to PEP 458
include modifications to the snapshot process, key compromise analysis,
auditing snapshots, and the steps that should be taken in the event of a PyPI
compromise.  The signing and key management process followed by developers that
PyPI MAY RECOMMEND is discussed but not strictly defined.  How the release
process should be implemented to manage keys and metadata is left to the
implementors of the signing tools.  That is, this PEP delineates the expected
cryptographic key type and signature format included in metadata that MUST be
uploaded by developers in order to support end-to-end verification of
distributions.


Rationale
=========

PEP 458 [1]_ proposes how PyPI should be integrated with The Update Framework
(TUF) [2]_.  It explains how modern package managers like pip can be made more
secure, and the types of attacks that could be prevented if PyPI were modified
on the server side to include TUF metadata.  Package managers can
reference the TUF metadata available on PyPI to download distributions more
securely.

PEP 458 also describes the metadata layout of the PyPI repository and the
minimum security model.  Although the minimum security model protects against
most software update attacks, such as mix-and-match and extraneous dependencies
attacks, it can be improved to also support end-to-end signing and to prohibit
forged distributions if PyPI is compromised.

The minimum security model supports continuous delivery of projects and uses
online cryptographic keys to sign the distributions uploaded by projects.  The
main strength of the minimum security model is the automated and simplified
release process: developers may upload distributions and then have PyPI sign
for their distributions.  Much of the release process is handled in an
automated fashion by online roles and this simplified approach requires that
cryptographic signing keys be stored on the PyPI infrastructure.
Unfortunately, cryptographic keys that are stored online are vulnerable to
theft, and thus distributions that are signed by these keys can be easily
forged if attackers compromise the servers that sign for distributions.

The maximum security model is an extension to the minimum model that allows
PyPI to survive a repository compromise and permits developers to sign for the
distributions that they make available to PyPI users.  The maximum security
provides additional protections while still supporting continuous delivery of
distributions.  However, for the following reasons, it is postponed and
covered in this PEP instead of PEP 458:

1.  A build farm (distribution wheels on supported platforms are generated on
    PyPI infrastructure for each project) may possibly complicate matters.
    PyPI wants to support a build farm in the future.  Unfortunately, if wheels
    are auto-generated externally, developer signatures for these wheels are
    unlikely.  However, there might still be a benefit to generating wheels
    from source distributions that are signed by developers (provided that
    reproducible wheels are possible).  Another possibility is to optionally
    delegate trust of these wheels to an online role.

2.  An easy-to-use key management solution is needed for developers.
    miniLock is one likely candidate for management and generation of keys.
    Although developer signatures can remain optional, this approach may be
    inadequate due to the great number of potentially unsigned dependencies for
    distributions a client may request.  Requiring developers to manually sign
    distributions and manage keys is expected to render key signing an unused
    feature.

3.  A two-phase approach, where the minimum security model is implemented
    before the maximum security model, will simplify matters and give PyPI
    administrators time to review the feasibility of end-to-end signing.


Threat Model
============

The threat model assumes the following:

* Offline keys are safe and securely stored.

* Attackers can compromise at least one of PyPI's trusted keys that are stored
  online, and may do so at once or over a period of time.

* Attackers can respond to client requests.

Attackers are considered successful if they can cause a client to install (or
leave installed) something other than the most up-to-date version of the

software the client is updating. When an attacker is preventing the
installation of updates, the attacker's goal is that clients *not* realize that
anything is wrong. 


Definitions
===========

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in RFC `2119`__.

__ http://www.ietf.org/rfc/rfc2119.txt

This PEP focuses on integrating TUF with PyPI; however, the reader is
encouraged to read about TUF's design principles [2]_.  It is also RECOMMENDED
that the reader be familiar with the TUF specification [3]_, and PEP 458 [1]_
(which this PEP is extending).

Terms used in this PEP are defined as follows:

* Projects: Projects are software components that are made available for
  integration.  Projects include Python libraries, frameworks, scripts,
  plugins, applications, collections of data or other resources, and various
  combinations thereof.  Public Python projects are typically registered on the
  Python Package Index [4]_.

* Releases: Releases are uniquely identified snapshots of a project [4]_.

* Distributions: Distributions are the packaged files that are used to publish

* Simple index: The HTML page that contains internal links to the
  distributions of a project [4]_.

* Roles: There is one *root* role in PyPI.  There are multiple roles whose
  responsibilities are delegated to them directly or indirectly by the *root*
  role. The term top-level role refers to the *root* role and any role
  delegated by the *root* role. Each role has a single metadata file that it is
  trusted to provide.

* Metadata: Metadata are files that describe roles, other metadata, and target
  files.

* Repository: A repository is a resource comprised of named metadata and target
  files.  Clients request metadata and target files stored on a repository.

* Consistent snapshot: A set of TUF metadata and PyPI targets that capture the
  complete state of all projects on PyPI as they existed at some fixed point in
  time.

* The *snapshot* (*release*) role: In order to prevent confusion due to the
  different meanings of the term "release" used in PEP 426 [1]_ and the TUF
  specification [3]_, the *release* role is renamed to the *snapshot* role.
  
* Developer: Either the owner or maintainer of a project who is allowed to
  update TUF metadata as well as distribution metadata and files for a given
  project. 

* Online key: A private cryptographic key that MUST be stored on the PyPI
  server infrastructure.  This usually allows automated signing with the key.
  An attacker who compromises the PyPI infrastructure will be able to
  immediately read these keys.

* Offline key: A private cryptographic key that MUST be stored independent of
  the PyPI server infrastructure.  This prevents automated signing with the
  key.  An attacker who compromises the PyPI infrastructure will not be able to
  immediately read these keys.

* Threshold signature scheme: A role can increase its resilience to key
  compromises by specifying that at least t out of n keys are REQUIRED to sign
  its metadata.  A compromise of t-1 keys is insufficient to compromise the
  role itself.  Saying that a role requires (t, n) keys denotes the threshold
  signature property.


Maximum Security Model
======================

The maximum security model permits developers to sign their projects and upload
signed metadata to PyPI.  If the PyPI infrastructure were to be compromised,
attackers would be unable to serve malicious versions of a *claimed* project
without having access to that project's developer key.  Figure 1 depicts the
changes made to the metadata layout of the minimum security model, namely that
developer roles are now supported and that three new delegated roles exist:
*claimed*, *recently-claimed*, and *unclaimed*.  The *bins* role has been
renamed *unclaimed* and can contain any projects that have not been added to
*claimed*.  The *unclaimed* role functions just as before (i.e., as explained
in PEP 458, projects added to this role are signed by PyPI with an online key).
Offline keys provided by developers ensure the strength of the maximum security
model over the minimum model.  Although the minimum security model supports
continuous delivery of projects, all projects are signed by an online key.
That is, an attacker is able to corrupt packages in the minimum security model,
but not in the maximum model, without also compromising a developer's key.

.. image:: figure1.png

Figure 1: An overview of the metadata layout in the maximum security model.
The maximum security model supports continuous delivery and survivable key
compromise.

Projects that are signed by developers and uploaded to PyPI for the first time
are added to the *recently-claimed* role.  The *recently-claimed* role uses an
online key, so projects uploaded for the first time are immediately available
to clients.  After some time has passed, PyPI administrators MAY move projects
listed in *recently-claimed* to the *claimed* role for maximum security.  The
*claimed* role uses an offline key, thus projects added to this role cannot be
easily forged if PyPI is compromised.


End-to-End Signing
==================

End-to-end signing allows both PyPI and developers to sign for the metadata
downloaded by clients.  PyPI is trusted to make uploaded projects available to
clients (PyPI signs the metadata for this part of the process), and developers
sign the distributions that they upload.


Metadata Signatures, Key Management, and Signing Distributions
==============================================================

This section discusses the tools, signature schemes, and signing methods that
PyPI MAY recommend to implementors of the signing tools.  Developers are
expected to use these tools to sign and upload distributions to PyPI.  To
summarize, developers MAY generate cryptographic keys and sign metadata in some
automated fashion, where the metadata includes the information required to
verify the authenticity of the distribution.  The metadata is then uploaded to
PyPI by the developer, where it will be available for download by package
managers such as pip (i.e., package managers that support TUF metadata).  The
entire process is transparent to end-users (using a package manager that
supports TUF) who download distributions from PyPI.


Cryptographic Signature Scheme: Ed25519
---------------------------------------

The package manager shipped with CPython (pip) MUST work on non-CPython
interpreters and cannot have dependencies that have to be compiled (i.e., the
PyPI+TUF integration MUST NOT require compilation of C extensions in order to
verify cryptographic signatures).  Verification of signatures MUST be done in
Python, and verifying RSA signatures in pure-Python may be impractical due to
speed.  Therefore, PyPI MAY use the `Ed25519`__ signature scheme.

__ http://ed25519.cr.yp.to/

Ed25519 is a public-key signature system that uses small cryptographic
signatures and keys.  A `pure-Python implementation` of the Ed25519 signature
scheme is available.  Verification of Ed25519 signatures is fast, even when
performed in Python.

__ https://github.com/pyca/ed25519


Cryptographic Key Files 
-----------------------

The implementation MAY encrypt key files with AES-256-CTR-Mode and strengthen
passwords with PBKDF2-HMAC-SHA256 (100K iterations by default, but this may be
overridden by the developer). The current Python implementation of TUF can use
any cryptographic library (support for PyCA cryptography will be added in the
future), may override the default number of PBKDF2 iterations, and the KDF
tweaked to taste.


Key Management: miniLock
------------------------

A key management solution that uses miniLock derives a private key from a
password so that developers do not have to manage cryptographic key files
across multiple computers.  Developers may view the cryptographic key as a
secondary password.  miniLock also works well with a signature scheme like
Ed25519, which only needs a very small key.

__ https://github.com/kaepora/miniLock#-minilock


Third-party Upload Tools: Twine
-------------------------------

Third-party tools like `Twine`__ may be modified (if they wish to support
distributions that include TUF metadata) to sign and upload developer projects
to PyPI.  Twine is a utility for interacting with PyPI that uses TLS to upload
distributions, and prevents MITM attacks on usernames and passwords.

__ https://github.com/pypa/twine


Distutils
---------

__ https://docs.python.org/2/distutils/index.html#distutils-index

Distutils MAY be modified to sign metadata and to upload signed distributions
to PyPI.  Distutils comes packaged with CPython and is the most widely-used
tool for uploading distributions to PyPI.


Automated Signing Solution
--------------------------

A default PyPI-mediated key management and package signing solution that is
transparent to developers and does not require a key escrow (sharing encrypted
private keys with PyPI) is RECOMMENDED for the signing tools.  Additionally,
the signing tools SHOULD also circumvent the sharing of encrypted private keys
between the multiple machines of each developer.

The following outlines the automated signing solution that a developer MAY
follow to upload a distribution to PyPI:

1.  Register project.
2.  Enter secondary password.
3.  Add new identity to user account from machine 2 (after a password prompt).
4.  Upload project.

Under the hood (the developer is not aware or needs to care that packages are
automatically signed):

The "create an identity with only a password" action generates an encrypted
private key file that uploads the ed25519 public key to PyPI.  An existing
identity (its public key is contained in project metadata or on PyPI) signs
(this is done transparently) for new identities.  By default, project metadata
has a signature threshold of 1 and other verified identities may create new
releases to satisfy the threshold.

However, the signing tools should be flexible; a single project key may also be
shared between multiple machines if manual key management is preferred (e.g.,
ssh-copy-id).

The current TUF `repository`__ and `developer`__ tools are available for
review:

__ https://github.com/theupdateframework/tuf/blob/develop/tuf/README.md
__ https://github.com/theupdateframework/tuf/blob/develop/tuf/README-developer-tools.md

The two signing tools above currently support all of the recommendations
previously mentioned except for the automated signing solution, which must be
added to Distutils, Twine, and other third-party signing tools.  The automated
signing solution simply calls available repository tool functions to sign
metadata and generate the cryptographic key files.

Producing Consistent Snapshots
------------------------------

PyPI is responsible for updating, depending on the project, either the
*claimed*, *recently-claimed*, or *unclaimed* metadata as well as associated
delegated metadata metadata. Every project MUST upload its set of metadata and
targets in a single transaction.  The uploaded set of files is called the
"project transaction."  How PyPI MAY validate files in a project transaction is
discussed in a later section.  The focus of this section is on how PyPI will
respond to a project transaction.

Every metadata and target file MUST include in its filename the `hex digest`__
of its `SHA-256`__ hash.  For this PEP, it is RECOMMENDED that PyPI adopt a
simple convention of the form: digest.filename, where filename is the original
filename without a copy of the hash, and digest is the hex digest of the hash.

__ http://docs.python.org/2/library/hashlib.html#hashlib.hash.hexdigest
__ https://en.wikipedia.org/wiki/SHA-2

When an unclaimed project uploads a new transaction, a project transaction
process MUST add all new targets and relevant delegated unclaimed metadata. (We
describe later in this section why the unclaimed role will delegate targets to
a number of delegated unclaimed roles.) Finally, the project transaction
process MUST inform the consistent snapshot process about new delegated
unclaimed metadata.

When a recently-claimed project uploads a new a transaction, a project
transaction process MUST add all new targets and delegated targets metadata for
the project. If the project is new, then the project transaction process MUST
also add new recently-claimed metadata with the public keys and threshold
number (which MUST be part of the transaction) for the project. Finally, the
project transaction process MUST inform the consistent snapshot process about
new recently-claimed metadata as well as the current set of delegated targets
metadata for the project.

The transaction process for a claimed project is slightly different in that
PyPI administrators will choose to move the project from the *recently-claimed*
role to the *claimed* role. A project transaction process MUST then add new
recently-claimed and claimed metadata to reflect this migration. As is the case
for a recently-claimed project, the project transaction process MUST always add
all new targets and delegated targets metadata for the claimed project.
Finally, the project transaction process MUST inform the consistent snapshot
process about new recently-claimed or claimed metadata as well as the current
set of delegated targets metadata for the project.

Project transaction processes SHOULD be automated, except when PyPI
administrators move a project from the recently-claimed role to the claimed
role. Project transaction processes MUST also be applied atomically: either all
metadata and targets -- or none of them -- are added. The project transaction
processes and consistent snapshot process SHOULD work concurrently. Finally,
project transaction processes SHOULD keep in memory the latest claimed,
recently-claimed, and unclaimed metadata so that they will be correctly updated
in new consistent snapshots.

All project transactions MAY be placed in a single queue and processed
serially.  Alternatively, the queue MAY be processed concurrently in order of
appearance, provided that the following rules are observed:

1.  No pair of project transaction processes must concurrently work on the same
    project.

2.  No pair of project transaction processes must concurrently work on
    *unclaimed* projects that belong to the same delegated *unclaimed* role.

3.  No pair of project transaction processes must concurrently work on new
    recently-claimed projects.

4.  No pair of project transaction processes must concurrently work on new
    claimed projects.

5.  No project transaction process must work on a new claimed project while
    another project transaction process is working on a new recently-claimed
    project and vice versa.

These rules MUST be observed to ensure that metadata is not read from or
written to inconsistently.


Snapshot Process
----------------

The snapshot process is fairly simple and SHOULD be automated.  The snapshot
process MUST keep in memory the latest working set of *root*, *targets*, and
delegated roles.  Every minute or so the snapshot process will sign for this
latest working set.  (Recall that project transaction processes continuously
inform the snapshot process about the latest delegated metadata in a
concurrency-safe manner.  The snapshot process will actually sign for a copy of
the latest working set while the latest working set in memory will be updated
with information that is continuously communicated by the project transaction
processes.)  The snapshot process MUST generate and sign new *timestamp*
metadata that will vouch for the metadata (*root*, *targets*, and delegated
roles) generated in the previous step.  Finally, the snapshot process MUST make
available to clients the new *timestamp* and *snapshot* metadata representing
the latest snapshot.


A claimed or recently-claimed project will need to upload in its transaction to
PyPI not just targets (a simple index as well as distributions) but also TUF
metadata. The project MAY do so by uploading a ZIP file containing two
directories, /metadata/ (containing delegated targets metadata files) and
/targets/ (containing targets such as the project simple index and
distributions which are signed for by the delegated targets metadata).

Whenever the project uploads metadata or targets to PyPI, PyPI SHOULD check the
project TUF metadata for at least the following properties:

* A threshold number of the developers keys registered with PyPI by that
  project MUST have signed for the delegated targets metadata file that
  represents the "root" of targets for that project (e.g. metadata/targets/
  project.txt).
* The signatures of delegated targets metadata files MUST be valid.
* The delegated targets metadata files MUST NOT be expired.
* The delegated targets metadata MUST be consistent with the targets.
* A delegator MUST NOT delegate targets that were not delegated to itself by
  another delegator.
* A delegatee MUST NOT sign for targets that were not delegated to itself by a
  delegator.
* Every file MUST contain a unique copy of its hash in its filename following
  the filename.digest.ext convention recommended earlier.

If PyPI chooses to check the project TUF metadata, then PyPI MAY choose to
reject publishing any set of metadata or targets that do not meet these
requirements.

PyPI MUST enforce access control by ensuring that each project can only write
to the TUF metadata for which it is responsible. It MUST do so by ensuring that
project transaction processes write to the correct metadata as well as correct
locations within those metadata. For example, a project transaction process for
an unclaimed project MUST write to the correct target paths in the correct
delegated unclaimed metadata for the targets of the project.

On rare occasions, PyPI MAY wish to extend the TUF metadata format for projects
in a backward-incompatible manner. Note that PyPI will NOT be able to
automatically rewrite existing TUF metadata on behalf of projects in order to
upgrade the metadata to the new backward-incompatible format because this would
invalidate the signatures of the metadata as signed by developer keys.
Instead, package managers SHOULD be written to recognize and handle multiple
incompatible versions of TUF metadata so that claimed and recently-claimed
projects could be offered a reasonable time to migrate their metadata to newer
but backward-incompatible formats.

The details of how each project manages its TUF metadata is beyond the scope of
this PEP.

A few implementation notes are now in order.  So far, we have seen that only
new metadata and targets are added, but not that old metadata and targets are
removed.  Practical constraints are such that eventually PyPI will run out of
disk space to produce a new consistent snapshot.  If that happens, PyPI MAY
then use something like a "mark-and-sweep" algorithm to delete sufficiently old
consistent snapshots. Specifically, in order to preserve the latest consistent
snapshot, PyPI would walk objects -- beginning from the root (*timestamp*) --
of the latest consistent snapshot, mark all visited objects, and delete all
unmarked objects.  The last few consistent snapshots may be preserved in a
similar fashion.  Deleting a consistent snapshot will cause clients to see
nothing except HTTP 404 responses to any request for a target of the deleted
consistent snapshot.  Clients SHOULD then retry (as before) their requests with
the latest consistent snapshot.

All package managers that support TUF metadata MUST be modified to download
every metadata and target file (except for *timestamp* metadata) by including,
in the request for the file, the cryptographic hash of the file in the
filename.  Following the filename convention recommended earlier, a request for
the file at filename.ext will be transformed to the equivalent request for the
file at digest.filename.

Finally, PyPI SHOULD use a `transaction log`__ to record project transaction
processes and queues so that it will be easier to recover from errors after a
server failure.

__ https://en.wikipedia.org/wiki/Transaction_log


Key Compromise Analysis
=======================

This PEP has covered the maximum security model, the TUF roles that should be
added to support continuous delivery of distributions, how to generate and sign
the metadata of each role, and how to support distributions that have been
signed by developers.  The remaining sections discuss how PyPI SHOULD audit
repository metadata and the methods PyPI can use to detect and recover from a
PyPI compromise.

Table 1 summarizes a few of the attacks possible when a threshold number of
private cryptographic keys (belonging to any of the PyPI roles) are
compromised.  The leftmost column lists the roles (or a combination of roles)
that have been compromised, and the columns to the right show whether the
compromised roles leaves clients susceptible to malicious updates, freeze
attacks, or metadata inconsistency attacks.

+-------------------+-------------------+-----------------------+-----------------------+
| Role Compromise   | Malicious Updates | Freeze Attack         | Metadata Inconsistency|
|                   |                   |                       | Attacks               |
+===================+===================+=======================+=======================+
|    timetamp       |       NO          |       YES             |       NO              |
|                   | snapshot and      | limited by earliest   | snapshot needs to     |
|                   | targets or any    | root, targets, or bin | cooperate             |
|                   | of the delegated  | metadata expiry time  |                       |
|                   | roles need to     |                       |                       |
|                   | cooperate         |                       |                       |
+-------------------+-------------------+-----------------------+-----------------------+
|    snapshot       |       NO          |         NO            |       NO              |
|                   | timestamp and     | timestamp needs to    | timestamp needs to    |
|                   | targets or any of | coorperate            | cooperate             |
|                   | the delegated     |                       |                       |
|                   | roles need to     |                       |                       |
|                   | cooperate         |                       |                       |
+-------------------+-------------------+-----------------------+-----------------------+
|    timestamp      |       NO          |         YES           |       YES             |
|    **AND**        | targets or any    | limited by earliest   | limited by earliest   |
|    snapshot       | of the delegated  | root, targets, or bin | root, targets, or bin |
|                   | roles need to     | metadata expiry time  | metadata expiry time  |
|                   | cooperate         |                       |                       |
|                   |                   |                       |                       |
+-------------------+-------------------+-----------------------+-----------------------+
|    targets        |       NO          |     NOT APPLICABLE    |    NOT APPLICABLE     |
|    **OR**         | timestamp and     | need timestamp and    | need timestamp        |
|    claimed        | snapshot need to  | snapshot              | and snapshot          |
|    **OR**         | cooperate         |                       |                       |
| recently-claimed  |                   |                       |                       |
|    **OR**         |                   |                       |                       |
|    unclaimed      |                   |                       |                       |
|    **OR**         |                   |                       |                       |
|    project        |                   |                       |                       |
+-------------------+-------------------+-----------------------+-----------------------+
|   (timestamp      |       YES         |       YES             |       YES             |
|   **AND**         |                   | limited by earliest   | limited by earliest   |
|   snapshot)       |                   | root, targets, or bin | root, targets, or bin |
|   **AND**         |                   | metadata expiry time  | metadata expiry time  |
|   project         |                   |                       |                       |
|                   |                   |                       |                       |
+-------------------+-------------------+-----------------------+-----------------------+
|  (timestamp       |     YES           |        YES            |           YES         |
|  **AND**          | but only of       | limited by earliest   | limited by earliest   |
|  snapshot)        | projects not      | root, targets,        | root, targets,        |
|  **AND**          | delegated by      | claimed,              | claimed,              |
| (recently-claimed | claimed           | recently-claimed,     | recently-claimed,     |
| **OR**            |                   | project, or unclaimed | project, or unclaimed |
| unclaimed)        |                   | metadata expiry time  | metadata expiry time  |
+-------------------+-------------------+-----------------------+-----------------------+
| (timestamp        |                   |         YES           |           YES         | 
| **AND**           |                   | limited by earliest   | limited by earliest   |   
| snapshot)         |                   | root, targets,        | root, targets,        |
| **AND**           |       YES         | claimed,              | claimed,              |
| (targets **OR**   |                   | recently-claimed,     | recently-claimed,     |
| claimed)          |                   | project, or unclaimed | project, or unclaimed |
|                   |                   | metadata expiry time  | metadata expiry time  |
+-------------------+-------------------+-----------------------+-----------------------+
|     root          |       YES         |         YES           |           YES         |
+-------------------+-------------------+-----------------------+-----------------------+

Table 1: Attacks that are possible by compromising certain combinations of role keys.
In `September 2013`__, it was shown how the latest version (at the time) of pip
was susceptible to these attacks and how TUF could protect users against them
[8]_.

__ https://mail.python.org/pipermail/distutils-sig/2013-September/022755.html

Note that compromising *targets* or any delegated role (except for project
targets metadata) does not immediately allow an attacker to serve malicious
updates.  The attacker must also compromise the *timestamp* and *snapshot*
roles (which are both online and therefore more likely to be compromised).
This means that in order to launch any attack, one must not only be able to
act as a man-in-the-middle but also compromise the *timestamp* key (or
compromise the *root* keys and sign a new *timestamp* key).  To launch any
attack other than a freeze attack, one must also compromise the *snapshot* key.

Finally, a compromise of the PyPI infrastructure MAY introduce malicious
updates to *bins* projects because the keys for these roles are online.  The
maximum security model discussed in the appendix addresses this issue.  PEP XXX
[VD: Link to PEP once it is completed] also covers the maximum security model
and goes into more detail on generating developer keys and signing uploaded
distributions.


In the Event of a Key Compromise
--------------------------------

A key compromise means that a threshold of keys (belonging to the metadata
roles on PyPI), as well as the PyPI infrastructure, have been compromised and
used to sign new metadata on PyPI.

If a threshold number of developer keys of a project have been compromised,
the project MUST take the following steps:

1.  The project metadata and targets MUST be restored to the last known good
    consistent snapshot where the project was not known to be compromised. This
    can be done by developers repackaging and resigning all targets with
    the new keys.

2.  The project's metadata MUST have its version numbers incremented, expiry
    times suitably extended, and signatures renewed.

Whereas PyPI MUST take the following steps:

1.  Revoke the compromised developer keys from the *recently-claimed* or
    *claimed* role.  This is done by replacing the compromised developer keys
    with newly issued developer keys.

2.  A new timestamped consistent snapshot MUST be issued.

If a threshold number of timestamp, snapshot, recently-claimed, or
unclaimed keys have been compromised, then PyPI MUST take the following steps:

1.  Revoke the timestamp, snapshot, and targets role keys from the
    root role. This is done by replacing the compromised timestamp,
    snapshot, and targets keys with newly issued keys.

2.  Revoke the recently-claimed and unclaimed keys from the targets role by
    replacing their keys with newly issued keys. Sign the new targets role
    metadata and discard the new keys (because, as we explained earlier, this
    increases the security of targets metadata).

3.  Clear all targets or delegations in the recently-claimed role and delete
    all associated delegated targets metadata. Recently registered projects
    SHOULD register their developer keys again with PyPI.

4.  All targets of the recently-claimed and unclaimed roles SHOULD be compared
    with the last known good consistent snapshot where none of the timestamp,
    snapshot, recently-claimed, or unclaimed keys were known to have been
    compromised. Added, updated, or deleted targets in the compromised
    consistent snapshot that do not match the last known good consistent
    snapshot MAY be restored to their previous versions. After ensuring the
    integrity of all unclaimed targets, the unclaimed metadata MUST be
    regenerated.

5.  The recently-claimed and unclaimed metadata MUST have their version numbers
    incremented, expiry times suitably extended, and signatures renewed.

6.  A new timestamped consistent snapshot MUST be issued.

This would preemptively protect all of these roles even though only one of them
may have been compromised.

If a threshold number of the targets or claimed keys have been compromised,
then there is little that an attacker would be able do without the timestamp and
snapshot keys. In this case, PyPI MUST simply revoke the compromised targets or
claimed keys by replacing them with new keys in the root and targets roles,
respectively.

If a threshold number of the timestamp, snapshot, and claimed keys have been
compromised, then PyPI MUST take the following steps in addition to the steps
taken when either the timestamp or snapshot keys are compromised:

1.  Revoke the claimed role keys from the targets role and replace them with
    newly issued keys.
    
2.  All project targets of the claimed roles SHOULD be compared with the last
    known good consistent snapshot where none of the timestamp, snapshot, or
    claimed keys were known to have been compromised.  Added, updated, or
    deleted targets in the compromised consistent snapshot that do not match
    the last known good consistent snapshot MAY be restored to their previous
    versions.  After ensuring the integrity of all claimed project targets, the
    claimed metadata MUST be regenerated.

3.  The claimed metadata MUST have their version numbers incremented, expiry
    times suitably extended, and signatures renewed.


Following these steps would preemptively protect all of these roles even though
only one of them may have been compromised.

If a threshold number of *root* keys have been compromised, then PyPI MUST take
the steps taken when the *targets* role has been compromised.  All of the
*root* keys must also be replaced.

It is also RECOMMENDED that PyPI sufficiently document compromises with
security bulletins.  These security bulletins will be most informative when
users of pip-with-TUF are unable to install or update a project because the
keys for the *timestamp*, *snapshot*, or *root* roles are no longer valid.  Users
could then visit the PyPI web site to consult security bulletins that would
help to explain why users are no longer able to install or update, and then take
action accordingly.  When a threshold number of *root* keys have not been
revoked due to a compromise, then new *root* metadata may be safely updated
because a threshold number of existing *root* keys will be used to sign for the
integrity of the new *root* metadata.  TUF clients will be able to verify the
integrity of the new *root* metadata with a threshold number of previously
known *root* keys.  This will be the common case.  In the worst
case, where a threshold number of *root* keys have been revoked due to a
compromise, an end-user may choose to update new *root* metadata with
`out-of-band`__ mechanisms.

__ https://en.wikipedia.org/wiki/Out-of-band#Authentication


Auditing Snapshots
------------------

If a malicious party compromises PyPI, they can sign arbitrary files with any
of the online keys.  The roles with offline keys (i.e., *root* and *targets*)
are still protected. To safely recover from a repository compromise, snapshots
should be audited to ensure that files are only restored to trusted versions.

When a repository compromise has been detected, the integrity of three types of
information must be validated:

1. If the online keys of the repository have been compromised, they can be
   revoked by having the *targets* role sign new metadata, delegated to a new
   key.

2. If the role metadata on the repository has been changed, this will impact
   the metadata that is signed by online keys.  Any role information created
   since the compromise should be discarded. As a result, developers of new
   projects will need to re-register their projects.

3. If the packages themselves may have been tampered with, they can be
   validated using the stored hash information for packages that existed in
   trusted metadata before the compromise.  Also, new distributions that are
   signed by developers in the claimed role may be safely retained.  However,
   any distributions signed by developers in the *recently-claimed* or
   *unclaimed* roles should be discarded.

In order to safely restore snapshots in the event of a compromise, PyPI SHOULD
maintain a small number of its own mirrors to copy PyPI snapshots according to
some schedule.  The mirroring protocol can be used immediately for this
purpose.  The mirrors must be secured and isolated such that they are
responsible only for mirroring PyPI.  The mirrors can be checked against one
another to detect accidental or malicious failures.

Another approach is to generate the cryptographic hash of *snapshot*
periodically and tweet it.  For example, upon receiving the tweet, a user comes
forward with the actual metadata and the repository maintainers are then able
to verify the metadata's cryptographic hash.  Alternatively, PyPI may
periodically archive its own versions of *snapshot* rather than rely on
externally provided metadata.  In this case, PyPI SHOULD take the cryptographic
hash of every package on the repository and store this data on an offline
device. If any package hash has changed, this indicates an attack has occurred.

Attacks that serve different versions of metadata or that freeze a version
of a package at a specific version can be handled by TUF with techniques
such as implicit key revocation and metadata mismatch detection [1].


References
==========

.. [1] https://www.python.org/dev/peps/pep-0458/
.. [2] https://isis.poly.edu/~jcappos/papers/samuel_tuf_ccs_2010.pdf
.. [3] https://github.com/theupdateframework/tuf/blob/develop/docs/tuf-spec.txt
.. [4] http://www.python.org/dev/peps/pep-0426/
.. [5] https://github.com/theupdateframework/pip/wiki/Attacks-on-software-repositories
.. [6] https://mail.python.org/pipermail/distutils-sig/2013-September/022773.html
.. [7] https://isis.poly.edu/~jcappos/papers/cappos_mirror_ccs_08.pdf
.. [8] https://mail.python.org/pipermail/distutils-sig/2013-September/022755.html
.. [9] https://pypi.python.org/security
.. [10] https://mail.python.org/pipermail/distutils-sig/2013-August/022154.html
.. [11] https://en.wikipedia.org/wiki/RSA_%28algorithm%29
.. [12] https://pypi.python.org/pypi/pycrypto
.. [13] http://ed25519.cr.yp.to/


Acknowledgements
================

This material is based upon work supported by the National Science Foundation
under Grant No. CNS-1345049 and CNS-0959138. Any opinions, findings, and
conclusions or recommendations expressed in this material are those of the
author(s) and do not necessarily reflect the views of the National Science
Foundation.

Nick Coghlan, Daniel Holth and the distutils-sig community in general for
helping us to think about how to usably and efficiently integrate TUF with
PyPI.

Roger Dingledine, Sebastian Hahn, Nick Mathewson,  Martin Peck and Justin
Samuel for helping us to design TUF from its predecessor Thandy of the Tor
project.

Konstantin Andrianov, Geremy Condra, Vladimir Diaz, Zane Fisher, Justin Samuel,
Tian Tian, Santiago Torres, John Ward, and Yuyu Zheng for helping us to develop
TUF.


Copyright
=========

This document has been placed in the public domain.