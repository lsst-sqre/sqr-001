:tocdepth: 1

.. sectnum::

.. Metadata such as the title, authors, and description are set in metadata.yaml

Abstract
========

A description of the architecture of the Git LFS service implementation

This is a SQuaRE Technical Note describing the architecture of the `Git LFS <https://git-lfs.github.com/>`_ service implementation. For actual
documentation on deployment, as well as the source code involved in
standing up this service, consult the :ref:`repositories <repos>`.

Motivation
==========

Astronomical software code repositories often contain data for test or
demo purposes. These large binary files don't tend to benefit from
version control beyond provenance and update tracking, and their
history expands repository size with no return. Historically there
have been a number of services to deal with this, such as `git-annex`_
and `git-fat`_, but in terms of workflow, storing files in those back
ends was too much of a departure from what seems like "normal"
workflow for users. Lacking a satisfactory option, the core package
`afwdata`_ was left on the in-house gitolite server after the rest of the
codebase migrated to GitHub.

.. _git-annex: https://git-annex.branchable.com
.. _git-fat: https://github.com/jedbrown/git-fat
.. _afwdata: https://github.com/lsst/afwdata

In 2015, `GitHub released a protocol and an open source reference
implementation of Git LFS <https://git-lfs.github.com>`_, a
specification for dealing with large binary files in git. Aside from
some upfront setup pain, the workflow was very close to "normal" GitHub
flow. GitHub also released a paid hosted service for those files. Given
the demand for storing data in our repositories, the cost would be
non-trivial. More, we did not wish to get in a position where developers
are self-censoring over what to store.  Finally, the largest a Git LFS
repository can be, if hosted at GitHub, is 5GB, no matter how much we'd
be willing to pay.

Following `a successful RFC
<https://jira.lsstcorp.org/browse/RFC-104>`_, we decided to proceed with
a Git LFS service. This would give users the advantages of working
predominantly with the GitHub services, while allowing us to offer
unmetered storage at the back end.

In 2023, we were faced with the facts that we did not want to store our
data at AWS anymore, and that the software we used for Git LFS was
dangerously ancient and unmaintained. After some consideration, we chose
a different product to provide Git LFS service, and proposed it in
`RFC-966 <https://jira.lsstcorp.org/browse/RFC-966>`_.  This document
now describes the current state of Git LFS at the Rubin Observatory.

Architecture
============

After `installing <https://git-lfs.github.com>`_ the ``git lfs`` client,
a user can commit additions and modifications to LFS-backed data using
the normal ``git`` commands. The repo's ``.gitattributes`` specifies
what files are tracked by Git LFS.

Our Git LFS server is hosted by `Roundtable
<https://roundtable.lsst.io>`_ and is protected by `Gafaelfawr
<https://gafaelfawr.lsst.io>`_. Before the user can commit Git
LFS-tracked data, they must first request a token with the
``write:git-lfs`` scope.  Having done this, they must update their
``~/.git-credentials`` file (or other Git credentials manager) with that
token, and update the Git LFS URL to
``https://git-lfs-rw.lsst.cloud/<org>/<repo>`` (where ``org`` is usually
``lsst`` and ``repo`` is the repository name, e.g. ``testdata_subaru``).

This process is described in the `Developer Guide
<https://developer.lsst.io/git/git-lfs.html>`_.

When the user pushes commits with
LFS-tracked data, they are in fact generating two requests. The first
one goes to the Git server containing the Git LFS pointer for the
file. It looks something like this:

.. code-block:: text

   version https://git-lfs.github.com/spec/v1
   oid sha256:7a6943ac4d8337727b93f410cf51b1ce748dabe9dc8e85c8942c97dd5c0a49e9
   size 123840

The second request is made by the ``git lfs`` client (due to the
smudge and clean filters) and uses the ``.lfsconfig`` to locate
the Git LFS server it should be addressing. In our case that will be
``git-lfs-rw.lsst.cloud``. The Git LFS server checks whether the requested
blob exists in the backing store, and hands the client a URL that it
can use to retrive or push it. The client then uses the URL to fetch or push to our object store.

Other ``git-lfs.lsst.cloud`` (and ``git-lfs-rw.lsst.cloud``) components:

- `Giftless <https://giftless.datopian.com>`_ provides the Git LFS
  server.

- `Google Cloud Storage <https://cloud.google.com/storage>`_ provides
  the backing store, using the s3 object storage protocol.

.. _repos:

Client Requirements
===================

It is recommended that users use the current stable ``git-lfs`` `client
version <https://github.com/git-lfs/git-lfs/releases/latest>`_
(``3.4.0`` at the time of writing).

Repositories
============

These are the repos involved in this deployment:

- `Phalanx <https://github.com/lsst-sqre/phalanx>`_.

  The application
  configuration is held in ``applications/giftless``.

- `Giftless repository <https://github.com/datopian/giftless>`_.

  This is the server implementation.  It uses Datopian's released
  `Giftless container <https://hub.docker.com/r/datopian/giftless>`_


.. _docs:

Documentation
=============

- `GitHub's Git Large File Storage website <https://git-lfs.github.com/>`_

  This is GitHub's git-lfs website and links to the canonical client
  source code, issues and documentation.

- `LSST's developer guide <http://developer.lsst.io/en/latest/tools/git_lfs.html>`_

  LSST's developer guide for using git-lfs.

- `SQuaRE Technical Note 001 <https://github.com/lsst-sqre/sqr-001>`_

  The source for this note.

RFCs
====

- `RFC-104 <https://jira.lsstcorp.org/browse/RFC-104>`_

  This is the RFC proposing Git LFS adoption.

- `RFC-966 <https://jira.lsstcorp.org/browse/RFC-966>`_

  This is the RFC updating the Git LFS architecture with Giftless (2023).

See the `reStructuredText Style Guide <https://developer.lsst.io/restructuredtext/style.html>`__ to learn how to create sections, links, images, tables, equations, and more.

.. Make in-text citations with: :cite:`bibkey`.
.. Uncomment to use citations
.. .. rubric:: References
..
.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
