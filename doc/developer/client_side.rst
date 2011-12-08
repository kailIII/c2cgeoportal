.. _developer_client_side:

Client-side development
=======================

The UI of c2cgeoportal applications is built from components of the CGXP
JavaScript library. This library is on GitHub:
https://github.com/camptocamp/cgxp.

Running tests
-------------

To run the CGXP tests start by cloning the repository, and updating its
submodules (for GXP, OpenLayers, etc.)::

    git clone https://github.com/camptocamp/cgxp
    cd cgxp
    git submodule update --init

Now open the Jasmine Spec Runner file (``core/tests/SpecRunner.html``) in your
browser. The tests should automatically run (and pass!).

Adding tests
------------

The test suite is located in the ``core/tests`` directory.

Test files (known as spec files in the Jasmine jargon) are located in the
``spec`` subdirectory. For example, to add tests for a new plugin whose js file
is ``core/src/script/CGXP/plugins/Foo.js``, a spec file named ``Foo.js`` is to
be added in ``core/tests/spec/script/CGXP/plugins/``.

Spec files are referenced using ``<script>`` tags ``SpecRunner.html``.

Coding style
~~~~~~~~~~~~

Lines should exceed 80 characters.