..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   The ``pipetask`` tool included in the Gen3 middleware for running ``PipelineTasks`` is often opaque and hard to use.  This is partly the result of history; the original interface was much more straightforward, but it assumed a very different collections model than what Butler has now, and the changes made to adapt it to the new collections system did not go deep enough.
But it also suffers from the fact that many of its steps are "leaky"; information provided in one step (e.g. input collections used for ``QuantumGraph``) cannot be straightforwardly passed to later steps (e.g. execution of that ``QuantumGraph``), forcing it to rely on the user to provide consistent arguments and limiting the guarantees we can make about behavior in the presence of user error.
Finally, the ``pipetask`` interface is currently only really usable from the command-line, making it hard to write unit tests for concrete ``PipelineTasks``; even if we were to keep the command-line interface the same, we need to refactor the implementation to provide a usable Python API.
This technote will provide a detailed design sketch for a new `pipetask` interface that attempts to address these problems, centered around a ``git``-like "campaign directory" concept that will be used to save state between different ``pipetask`` invocations.

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
