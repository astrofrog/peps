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
specify *extras*, which are optional components of a distribution that, when
used, install additional dependencies. This PEP proposes a mechanism to allow
one or more extras to be included or "selected" by default and defines a method
for these default extras to be "unselected" if desired.

Motivation
==========

Package maintainers often use extras to declare optional dependencies that
extend the functionality or performance of a package. In some cases, it can be
difficult to determine which dependencies should be required and which should be
categorized as extras. A balance must be struck between the needs of typical
users (who may prefer most features to be available 'by default') and users who
want minimal installations without large, optional dependencies. One current
solution is to define an extra called, for example, ``recommended``, which
includes all non-essential but suggested dependencies. Users are then told to
install the package using ``package[recommended]``, while those who prefer more
control can simply use ``package``. However, in practice, many users are unaware
of the ``[recommended]`` syntax, placing the burden on them to know this for a
typical installation.

Another motivating use case is for packages that support different backends or
frontends, where at least one must be installed for the package to be
functional, but where users may want to install only one of the other backends
and not the default one.

This PEP proposes a method to specify one or more extras to be included by
default while providing a way for users to remove one or more of these default
extras, which would solve both use cases above.

The use cases and possible solutions in this PEP were discussed extensively at
https://discuss.python.org/t/adding-a-default-extra-require-environment/4898/38.

Specification
=============

Terminology
-----------

When installing a package, selecting optional dependencies (commonly known as
*extras*) will now be referred to as *selecting* extras. For example, in
the following command, a user has *selected* the ``recommended`` extra::

    pip install package[recommended]

``Default-Extras`` Metadata Field
---------------------------------

A new metadata field, ``Default-Extras``, will be added to the `core package
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
minus sign (``-``) before the extra name. The proposed syntax update is as follows::

    extras_list   = (-)?identifier (wsp* ',' wsp* (-)?identifier)*

If both an extra and its negated version appear in an extras list, the
non-negated extra should take precedence.

Attempting to unselect an extra that does not exist or is not in the list of
default extras should do nothing. The rationale for these rules is to avoid
having to build the package simply to validate the name of the extras.

Valid examples of the new syntax include:

* ``package[-recommended]``
* ``package[-backend1, backend2]``
* ``package[pdf, -svg]``
* ``package[pdf, -pdf]`` (equivalent to ``package[pdf]``)
* ``package[-nondefault, pdf]`` where ``nondefault`` is not in ``Default-Extras`` (equivalent to ``package[pdf]``)
* ``package[-nonexistent, svg]`` where ``nonexistent`` is not defined as an extra (equivalent to ``package[svg]``)

Note that unselecting with ``-`` should *not* be considered equivalent to
*requiring* that the dependencies are not present, since this would make it
impossible to interpret a dependency tree containing both ``package[-pdf]`` and
``package[pdf]``. Unselecting an extra is merely a way of stating that the extra
isn't needed, not that it must be absent.

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

Adding a special entry in ``extras_require``
--------------------------------------------

A potential solution that has been explored is to make use of an extra with a
'special' name, for example, an empty string::

    Provides-Extra:
    Requires-Dist: numpy ; extra == ''

The idea would be that dependencies installed as part of the 'empty' extras
would only get installed if another extra was not specified. However, there are
several issues with this approach:

* The syntax ``extra == ''`` does not fit into how extras are evaluated in the
  context of environment markers. ``extra == 'pdf'`` does not mean that a user
  specified only the ``pdf`` extra, but that ``pdf`` was one of the extras
  specified. In this context, ``extra == ''`` would not behave the same,
  as it does not indicate that no extras were specified at all.
* The only way to avoid installing the default empty string extra would be to select
  another extra. There would be no way to have a minimal installation with no
  extras at all.
* It might surprise some users that requesting, e.g., ``package[pdf]`` may actually
  result in some dependencies no longer being installed (those that were part of
  the default extras).

This approach was one of those discussed extensively in
`https://github.com/pypa/setuptools/issues/1139` and
`https://github.com/pypa/setuptools/pull/1503`.

``Default-Extras`` only apply if no other extras are specified
--------------------------------------------------------------

An alternative considered was that default extras would be specified as proposed
in this PEP, but the ``-`` syntax for unselecting dependencies would not be
introduced. Instead, default extras would apply only if no extras were
explicitly requested.

However, this would not be sufficient. Similar to the approach of using a special entry
in ``extras_require``, there would be no way to remove default extras without
selecting a new extra. Thus, there would be no way to do a minimal installation, and
users might be surprised if specifying an extra resulted in some dependencies no longer
being installed.

Relying on tooling to deselect any default extras
-------------------------------------------------

Another option to unselect extras would be to implement this at the
level of packaging tools. For instance, pip could include an option such as::

    pip install package --no-default-extras

This option could apply to all or specific packages, similar to
the ``--no-binary`` option, e.g.,::

    pip install package --no-default-extras :all:

The advantage of this approach is that tools supporting default extras could
also support unselecting them. This approach would be similar to the ``--no-install-recommends``
option for the ``apt`` tool.

However, this solution is not ideal because it would not allow packages to
specify themselves that they do not need some of the default extras of a
dependency. It would also carry risks for users who might disable all
default extras, potentially breaking packages in the dependency tree that rely on
the default extras.

Disabling all default extras
----------------------------

One idea was to allow a special syntax to disable all default dependencies,
such as ``package[-*]``. However, there was concern that some package maintainers
might overuse this option, always rejecting default dependencies.

``package[]`` disables default extras
-------------------------------------

Another way to specify not to install any extras, including default extras, would
be to use ``package[]``. However, this would break the current assumption that
``package[]`` is equivalent to ``package``, and may also (similarly to ``-*``) result
in developers overusing ``[]`` by default. This approach would also not
allow any extras to be installed while removing the default ones.
