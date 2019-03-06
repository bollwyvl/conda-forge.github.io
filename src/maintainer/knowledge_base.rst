Knowledge base
**************

Particularities on Windows
==========================

This document presents conda-forge and conda-build information and examples
when building on Windows.


Local testing
-------------

The first thing that you should know is that you can locally test Windows
builds of your packages even if you don’t own a Windows machine. Microsoft
makes available free, official Windows virtual machines (VMs) `at this website
<https://developer.microsoft.com/en-us/microsoft-edge/tools/vms/>`_. If you
are unfamiliar with VM systems or have trouble installing Microsoft’s, please
use a general web search to investigate — while these topics are beyond the
scope of this documentation, there is ample discussion of them on the broader
Internet.

In order to compile native code (C, C++, etc.) on Windows, you will need to
install Microsoft’s Visual C++ build tools on your VM. You must install
particular versions of these tools — this is to maintain compatibility between
compiled libraries used in Python, `as described on this Python wiki page
<https://wiki.python.org/moin/WindowsCompilers>`_. The current relevant
versions are:

* For Python 2.7: Visual C++ 9.0
* For Python 3.5–3.7: Visual C++ 14.0

While you can obtain these tools by installing the right version of the full
`Visual Studio <https://visualstudio.microsoft.com/>`_ development
environment, you can save a lot of time and bandwidth by installing standalone
“build tools” packages. The links are:

* For Python 2.7: `Microsoft Visual C++ Compiler for Python 2.7
  <https://www.microsoft.com/download/details.aspx?id=44266>`_.
* For Python 3.5–3.7: `Microsoft Build Tools for Visual Studio 2017
  <https://www.visualstudio.com/downloads/#build-tools-for-visual-studio-2017>`_.

Please see `the Python wiki page on Windows compilers
<https://wiki.python.org/moin/WindowsCompilers>`_ if you need more
information.


Simple CMake-Based ``bld.bat``
------------------------------
Some projects provide hooks for CMake to build the project. The following
example ``bld.bat`` file demonstrates how to build a traditional, out-of-core
build for such projects.

**CMake-based bld.bat:**

.. code-block:: bat

    setlocal EnableDelayedExpansion

    :: Make a build folder and change to it.
    mkdir build
    cd build

    :: Configure using the CMakeFiles
    cmake -G "NMake Makefiles" ^
          -DCMAKE_INSTALL_PREFIX:PATH="%LIBRARY_PREFIX%" ^
          -DCMAKE_PREFIX_PATH:PATH="%LIBRARY_PREFIX%" ^
          -DCMAKE_BUILD_TYPE:STRING=Release ^
          ..
    if errorlevel 1 exit 1

    :: Build!
    nmake
    if errorlevel 1 exit 1

    :: Install!
    nmake install
    if errorlevel 1 exit 1

The following feedstocks are examples of this build structure deployed:

* `libpng <https://github.com/conda-forge/libpng-feedstock/blob/master/recipe/bld.bat>`_
* `pugixml <https://github.com/conda-forge/pugixml-feedstock/blob/master/recipe/bld.bat>`_


Building for different VC versions
----------------------------------

On Windows, different Visual C versions have different ABI and therefore a package needs to be built for different Visual C versions. Packages are tied to the VC version that they were built with and some packages have specific requirements of the VC version. For example, python 2.7 requires ``vc 9`` and python 3.5 requires ``vc 14``.

With ``conda-build 3.x``, ``vc`` can be used as a selector when using the ``compiler`` jinja syntax.

.. code-block:: yaml

    requirements:
      build:
        - {{ compiler('cxx') }}

To skip building with a particular ``vc`` version, add a skip statement.

.. code-block:: yaml

    build:
        skip: true  # [win and vc<14]

    requirements:
      build:
        - {{ compiler('cxx') }}



Special dependencies
====================

.. _dep_compilers:

Compilers
---------

Compilers are dependencies with a special syntax and are always added to ``requirements/build``.

There are currently three supported compilers:

 - c
 - cxx
 - fortran

A package that needs all three compilers would define

.. code-block:: yaml

    requirements:
      build:
        - {{ compiler('c') }}
        - {{ compiler('cxx') }}
        - {{ compiler('fortran') }}

.. note::

  Appropriate compiler runtime packages will be automatically added to the package's runtime requirements and therefore there's no need to specify ``libgcc`` or ``libgfortran``.
  There is additional information about how conda-build 3 treats compilers in the `conda docs <https://docs.conda.io/projects/conda-build/en/latest/source/compiler-tools.html>`_.

.. _cdt_packages:

Core dependency tree packages (CDT)
-----------------------------------

Dependencies outside of the conda-forge channel should be avoideded (see :ref:`no_external_deps`).
However there are very few exceptions: some dependencies are so close to the system that they are not packaged with conda-forge.
These dependencies have to be satisfied with *Core Dependency Tree* packages.

In conda-forge this currently affects only packages that link against libGL.

libGL
.....

In addition to the required compilers ``{{ compiler('c') }}`` and/or ``{{ compiler('cxx') }}``, following CDT packages are required for linking against libGL:

.. code-block:: yaml

  requirements:
    build:
      - {{ cdt('mesa-libgl-devel') }}  # [linux]
      - {{ cdt('mesa-dri-drivers') }}  # [linux]
      - {{ cdt('libselinux') }}  # [linux]
      - {{ cdt('libxdamage') }}  # [linux]
      - {{ cdt('libxxf86vm') }}  # [linux]
    host:
      - xorg-libxfixes  # [linux]


If you need a fully functional binary in the test phase, you have to also provide the shared libraries via ``yum_requirements.txt`` (see :ref:`yum_deps`).

::

  mesa-libGL
  mesa-dri-drivers
  libselinux
  libXdamage
  libXxf86vm


.. _linking_numpy:

Building Against NumPy
----------------------

Packages that link against NumPy need a special treatment in the dependency section.
Finding ``numpy.get_include()`` in ``setup.py`` or ``cimport`` statements in ``.pyx`` or ``.pyd`` fil
es are a telltale sign that the package links against NumPy.

In the case of linking, you need to use the ``pin_compatible`` function to ensure having a compatible
 numpy version at run time:

.. code-block:: yaml

    host:
      - numpy
    run:
      - {{ pin_compatible('numpy') }}


At the time of writing, above is equivalent to the following,

.. code-block:: yaml

    host:
      - numpy 1.9.3              # [unix]
      - numpy 1.11.3             # [win]
    run:
      - numpy >=1.9.3,<2.0.a0    # [unix]
      - numpy >=1.11.3,<2.0.a0   # [win]


.. admonition:: Notes

    1. You still need to respect minimum supported version of ``numpy`` for the package!
    That means you cannot use ``numpy 1.9`` if the project requires at least ``numpy 1.12``,
    adjust the minimum version accordingly!

    .. code-block:: yaml

        host:
          - numpy 1.12.*
        run:
          - {{ pin_compatible('numpy') }}


    2. if your package supports ``numpy 1.7``, and you are brave enough :-),
    there are ``numpy`` packages for ``1.7`` available for Python 2.7 in the channel.


Message passing interface (MPI)
-------------------------------

.. note::
  
  This section originates from Min's notes: https://hackmd.io/ry4uI0thTs2q_b4mAQd_qg

MPI Variants in conda-forge
...........................

How are MPI variants best handled in conda-forge?


There are a few broad cases:

- package requires a specific MPI provider (easy!)
- package works with any MPI provider (e.g. mpich, openmpi)
- package works with/without MPI



Building MPI variants
^^^^^^^^^^^^^^^^^^^^^

In `conda_build_config.yaml`:

.. code-block:: yaml

  mpi:
    - mpich
    - openmpi


In `meta.yaml`:

.. code-block:: yaml

  requirements:
    host:
      - {{ mpi }}

And rerender with:

.. code-block:: bash

  conda-smithy rerender -c auto

to produce the build matrices.

Current builds of both mpi providers have `run_exports` which is equivalent to adding:

.. code-block:: yaml

  requirements:
    run:
      - {{ pin_run_as_build(mpi, min_pin='x.x', max_pin='x.x') }}

If you want to do the pinning yourself (i.e. not trust the mpi providers, or pin differently, add):

.. code-block:: yaml

  # conda_build_config.yaml
  pin_run_as_build:
    mpich: x.x
    openmpi: x.x
 
.. code-block:: yaml

  # meta.yaml
  requirements:
    host:
      - {{ mpi }}
    run:
      - {{ mpi }}

Including a no-mpi build
^^^^^^^^^^^^^^^^^^^^^^^^

Some packages (e.g. hdf5) may want a no-mpi build, in addition to the mpi builds.
To do this, add `nompi` to the mpi matrix:

.. code-block:: yaml

  mpi:
    - nompi
    - mpich
    - openmpi

and apply the appropriate conditionals in your build:

.. code-block:: yaml

  requirements:
    host:
      - {{ mpi }}  # [mpi != 'nompi']
    run:
      - {{ mpi }}  # [mpi != 'nompi']



Preferring a provider (usually nompi)
"""""""""""""""""""""""""""""""""""""

Up to here, mpi providers have no explicit preference. When choosing an MPI provider, the mutual-exclusivity of the ``mpi`` metapackage allows picking between mpi providers by installing an mpi provider, e.g.

.. code-block:: bash

    conda install mpich ptscotch

or

.. code-block:: bash

    conda install openmpi ptscotch

This doesn't extend to ``nompi``, because there is no ``nompi`` variant of the mpi metapackage. And there probably shouldn't be, because some packages built with mpi doesn't preclude other packages in the env that *may* have an mpi variant from using the no-mpi variant of the library (e.g. for a long time, fenics used mpi with no-mpi hdf5 since there was no parallel hdf5 yet. This works fine, though some features may not be available).

Typically, if there is a preference it will be packages with a nompi variant, where the serial build is preferred, such that installers/requirers of the package only get the mpi build if explicitly requested.


.. admonition:: Outdated

  To de-prioritize a build in the solver, it can be given a special ``track_features`` field:

  - All builds *other than* the priority build should have a ``track_features`` field
  - Build strings can be used to allow downstream packages to make explicit dependencies.
  - No package should actually *have* the tracked feature.


  .. note:: **update**: track_features deprioritization has too high priority in the solver, preventing a package from adopting a variant of a dependency after some builds have already been made. Instead, use a build number offset to apply the preference at a more appropriate level.


Here is an example build section:

::

  {% if mpi == 'nompi' %}
  # prioritize nompi variant via build number
  {% set build = build + 100 %}
  {% endif %}
  build:
    number: {{ build }}

    # add build string so packages can depend on
    # mpi or nompi variants explicitly:
    # `pkg * mpi_mpich_*` for mpich
    # `pkg * mpi_*` for any mpi
    # `pkg * nompi_*` for no mpi

    {% set mpi_prefix = "mpi_" + mpi %}
    {% else %}
    {% set mpi_prefix = "nompi" %}
    {% endif %}
    string: "{{ mpi_prefix }}_h{{ PKG_HASH }}_{{ build }}"

.. note::

  ``{{ PKG_HASH }}`` avoids build string collisions on *most* variants,
  but not on packages that are excluded from the default build string,
  e.g. Python itself. If the package is built for multiple Python versions, use:

  .. code-block:: yaml

    string: "{{ mpi_prefix }}_py{{ py }}h{{ PKG_HASH }}_{{ build }}"

  as seen in `mpi4py <https://github.com/conda-forge/h5py-feedstock/pull/49/commits/b08ee9307d16864e775f1a7f9dd10f25c83b5974>`__


This build section creates the following packages:

- ``pkg-x.y.z-mpi_mpich_h12345_0``
- ``pkg-x.y.z-mpi_openmpi_h23456_0``
- ``pkg-x.y.z-nompi_h34567_100``

Which has the following consequences:

- The ``nompi`` variant is preferred, and will be installed by default unless an mpi variant is explicitly requested.
- mpi variants can be explicitly requested with ``pkg=*=mpi_{{ mpi }}_*``
- any mpi variant, ignoring provider, can be requested with ``pkg=*=mpi_*``
- nompi variant can be explicitly requested with ``pkg=*=nompi_*``

If building with this library creates a runtime dependency on the variant, the build string pinning can be added to ``run_exports``.

For example, if building against the nompi variant will work with any installed version, but building with a given mpi provider requires running with that mpi:


::

  build:
    ...
    {% if mpi != 'nompi' %}
    run_exports:
      - {{ name }} * {{ mpi_prefix }}_*
    {% endif %}

Remove the ``if mpi...`` condition if all variants should create a strict runtime dependency based on the variant chosen at build time (i.e. if the nompi build cannot be run against the mpich build).

Complete example
""""""""""""""""

Combining all of the above, here is a complete recipe, with:

- nompi, mpich, openmpi variants
- run-exports to apply mpi choice made at build time to runtime where nompi builds can be run with mpi, but not vice versa.
- nompi variant is preferred by default
- only build nompi on Windows

This matches what is done in `hdf5 <https://github.com/conda-forge/hdf5-feedstock/pull/90>`__.


.. code-block:: yaml

  # conda_build_config.yaml
  mpi:
    - nompi
    - mpich  # [not win]
    - openmpi  # [not win]

::

  # meta.yaml
  {% set name = 'pkg' %}
  {% set build = 1000 %}

  # ensure mpi is defined (needed for conda-smithy recipe-lint)
  {% set mpi = mpi or 'nompi' %}

  {% if mpi == 'nompi' %}
  # prioritize nompi variant via build number
  {% set build = build + 100 %}
  {% endif %}

  build:
    number: {{ build }}

    # add build string so packages can depend on
    # mpi or nompi variants explicitly:
    # `pkg * mpi_mpich_*` for mpich
    # `pkg * mpi_*` for any mpi
    # `pkg * nompi_*` for no mpi

    {% if mpi != 'nompi' %}
    {% set mpi_prefix = "mpi_" + mpi %}
    {% else %}
    {% set mpi_prefix = "nompi" %}
    {% endif %}
    string: "{{ mpi_prefix }}_h{{ PKG_HASH }}_{{ build }}"

    {% if mpi != 'nompi' %}
    run_exports:
      - {{ name }} * {{ mpi_prefix }}_*
    {% endif %}

  requirements:
    host:
      - {{ mpi }}  # [mpi != 'nompi']
    run:
      - {{ mpi }}  # [mpi != 'nompi']

And then a package that depends on this one can explicitly pick the appropriate mpi builds:

.. code-block:: yaml

  # meta.yaml

  requirements:
    host:
      - {{ mpi }}  # [mpi != 'nompi']
      - pkg
      - pkg * mpi_{{ mpi }}_*  # [mpi != 'nompi']
    run:
      - {{ mpi }}  # [mpi != 'nompi']
      - pkg * mpi_{{ mpi }}_*  # [mpi != 'nompi']

mpi-metapackage exclusivity allows ``mpi_*`` to resolve the same as ``mpi_{{ mpi }}_*`` if ``{{ mpi }}`` is also a direct dependency, though it's probably nicer to be explicit.

Just mpi example
""""""""""""""""

Without a preferred ``nompi`` variant, recipes that require mpi are much simpler. This is all that is needed:

.. code-block:: yaml

  # conda_build_config.yaml
  mpi:
    - mpich
    - openmpi

.. code-block:: yaml

  # meta.yaml
  requirements:
    host:
      - {{ mpi }}
    run:
      - {{ mpi }}



OpenMP on macOS
---------------

.. todo::

  Get help from @beckermr


.. _yum_deps:

yum_requirements.txt
--------------------

Dependencies can be installed into the build container with ``yum``, by listing package names line by line in a file named ``yum_requirements.txt`` in the ``recipe`` directory of a feedstock.

There are only very few situations where dependencies installed by yum are acceptable. These cases include

  - satisfying the requirements of :term:`CDT` packages during test phase
  - installing packages that are only required for testing


Special packages
================

BLAS
----

.. todo::

  The information regarding the blas package may now be outdated and needs checking.

BLAS metapackage
................

   -  Version will have two values X.Y

      -  X represents changes to the metapackage.
      -  Y represents priority of BLAS (if we change priorities BLASes X
         must be bumped, if we want to prioritize something new over
         OpenBLAS we do not need to change X)

         -  1 is OpenBLAS
         -  0 is None (no BLAS whatsoever)

   -  needs to have version stay the same across all variants.
   -  build number cannot be touched (it won't be in the string anyways)
   -  no pinning of BLAS inside the metapackages (dependencies can pin
      this)
   -  to preserve a BLAS in an environment it is recommend to add
      ``pinned`` to ``conda-meta`` and specify down to the build string
      what is the expected BLAS
   -  To install a specific variant, ``conda install blas=1.0=none`` /
      ``conda install blas=1.0=openblas``. It is hoped the syntax will be
      improved in conda.

      -  In the future, with some fixes to ``conda`` will allow for syntax
         like ``conda install blas=*=openblas``. We should keep an eye on
         this. (maybe even ``conda install blas:openblas``)

   -  There will be two variants initially:

      -  openblas
      -  noblas - no BLAS optimisations (e.g. for reasons of smaller
         installations)

NumPy package
.............

   -  "version + build number" must always be greater than or equal to that
      in defaults. If not, defaults "numpy" will be chosen, complete with
      mkl

      -  to make this simple we can pick a high build number so this is
         prioritized 100 and then bump from there

         -  Should make the build number ranges tied to BLAS X version above.
         -  Build number should start at ``(X+1)*100``.

            -  Means OpenBLAS starts at 200.
            -  No BLAS starts at 100.

         -  Unfortunately the 1.11.0 release breaks this rule so we will have
            No BLAS at 101.

      -  if defaults gains a newer version and build without conda-forge
         updating, users will be prompted to upgrade to the defaults numpy.
         Even if a user does this, as soon as an equivalent build is
         available on conda-forge, they will be prompted to update to their
         previous variant

   -  will track the blas\_{{ variant }} feature enabled by the BLAS
      metapackage
   -  will pin the specific blas package versions (e.g. openblas .)

SciPy, scikit-learn, etc.
.........................

   -  Same thing as NumPy
   -  Add numpy dependency as if linking occurs


Noarch builds
=============

Noarch packages are packages that are not architecture specific and therefore only have to be built once.

Declaring these packages as ``noarch`` in the ``build`` section of the meta.yaml, reduces shared CI resources. Therefore all packages that qualify to be noarch packages, should be declared as such.


.. _noarch:

Noarch python
-------------
The ``noarch: python`` directive, in the ``build`` section, makes pure-Python
packages that only need to be built once.

In order to qualify as a noarch python package, all of the following criteria must be fulfilled:

  - No compiled extensions
  - No post-link or pre-link or pre-unlink scripts
  - No OS specific build scripts
  - No python version specific requirements
  - No skips except for python version. (If the recipe is py3 only, remove skip statement and add version constraint on python)
  - 2to3 is not used
  - Scripts argument in setup.py is not used
  - If entrypoints are in setup.py, they are listed in meta.yaml
  - No activate scripts
  - Not a dependency of `conda`

.. note::
  While ``noarch: python`` does not work with selectors, it does work with version constraints.
  ``skip: True  # [py2k]`` can sometimes be replaced with a constrained python version in the build/run subsections: say ``python >=3`` instead of just ``python``.

If an existing python package qualifies to be converted to a noarch package, you can request the required changes by opening a new issue and including ``@conda-forge-admin, please add noarch: python``.


Noarch generic
--------------

.. todo::

  add some information on r packages which make heavy use of ``noarch: generic``


Build matrices
==============

Currently, ``python, vc, r-base`` will create a matrix of jobs for each supported version. If ``python`` is only a build dependency and not a runtime dependency (eg: build script of the package is written in Python, but the package is not dependent on python), use ``build`` section

Following implies that ``python`` is only a build dependency and no Python matrix will be created.

.. code-block:: yaml

    build:
      - python
    host:
      - some_other_package


Note that ``host`` should be non-empty or ``compiler`` jinja syntax used or ``build/merge_build_host`` set to True for the ``build`` section to be treated as different from ``host``.

Following implies that ``python`` is a runtime dependency and a Python matrix for each supported python version will be created.

.. code-block:: yaml

    host:
      - python

``conda-forge.yml``'s build matrices is removed in conda-smithy=3. To get a build matrix, create a ``conda_build_config.yaml`` file inside recipe folder. For example following will give you 2 builds and you can use the selector ``vtk_with_osmesa`` in the ``meta.yaml``

.. code-block:: yaml

    vtk_with_osmesa:
      - False
      - True

You need to rerender the feedstock after this change.
