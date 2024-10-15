PEP: TBD
Title: Default Extras for Python Packages
Author: Thomas Robitaille <thomas.robitaille@gmail.com>,
        Jonathan Dekhtiar <jonathan@dekhtiar.com>
Sponsor: TBD
Discussions-To: TBD
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 24-Sep-2024
Post-History: TBD

Abstract
========

:pep:`508` specifies a mini-language for
declaring package dependencies. One feature of this language is the ability to
specify *extras*, which are optional components of a distribution that, when
used, install additional dependencies. This PEP proposes a mechanism to allow
one or more extras to be included or "selected" by default and defines a method
for these default extras to be "unselected" if desired.

Motivation
==========

This PEP proposes a method to specify one or more extras to be included by
default while providing a way for users to remove one or more of these default
extras, which would solve both use cases above. Various use cases and possible
solutions in this PEP were discussed extensively at
https://discuss.python.org/t/adding-a-default-extra-require-environment/4898/38.
In this section we take a look at two common use cases that provide the
motivation for the present PEP.

Recommended but not required dependencies
-----------------------------------------

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
typical installation. Having a way to have recommended dependencies be installed
by default while providing a way for users to request a more minimal installation
would satisfy this use case.

Packages supporting multiple backends or frontends
--------------------------------------------------

Another common use case for using extras are to define different backends or
frontends, and dependencies that need to be installed for each backend or
frontend. A package might need at least one backend or frontend to be installed
in order to be functional, but may be flexible on which backend or frontend this
is. With current packaging standards, maintainers have the choice between either
making one of the backends or frontends be always required, or requiring users
to always specify extras, e.g. ``package[backend]``. Having a way to specify one
or more default backend or frontend and providing a way to override these
defaults would provide a much better experience for users.

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
This field allows package maintainers to define an extra that is
automatically selected when a user installs the package without specifying any
extras::

    Default-Extras: recommended

If multiple default extras are needed, one ``Default-Extras:`` entry
should be provided for each one.

    Default-Extras: backend1
    Default-Extras: backend2
    Default-Extras: backend3

Overriding default extras
-------------------------

The proposal here is simple - if extras are explicitly given in a dependency
specification, the default extras are not installed. For example, if a package
defines a ``recommended`` default extra as well as a non-default ``optional``
extras, then if a user was to install the package with::

    pip install package

the ``recommended`` dependency would be included. If the user instead uses::

    pip install package[optional]

then the ``recommended`` extras would not be installed. A minimal installation
of a package can then be requested with::

    pip install package[]

**To be updated after implementations are tested out**: if it is not
possible to easily support ``[]`` as meaning explicitly provided no extras,
another option is that package maintainers wanting to support a minimal
installation could define an empty extras called e.g. ``nodefault`` (the name
would be up to the maintainer), and then tell users that to get a minimal
installation they could use e.g. ``package[nodefault]``

Backward Compatibility
======================

All package specification cases valid under :pep:`508` will remain valid.
Therefore, this proposal is fully backward-compatible with existing :pep:`508`
usage.

Once packages start defining default extras, those defaults will only be honored
with recent versions of packaging tools which implement this PEP, but those
packages will remain fully backward-compatible with older packaging tools - with
the only difference that the default extras will not be installed.

Implementation
==============

**To be updated after implementations are tested out**

Rejected Alternatives
=====================

Syntax for unselecting extras
-----------------------------

One of the main competing approaches was as follows: instead of having defaults
be unselected if any extras were explicitly provided, default extras would need
to be explicitly unselected.

In this picture, a new syntax for unselecting extras would be introduced as an
extension of the mini-language defined in :pep:`508`. If a package defined
default extras, users could opt out of these defaults by using a minus sign
(``-``) before the extra name. The proposed syntax update is as follows::

    extras_list   = (-)?identifier (wsp* ',' wsp* (-)?identifier)*

If both an extra and its negated version appear in an extras list, the
non-negated extra should take precedence.

Valid examples of this new syntax would have included, e.g.:

* ``package[-recommended]``
* ``package[-backend1, backend2]``
* ``package[pdf, -svg]``

However, there are two main issues with this approach:

* One would need to define a number of rules for how to interpret corner cases
  such as if an extras and its negated version were both present in the same
  dependency specification (e.g. ``package[pdf, -pdf]``) or if a dependency
  tree included both ``package[pdf]`` and ``package[-pdf]``, and the rules would
  not be intuitive to users.

* More importantly, this would introduce new syntax into dependency specification,
  which means that if any package defined a dependency using the new syntax, it
  and any other package depending on it would no longer be installable by existing
  packaging tools, so this would be a major backward compatibility break.

For these reasons, this alternative was not included in the final proposal.

Adding a special entry in ``extras_require``
--------------------------------------------

A potential solution that has been explored as an alternative to introducing the
new ``Default-Extras`` metadata field would be to make use of an extra with a
'special' name.

One example would be to use an empty string::

    Provides-Extra:
    Requires-Dist: numpy ; extra == ''

The idea would be that dependencies installed as part of the 'empty' extras
would only get installed if another extra was not specified. An implementation
of this was proposed in https://github.com/pypa/setuptools/pull/1503, but it
was found that there would be no way to make this work without breaking
compatibility with existing usage. For example, packages using setuptools via
a setup.py file can do:

```
setup(
    ...
    extras_require={'': ['package_a']},
)
```

which is valid and equivalent to having ``package_a`` being defined in
``install_requires``, so changing the meaning of the empty string requires would
break compatibility.

In addition, no other string can be used as a special string since all strings
that would be a backward-compatible valid extras name may already be used in
existing packages.

There have been suggestions of using the special ``None`` Python variable, but
again this is not possible, because even though one can use ``None`` in a ``setup.py`` file,
this is not possible in declarative files such as ``setup.cfg`` or
``pyproject.toml``, and furthermore ultimately extras names have to be converted
to strings in the package metadata. Having:

    Provides-Extra: None

would be indistinguishable from the string 'None' which may already be used as
an extras name in a Python package. If we were to modify the core metadata
syntax to allow non-string 'special' extras names, then we would be back to
modifying the core metadata specification, in which case we might as well
introduce ``Default-Extras``.

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
dependency. It would also carry risks for users who might disable all default
extras in a big dependency tree, potentially breaking packages in the tree that
rely on default extras at any point.
