.. image:: https://img.shields.io/badge/dmtn--175-lsst.io-brightgreen.svg
   :target: https://dmtn-175.lsst.io
.. image:: https://github.com/lsst-dm/dmtn-175/workflows/CI/badge.svg
   :target: https://github.com/lsst-dm/dmtn-175/actions/
..
  Uncomment this section and modify the DOI strings to include a Zenodo DOI badge in the README
  .. image:: https://zenodo.org/badge/doi/10.5281/zenodo.#####.svg
     :target: http://dx.doi.org/10.5281/zenodo.#####

#####################################
Design sketch for a pipetask overhaul
#####################################

DMTN-175
========

The ``pipetask`` tool included in the Gen3 middleware for running ``PipelineTasks`` is often opaque and hard to use.  This is partly the result of history; the original interface was much more straightforward, but it assumed a very different collections model than what Butler has now, and the changes made to adapt it to the new collections system did not go deep enough.
But it also suffers from the fact that many of its steps are "leaky"; information provided in one step (e.g. input collections used for ``QuantumGraph``) cannot be straightforwardly passed to later steps (e.g. execution of that ``QuantumGraph``), forcing it to rely on the user to provide consistent arguments and limiting the guarantees we can make about behavior in the presence of user error.
Finally, the ``pipetask`` interface is currently only really usable from the command-line, making it hard to write unit tests for concrete ``PipelineTasks``; even if we were to keep the command-line interface the same, we need to refactor the implementation to provide a usable Python API.
This technote will provide a detailed design sketch for a new `pipetask` interface that attempts to address these problems, centered around a ``git``-like "campaign directory" concept that will be used to save state between different ``pipetask`` invocations.

**Links:**

- Publication URL: https://dmtn-175.lsst.io
- Alternative editions: https://dmtn-175.lsst.io/v
- GitHub repository: https://github.com/lsst-dm/dmtn-175
- Build system: https://github.com/lsst-dm/dmtn-175/actions/


Build this technical note
=========================

You can clone this repository and build the technote locally with `Sphinx`_:

.. code-block:: bash

   git clone https://github.com/lsst-dm/dmtn-175
   cd dmtn-175
   pip install -r requirements.txt
   make html

.. note::

   In a Conda_ environment, ``pip install -r requirements.txt`` doesn't work as expected.
   Instead, ``pip`` install the packages listed in ``requirements.txt`` individually.

The built technote is located at ``_build/html/index.html``.

Editing this technical note
===========================

You can edit the ``index.rst`` file, which is a reStructuredText document.
The `DM reStructuredText Style Guide`_ is a good resource for how we write reStructuredText.

Remember that images and other types of assets should be stored in the ``_static/`` directory of this repository.
See ``_static/README.rst`` for more information.

The published technote at https://dmtn-175.lsst.io will be automatically rebuilt whenever you push your changes to the ``master`` branch on `GitHub <https://github.com/lsst-dm/dmtn-175>`_.

Updating metadata
=================

This technote's metadata is maintained in ``metadata.yaml``.
In this metadata you can edit the technote's title, authors, publication date, etc..
``metadata.yaml`` is self-documenting with inline comments.

Using the bibliographies
========================

The bibliography files in ``lsstbib/`` are copies from `lsst-texmf`_.
You can update them to the current `lsst-texmf`_ versions with::

   make refresh-bib

Add new bibliography items to the ``local.bib`` file in the root directory (and later add them to `lsst-texmf`_).

.. _Sphinx: http://sphinx-doc.org
.. _DM reStructuredText Style Guide: https://developer.lsst.io/restructuredtext/style.html
.. _this repo: ./index.rst
.. _Conda: http://conda.pydata.org/docs/
.. _lsst-texmf: https://lsst-texmf.lsst.io
