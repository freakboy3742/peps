PEP: ???
Title: Extending PEP 517 Build Backends to Handle Cross Builds
Author: Hood Chatham <roberthoodchatham at gmail.com>, Russell Keith-Magee <russell@keith-magee.com>
Sponsor: Russell Keith-Magee
Discussions-To: https://discuss.python.org/t/86862
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 22-Oct-2026

Abstract
========



Motivation
==========

New platforms such as Android, Emscripten, and iOS are not self-hosted: It is
impossible to run their compilers on the system itself. Instead they require
cross compilation.

Rationale
=========


Specification
=============

Changes to the `pyproject.toml` format
--------------------------------------

The ``[build-system]`` section gains a new `host-requires` field.

Changes to the ``packaging`` package
------------------------------------

The ``packaging.tags`` module gains a new function called
``host_platform_tags(build_details)`` which returns an iterator over the
platform tags that are compatible with the host platform.

The ``packaging.markers`` modules gains a new function called
``host_enviornment(build_details)`` which computes an appropriate
``packaging.markers.Environment`` for the host ``build_details``.


Changes to the PEP 517 backend Hooks
------------------------------------

We add three new hooks for cross-build-aware backends,
``get_build_requires_for_cross_build_wheel()``,
``get_host_requires_for_cross_build_wheel()`` and ``cross_build_wheel()``.
Presence of the ``cross_build_wheel()`` hook should be used by the build
frontend to determine whether the build backend is cross-build aware. It is
optional whether a cross-build-aware build backend defines
``get_build_requires_for_cross_build_wheel()`` or
``get_host_requires_for_cross_build_wheel()``.

The new hooks take a common ``build_details`` argument which is created by
parsing the ``build_details.json`` file for the host platform.


``get_build_requires_for_cross_build_wheel``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: python

    def get_build_requires_for_cross_build_wheel(config_settings, build_details):
        ...

This hook is the cross-build counterpart of ``get_requires_for_build_wheel()``.
If present, it MUST return an additional list of strings containing :pep:`508`
dependency specifications, above and beyond those specified in the
``pyproject.toml`` file, to be installed into the build environment before
calling ``cross_build_wheel()``. The implicit default hook always returns an
empty list.


``get_host_requires_for_cross_build_wheel``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: python

    def get_host_requires_for_cross_build_wheel(config_settings, build_details):
        ...

This hook, if present, MUST return an additional list of strings containing
:pep:`508` dependency specifications, above and beyond those specified in the
``pyproject.toml`` file, to be installed into the ``host_requirements``
directory when calling the ``cross_build_wheel``. The implicit default hook
always returns an empty list.


``cross_build_wheel``
~~~~~~~~~~~~~~~~~~~~~

Presence of this hook signals that the backend supports cross builds. It takes
the same arguments as the ``build_wheel`` hook, as well as two additional
arguments: ``build_details`` and ``host_requires_directory``. The
``host_requires_directory`` will be prepared by the build frontend containing
installed copies of all the host requirements.


.. code:: python

    def cross_build_wheel(wheel_directory, metadata_directory=None, config_settings=None, build_details=None, host_requires_directory=None, toolchain=None):
        ...

The ``toolchain`` argument consists of a config object with the values of
``CC``, ``CXX``, ``RANLIB``, ``AR``, ``AS``, etc.


What a Build Frontend should Do
-------------------------------

Suppose given a host platform tag, a ``build_details.json`` file, and a project
directory. To cross build the project, the build frontend should do the
following steps:

1. Check that the build backend implements a ``cross_build_wheel`` hook. If it
   does not, error out.
2. Install build requirements.
   a. Install the `build.requirements` from ``pyproject.toml``.
   b. If it is present, invoke the
      ``get_build_requires_for_cross_build_wheel()`` build backend hook  and
      install the requirements returned.
3. Install host requirements:
   a. Make a directory to use as `host_requires_directory`.
   b. Look up the `build.host-requirements` from ``pyproject.toml``
   c. If it is present, invoke the
      ``get_host_requires_for_cross_build_wheel()`` hook and extend the
      ``host-requirements`` list with the result.
   d. Call ``packaging.markers.host_environment(build_details)`` to get the
      host environment for evaluating requirement markers.
   e. For each host requirement, call
      ``requirement.marker.evaluate(host_environment, "requirement")``. Filter
      the ``host-requirements`` list to only include the requirements that return
      ``True``.
   f. Call ``packaging.tags.host_platform_tags(build_details)`` to get an
      iterator over the compatible host platform tags.
   g. For each of the host requirements, find a matching wheel with a
      compatible platform tag and install it into the
      `host_requires_directory`.
4. Invoke the `cross_build_wheel()` build backend hook.

The `host_requires_directory` is just a directory. If the build frontend
prefers it can be the `site-packages` inside of some cross virtual environment,
but we do not need to execute the host packages as part of the build and there
is not yet a well-specified notion of a cross virtual environment.



