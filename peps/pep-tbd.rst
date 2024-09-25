PEP: TBD
Title: Default Extras for Python Packages
Author: Thomas Robitaille <thomas.robitaille@gmail.com>
Sponsor: TBD
Discussions-To: TBD
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 24-Sep-2024
Post-History: TBD

Abstract
========

`PEP 508 <https://peps.python.org/pep-0508/>`_ specifies a mini-language for
declaring package dependencies. One feature of this language is the ability to
specify 'extras', which are optional components of a distribution that, when
used, install additional dependencies. This PEP proposes a mechanism to allow
one or more extras to be included or "selected" by default, and defines a method
for these default extras to be "unselected" if desired.

Motivation
==========

Package maintainers often use extras to declare optional dependencies that extend
the functionality or performance of a package. In some cases, it can be difficult
to determine which dependencies should be required and which should be categorized
as extras. A balance must be struck between the needs of typical users (who may
prefer most features to be available 'by default') and users who want minimal
installations without large, optional dependencies.

One current solution is to define an extra called, for example, ``recommended``,
which includes all non-essential but suggested dependencies. Users are then told
to install the package using ``package[recommended]``, while those who prefer
more control can simply use ``package``. However, in practice, many users are
unaware of the ``recommended`` syntax, placing the burden on them to know this
for a typical installation.

This PEP proposes a method to specify one or more extras to be included by
default, while providing a way for users who prefer a minimal installation
to opt out of these extras.

Another motivating use case is for packages that support different backends or
frontends, where at least one must be installed for the package to be functional.
The proposed approach allows maintainers to define default extras for such cases,
installing a default backend or frontend while offering users the option to opt
out and install an alternative.

Specification
=============

Terminology
-----------

When installing a package, selecting optional dependencies (commonly known as
"extras") will now be referred to as *selecting* extras. For example, in
the following command, a user has *selected* the ``recommended`` extra::

    pip install package[recommended]

``Default-Extras`` Metadata Field
---------------------------------

A new metadata field, `Default-Extras`, will be added to the `core package
metadata <https://packaging.python.org/en/latest/specifications/core-metadata/#core-metadata>`_.
This field allows package maintainers to define one or more extras that are
automatically selected when a user installs the package without specifying any
extras. If multiple extras are specified, they should be separated by commas.

Example with a single default extra::

    Default-Extras: recommended

Example with multiple default extras::

    Default-Extras: recommended,pdf

Unselecting Default Extras
--------------------------

A new syntax for *unselecting* extras will be introduced as an extension of the
mini-language defined in `PEP 508 <https://peps.python.org/pep-0508/>`_. If a
package defines default extras, users can opt out of these defaults by using a
minus sign (`-`) before the extra name. The proposed syntax update is as follows::

    extras_list   = (-)?identifier (wsp* ',' wsp* (-)?identifier)*

If both an extra and its negated version appear in an extras list, the
non-negated extra should take precedence.

Attempting to unselect an extra that does not exist, or is not in the list of
default extras, should do nothing. The rationale for these rules is to avoid
having to build the package simply to validate the name of the extras.

Valid examples of the new syntax include:

* ``package[-recommended]``
* ``package[-backend1, backend2]``
* ``package[pdf, -svg]``
* ``package[pdf, -pdf]`` - same as ``package[pdf]``
* ``package[-nondefault, pdf]`` where ``nondefault`` is not in ``Default-Extras`` - equivalent to ``package[pdf]``
* ``package[-nonexistent, svg]`` where ``nonexistent`` is not defined as an extra - equivalent to ``package[svg]``

We note that unselecting with ``-`` should *not* be considered to be the same as
**requiring** that the dependencies are not present, since this would make it
impossible to interpret a dependency tree containing both ``package[-pdf]`` and
``package[pdf]``. Unselecting an extra is merely a way of saying that the extras
isn't needed, not that it has to be absent.

Backward Compatibility
======================

All package specification cases valid under `PEP 508 <https://peps.python.org/pep-0508/>`_
will remain valid. Therefore, this proposal is fully backward-compatible with
existing `PEP 508` usage.

Users will gain the ability to deselect default extras once a package defines
default extras and the package installation tools (e.g., pip) support the new syntax.

Implementation
==============

TBD - once we agree on the best path forward.

Rejected Alternatives
=====================

Adding a special entry in extras_require
----------------------------------------

A potential solution that has been explored is to make use of an extras with a
'special' name, for example an empty string:

    Provides-Extra:
    Requires-Dist: numpy ; extra == ''

The idea would be that dependencies installed as part of the 'empty' extras
would only get installed if another extras was not specified. However, there are
several issues with this approach:

* The syntax ``extra == ''`` does not fit in to how extras are evaluated in the
  context of environment markers, ``extra == 'pdf'`` does not mean that a user
  specified only the ``pdf`` extra but that ``pdf`` was one of the extras
  specified. In this context, ``extra == ''`` would not really behave the same,
  as it does not indicate that no extras were specified at all.
* The only way to not install the default empty string extras would be to select
  another extras - there would be no way to have a minimal installation with no
  extras at all.
* It might surprise some users that requesting e.g. ``package[pdf]`` may actually
  result in some dependencies no longer being installed (those that were part of
  the default extras)

This approach was one of those discussed extensively in
https://github.com/pypa/setuptools/issues/1139 and
https://github.com/pypa/setuptools/pull/1503.

``Default-Extras`` only apply if no other extras are specified
--------------------------------------------------------------

An alternative considered was that default extras would be specified as proposed
in this PEP, but the - syntax for unselecting dependencies would not be
introduced. Instead, default extras would apply only if no extras were
explicitly requested.

However, this would not be sufficient. As for the approach of the special entry
in ``extras_require``, there would be no way of removing default extras without
selecting a new extra, and therefore no way of doing a minimal installation, and
again it might surprising to users to have some dependencies no longer be
installed just because an extra was specified.

Relying on tooling to deselect any default extras
-------------------------------------------------

Another option to unselect extras would be to have this be implemented at the
level of packaging tools. For instance, pip could include an option such as::

    pip install package --no-default-extras

which would then require all desired extras to be explicitly specified. This
could also be made to be applicable to all or just specific packages, similar to
e.g. the ``--no-binary`` option, so::

    pip install package --no-default-extras :all:

or specifying specific packages to ignore the defaults from. The advantage of
this approach would be that it would be guaranteed that tools that support
installing the defaults also support unselecting them. This approach would be
similar to the ``--no-install-recommends`` option for the ``apt`` tool.

However, this solution is not ideal because it would not allow packages to
specify themselves that they do not need some of the default extras of a
dependency, and would carry a lot of risk for users who would simply disable all
default extras, potentially risking breaking packages in the dependency tree who
are relying on the default extras being present.

Having a way of disabling any default extra
-------------------------------------------

One idea that was raised was to allow a special way of specifying that all
default dependencies should be unselected, for example ``package[-*]``.
However, it was deemed that some package maintainers might over-use this option
to always reject default dependencies.

``package[]`` disables default extras
---------------------------------------

Another way to specify to not install any extras, including any default extras,
would be to use ``package[]``. However, this would break the current assumption that
``package[]`` is the same as ``package``, and may also (similarly to ``-*``) result
in developers over-using ``[]`` everywhere by default. This approach would also not
allow any extras to be installed while removing the default ones.

References
==========

.. [1] Support the /usr/bin/python2 symlink upstream (with bonus grammar class!)
   (https://mail.python.org/pipermail/python-dev/2011-March/108491.html)
