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
extras. Various use cases and possible solutions in this PEP were discussed
extensively at
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
mini-language defined in :pep:`508`. If a
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

All package specification cases valid under :pep:`508` will remain valid.
Therefore, this proposal is fully backward-compatible with existing :pep:`508`
usage. Users will gain the ability to deselect default extras once a package defines
default extras and the package installation tools (e.g., pip) support the new syntax.

However, it is important to note that the ``-`` syntax for unselecting extras is
itself not backward-compatible with existing packaging tools. There are two main
scenarios where this syntax is likely to be used:

* Users setting up an environment with packages using e.g. a pip command, a pip
  requirement file, or other ways of specifying environments should be able to
  use the new syntax provided they have a recent version of pip. This is not a
  major concern because using this syntax will be opt-in and it is likely to be
  sufficient to document that this can only be used with recent packaging tools.

* Developers defining dependencies for their own packages and deciding to use
  the ``-`` syntax in dependency lists will be making their packages and all
  dependent packages incompatible with older versions of packaging tools, since
  these will crash when encountering ``-`` which is not allowed by :pep:`508`.
  Using this new syntax in package dependency lists should therefore be done
  with great caution as it may have an impact on users.

We therefore strongly recommend that package maintainers avoid using the new syntax in
package dependency lists until they are confident that most users that use their
package or one of the dependent packages are likely to be using recent versions
of pip.

This feature should be safe to use in package dependency lists once the package
only supports Python versions on which all supported versions of packaging tools
support this feature.

Implementation
==============

TBD - once we agree on the best path forward.

Rejected Alternatives
=====================

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

Another shortcoming of the approach of using a 'special' extras is that only one
special extras can be defined - it isn't possible for instance to have two default
backends and then have a way to unselect one of them.

``Default-Extras`` only apply if no other extras are specified
--------------------------------------------------------------

An alternative considered was that default extras would be specified as proposed
in this PEP, but the ``-`` syntax for unselecting dependencies would not be
introduced. Instead, default extras would apply only if no extras were
explicitly requested.

However, this would not be sufficient. Similar to the approach of using a special entry
in ``extras_require``, there would be no way to remove default extras without
selecting a new extra, thus there would be no way to do a minimal installation. In addition,
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
dependency. It would also carry risks for users who might disable all default
extras in a big dependency tree, potentially breaking packages in the tree that
rely on default extras at any point.

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
