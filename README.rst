001: GitLFS Architecture Note
=============================

This is a SQuaRE Technical Note describing the architecture of the
GitLFS service implementation. For actual documentation on deployment,
as well as the source code involved in standing up this service,
consult the repositories.


Motivation
----------

Astronomical software code repositories often contain data for test or
demo purposes. These large binary files don't tend to benefit from
version control beyond provenance and update tracking, and their
history expands repository size with no return. Historically there
have been a number of services to deal with this, such as git-annex
and git-fat, but in terms of workflow, storing files in those back
ends was too much of a departure from what seems like "normal"
workflow for users. Lacking a satisfactory option, the core package
afwdata was left on the ingouse gitolite server after the rest of the
codebase migrated to Github. 

In 2015, Github released a protocol and an open source referebce
implementation of GitLFS, a specification for dealing with large
binary files in git. Aside from some upfront setup pain, the worflow
was very close to "normal" Github flow. Github also released a paid
hosted service for those files. Given the demand for storing data in
our repositories, the cost would be non-trivial. More, we did not wish
to get in a position where developers are self-censoring over what to
store.

Following a successful RFC, we decided to proceed with a GitLFS
service backed by our developer infrastructure OpenStack resources at
NCSA's Nebula cluster. This would give users the advantages of working
predominantly with the Github services, while allowing us to offer
umetered storage at the back end. 

Challenges
----------

There were a number of challenges to overcome.

- As early GitLFS adopters, we encountered non-trivial bugs. Developer
  response was excellent, but we did have to delay deployment waiting
  for a fixed client for a better user experience.

- At the time, the Nebula cluster did not provide an object store, so
  we built our own with Ceph.

- There was also no disaster recovery backup service available, so we
  built our own backed by AWS S3 (we may move to Glacier if volumes
  get high).

- The S3 protocol implementation we used to back our server lacked
  Ceph support.

Architecture
------------

.. image:: https://github.com/lsst-sqre/technote-001/blob/master/gitlfs.png
   :alt: GitLFS Architecture Diagram

After installing the gitLFS client, a user who having cloned and
modified now pushes their GitLFSrepo is in fact generating two
requests when pushing a file specified in a repo's .gitattributes as
being tracked. The first one goes to the Github server and contains a
JSON packet that looks something like this:

.. code-block:: json
   version https://git-lfs.github.com/spec/v1
   oid sha256:7a6943ac4d8337727b93f410cf51b1ce748dabe9dc8e85c8942c97dd5c0a49e9
   size 123840

The second one is intercepted by the gitLFS client (due to the smudge
and clean filters set up) and uses the .gitconfig to locare the
location of the gitLFS server it should be addressing. In our case
that is git-lfs.lsst.codes. The gitLFS server queries the Github API
to ensure the user has org permissions (we can't let anyone on the
open Internet push to our server!). It checks that the requested blob
exists in the backing store, and hands the client a URL that it can
use to retrieve it. The client then fetches the URL to retrieve it
from our object store.

Other git-lfs.lsst.codes components

- Passenger is used to run the git-lfs-s3-server ruby gem
- Nginx is used for SSL termination.
- Redis is used for credential caching.
- s3s3 is our backup service backing onto AWS S3

The object store components are:

- the s3.lsst.codes head node, that provides the radosgw REST API
  server for Ceph.

- Our Ceph instance (speaking the S3 protocol, can be switched to
  Swift when that is available) that uses a 3-node (the quorum
  required by Ceph) configuration on ceph[0,1,2].lsst.codes.

All lsst.codes instances (in green in the diagram) are deployed on the
Nebula cluster at NCSA.
		 

Repositories
------------

These are the repos involved in this deployment:

- `git-lfs-s3-server <https://github.com/lsst-sqre/git-lfs-s3-server>`_

  This is the server implementation. It also contains the deploy
  instructions. 


- `s3s3 <https://github.com/lsst-sqre/s3s3>`_

  This is the code for the backup service. 
  
- `git-lfs-s3 <https://github.com/lsst-sqre/git-lfs-s3>`_

  This is a fork that we made of an LFS S3 implementation. We extended
  it because at the time of writing it lacked support for Ceph and for
  public repos (it only worked for private repos). We are working on
  upstreaming those changes, and when this is done, the private fork
  will be removed.

Documentation
-------------

- `afwdata's README <https://github.com/lsst/afwdata>`_

  Because this was the first (and main) repo transitioned,
  user-oriented notes can be found in afwdata's README. As this
  service is extended, it will be rehomed.

- `SQuaRE Technical Note 001`_

  The source for this note. 

RFCs
----

- `RFC-104 <https://jira.lsstcorp.org/browse/RFC-104>`

  This is the RFC proposing GitLFS adoption
