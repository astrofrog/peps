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

Additionally, an extra cannot appear in both negated and non-negated forms, as
this would create ambiguity.

**Valid new examples:**

* ``package[-recommended]``
* ``package[-backend1, backend2]``
* ``package[pdf, -svg]``

**Invalid examples:**

* ``package[pdf, -pdf]`` - an extra cannot be both negated and non-negated.
* ``package[-nondefault]`` - if ``nondefault`` is not specified as a default extra.
* ``package[-nonexistent]`` - if ``nonexistent`` is not defined as an extra.

Backward Compatibility
======================

All package specification cases valid under `PEP 508 <https://peps.python.org/pep-0508/>`_
will remain valid. Therefore, this proposal is fully backward-compatible with
existing `PEP 508` usage.

Users will gain the ability to deselect default extras once a package defines
default extras and the package installation tools (e.g., pip) support the new syntax.

Implementation
==============

TBD

Alternatives
============

Specifying any extras explicitly unselects all defaults
-------------------------------------------------------

An alternative considered was that specifying any extras would automatically
unselect all default extras, removing the need for a new syntax for unselecting.

TBD - Further discussion is required to determine if this is the preferred approach.
