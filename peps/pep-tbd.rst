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
specifying a dependency to be installed. One of the features of this language is
the ability to specify 'extras', which are optional parts of a distribution
which result in additional dependencies being installed. This PEP proposes to
provide a way for one or more extras to be included or "selected" by default, and
to define a way for these default extras to be "unselected".

Motivation
==========

Package maintainers who define extras in their packages will typically use these
to declare optional sets of dependencies which expand the available
functionality or the performance of a package. In some cases, deciding what to
declare as a required dependency compared to in an extras can be tricky, as
there is a need to balance the needs of typical users (who may prefer to have
most of the package working 'by default') and the need of users that want
minimal installations that do not include large and optional dependencies. An
example of a solution to this problem currently is to define an extras called
e.g. ``recommended``, which includes all recommended but not strictly required
dependencies, and to then tell users to install the package as
``package[recommended]``, which then allows users that want more control over
dependencies to use ``package``. In practice however, users are often not aware
of the ability to use the ``recommended`` syntax, but the burden is on them to
know this in order to get a 'typical' recommended installation of the package.
This PEP proposes to have a way to make one or more extras be included by
default, and provide a way for users who want a minimal installation to opt-out
of installing these extras.

Another use case which motivates this proposal is that of packages that can
provide support for different backends or frontends, and which need at least one
backend or front-end to be installed to be usable. With the proposed approach
below, it would then be possible for package maintainers to define a default
extras containing a backend or frontend to install by default, but having a way
for users to opt-out of this backend or front-end if desired and optionally
install another one.

Specification
=============

Terminology
-----------

When installing a package, the process of specifying optional dependencies
(commonly called "extras") will now be referred to as **"selecting"** extras.
For example, in the following example, a user has **selected** the
``recommended`` extras::

    pip install package[recommended]

``Default-Extras`` Metadata Field
---------------------------------

A new metadata field, `Default-Extras`, will be added to the `core package
metadata
<https://packaging.python.org/en/latest/specifications/core-metadata/#core-metadata>`_.
This field allows package maintainers to define one or more extras that are
automatically selected when a user installs the package without specifying any
extras. If multiple extras are specified, they should be separated by commas.

Example with a single default extras::

    Default-Extras: recommended

Example with multiple default extras::

    Default-Extras: recommended,pdf

Unselecting Default Extras
--------------------------

A new syntax for **unselecting** extras will be introduced, which is an addition
to the mini-language defined in `PEP 508 <https://peps.python.org/pep-0508/>`_.
If a package defines default extras, the user can opt out of these defaults by
using a minus sign (`-`) before the extra name. Specifically, we propose
updating the following syntax::

    extras_list   = identifier (wsp* ',' wsp* identifier)*

to instead be::

    extras_list   = (-)?identifier (wsp* ',' wsp*  (-)?identifier)*

In addition, an extra cannot appear both in negated and non-negated form as this
would be ambiguous.

**Valid new examples**

* ``package[-recommended]``
* ``package[-backend1, backend2]``
* ``package[pdf, -svg]``


**Invalid examples**

* ``package[pdf, -pdf]`` - an extras cannot be both negated and non-negated
* ``package[-nondefault]`` - if ``nondefault`` was not specified as a default extras
* ``package[-nonexistent]`` - if ``nonexistent`` is not defined as an extras

Backward Compatibility
======================

All cases of package specification which were valid as per `PEP 508
<https://peps.python.org/pep-0508/>`_ will remain valid. Thus, the proposed
change is fully backward-compatible in supporting any existing use of `PEP 508
<https://peps.python.org/pep-0508/>`_.

Users will be able to benefit from the ability to deselect default extras for
a given package once that package defines one or more default extras, and once
the tooling used to install the package (e.g. pip) support this syntax.

Implementation
==============

TBD

Alternatives
============

Specifying any extras explicitly unselects any default
------------------------------------------------------

Another idea that was considered was that, rather than introducing a new syntax
to unselect extras, specifying any extras explicitly would automatically
unselect any default.

TBD - determine if this is actually the preferred approach!
