
:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   The ``pipetask`` tool included in the Gen3 middleware for running ``PipelineTasks`` is often opaque and hard to use.  This is partly the result of history; the original interface was much more straightforward, but it assumed a very different collections model than what Butler has now, and the changes made to adapt it to the new collections system did not go deep enough.
   But it also suffers from the fact that many of its steps are "leaky"; information provided in one step (e.g. input collections used for ``QuantumGraph``) cannot be straightforwardly passed to later steps (e.g. execution of that ``QuantumGraph``), forcing it to rely on the user to provide consistent arguments and limiting the guarantees we can make about behavior in the presence of user error.
   Finally, the ``pipetask`` interface is currently only really usable from the command-line, making it hard to write unit tests for concrete ``PipelineTasks``; even if we were to keep the command-line interface the same, we need to refactor the implementation to provide a usable Python API.
   This technote will provide a detailed design sketch for a new ``pipetask`` interface that attempts to address these problems, centered around a ``git``-like "campaign directory" concept that will be used to save state between different ``pipetask`` invocations.

Preliminaries
=============

The reader is assumed to be familiar with the Gen3 butler's main organizational concepts (dimensions, data IDs, datasets, dataset types, and collections), the types of collections, and the basic functionality of the current ``pipetask`` tool.


Conceptual Overview
===================

A "campaign" is a directory with (at least) a ``.campaign.json`` file that tracks the what *will be* run, and how.
As processing proceeds, this will also be the (default) directory where pipeline definitions, ``QuantumGraphs``, and logs are written.
The same campaign is generally used for multiple "runs", each of which *usually* corresponds to a single output RUN-type collection.
An important aspect of this design is that the ``.campaign.json`` information does not attempt to track what *has been* run; it may frequently do so incidentally, because our best guess at what to run next is often what we just ran, but actual provenance will always be stored in the data repository and managed by butler.

A campaign directory may be in a non-POSIX location (reads and writes will use ``ButlerURI``), and ideally we would make it possible (but not necessary) for this to be the same as the directory that maps to the common prefix of the ``RUN`` collections that hold butler-managed output datasets, even though many of the files we'll write to it (certainly ``.campaign.json`` and logs) are not butler datasets.

A campaign is often associated with a single CHAINED collection in a data repository that aggregates its inputs and outputs, but this is not required.

``pipetask`` interacts with a campaign in the same way that ``git`` interacts with a local repository: one special command, `Campaign.init`, is first invoked to set up the campaign, and all others are then invoked from within the campaign directory and then do not need to repeatedly provide the information stored within it by previous invocations.
Unlike ``git``, however, commands other than `Campaign.init` can also be passed options that create or fundamentally modify a campaign on-the-fly, in order to allow simple processing to be performed with a single (albeit verbose) command.
The analogy also does not extend to the term "repository"; a ``git`` repository is analogous to a campaign, not a butler data repository.

The ``.campaign.json`` file is a hidden file (and JSON rather than YAML) to reflect the fact that it should be generally be manipulated by the ``pipetask`` tool, not humans running their favorite text editor.

Interfaces
==========

The `Campaign` class is used to represent a campaign directory; instances can be contructed from an existing campaign directory and written out to create or modify a campaign directory.

At least in most cases, `Campaign` methods correspond directly to ``pipetask`` subcommands, and both are described together in the method documentation below.
Most subcommands are expected to be implemented in 2-3 lines, aside from the translation of command-line options to keyword function arguments:

- a call to `Campaign.init` or (usually) `Campaign.load` to construct the `Campaign` instance;
- a call to the method that corresponds directly to the subcommand;
- a call to `Campaign.save` to write the updated campaign to disk.

Because most parameters are common to multiple methods/subcommands, these are described in detail later in :ref:`opts`, using command-line option syntax instead of method parameter syntax to flesh out the command-line interface further.

.. py:class:: Campaign

   .. py:staticmethod:: init(repo: ButlerURI, **kwargs) -> Campaign

      Create and return a new `Campaign` instance.

      This method corresponds directly to the ``init`` subcommand::

         pipetask init REPO URI <OPTIONS>

      That should be implemented simply as::

         campaign = Campaign.init(REPO, **kwargs)
         campaign.save(URI)

      (with ``**kwargs`` generated from command-line options).

      Option groups:

      - :ref:`opts-campaign`:  The :option:`--repo` and :option:`--campaign-dir` options are replaced by the ``REPO`` and ``URI`` positional arguments for this subcommand (only), but the others are still valid here as-is.
        The ``URI`` argument is not relevant for the Python method call, because the campaign is not actually written until `save` is called.

      - :ref:`opts-pipeline`: Optional; if not provided, no pipeline information will be present in the campaign (yet).

      - :ref:`opts-collections`: Optional; if not provided, no input collections will be present in the campaign (yet) and output collection names will be set to their default values.

   .. py:staticmethod:: load(uri: ButlerURI) -> Campaign

      Create a `Campaign` instance corresponding to an existing campaign directory.

      This method has no direct subcommand equivalent, and does not use any of the common option groups.

   .. py:method:: save(uri: ButlerURI)

      Save the campaign to the given directory URI.

      This method has no direct subcommand equivalent, and does not use any of the common option groups.

   .. py:method:: edit(**kwargs)

      Modify an existing campaign in-place.

      This method corresponds directly to the ``edit`` subcommand::

         pipetask edit <OPTIONS>

      This method can be used to set all campaign information that can be specified in `init`, but it can be used on existing campaigns.

      Option groups:

      - :ref:`opts-campaign`
      - :ref:`opts-pipeline`
      - :ref:`opts-collections`

   .. py:method:: register_dataset_types(**kwargs)

      Register all intermediate and output dataset types that would be written by a pipeline, and check that all input dataset types are consistent with the definitions in the pipeline.

      This method corresponds directly to the ``register-dataset-types`` subcommand::

         pipetask register-dataset-types <OPTIONS>

      The action of this method intentionally cannot be performed by providing options to any other method; registering dataset types is something that should be done only rarely, when they are first defined, and attempting to register them with every `pipetask` (as is all too easy to do now) is an antipattern that can lead to incorrectly-defined or typo'd dataset types that are hard to clean up.

      Option groups:

      - :ref:`opts-campaign`
      - :ref:`opts-pipeline`
      - :ref:`opts-collections`

   .. py:method:: prep_run(**kwargs)

      Register the output ``RUN`` collection and write all "init output" datasets to it, including configuration for all tasks.

      This method corresponds directly to the ``prep-run`` subcommand::

         pipetask prep-run <OPTIONS>

      This method creates the `collections.current_run` campaign entry if it does not exist and does not clear it, indicating that the next dataset-writing step should write to that same collection.
      If `collections.chain` is not ``null``, it also registers and defines that collection.

.. _opts:

Common Option Groups
--------------------

.. _opts-campaign:

Campaign Definition
^^^^^^^^^^^^^^^^^^^

These options are used to provide the core campaign definition information.

.. option:: --repo <URI>

   Data repository URI; sets `repo` in ``.campaign.json``.
   Required whenever creating a new campaign.

.. option:: --campaign-dir <URI>

   Campaign directory.

   Except where otherwise noted, this option is optional if and only if the current working directory is a campaign directory.

.. option:: --campaign-name <NAME>

   Name of the campaign; sets `name` in ``.campaign.json``
   If used with an existing campaign, its name is modified.
   If the campaign does not exist and this option is not provided, a name is inferred from its directory.
   Must be provided if creating a new campaign with a directory that includes ``.`` or ``..``.

.. option:: --campaign-docs <STRING>

   Documentation string for the campaign; sets `doc` in ``.campaign.json``
   Always optional, but strongly encouraged for shared campaigns.

.. _opts-pipeline:

Pipeline Definition
^^^^^^^^^^^^^^^^^^^

These options are used to define, modify, or inspect the pipeline.

The behavior of options that modify the pipeline is specified such that repeated invocations with the same set of options are idempotent.

.. option:: -p <URI>, --pipeline <URI>

   URI to a pipeline definition file.
   If the campaign already has a local pipeline, this new pipeline will be added to its imports.
   If the campaign already has a URI to an external pipeline other than this one, a local pipeline will be created that imports both.

.. option:: -t <LABEL>:<TASK>

   ``PipelineTask`` to add to the pipeline.
   This creates a local pipeline if one does not exist.
   If a URI to an external pipeline exists, it will be imported in the new local pipeline.

.. option:: -c <LABEL>:<PARAMETER>=<VALUE>, --config <LABEL>:<PARAMETER>=<VALUE>

   Override a ``pex_config`` parameter value.
   This creates a local pipeline if one does not exist.
   If a URI to an external pipeline exists, it will be imported in the new local pipeline.
   If a local pipeline does exist, this is added as a (YAML) config override to it, replacing an existing override for the same option if it exists and creating a section for the label of necessary.

.. option:: -C <LABEL>:<URI>, --config-file <LABEL>:<URI>

   Apply a ``pex_config`` config override file.
   Affects new and existing pipelines the same way as :option:`-c`.

.. option:: --instrument <NAME>

   Set an instrument whose ``obs``-package config overrides should be loaded.
   This creates a local pipeline if one does not exist, unless a URI to an external pipeline exists and it already has the same instrument.

.. option:: --pipeline-dot <URI>

   Write a GraphViz dot diagram for the pipeline graph to the given file.

.. option:: --write-pipeline [<URI>]

   Write the pipeline YAML file to the given URI, and update the `pipeline` entry in ``.campaign.json`` to point to it.
   If invoked with no argument, or if not provided but other options require a local pipeline to be created, a default filename (``pipeline.yaml``) within the campaign directory is used.

.. _opts-collections:

Collections
^^^^^^^^^^^

These options control the input and output collections.


.. option:: -i <COLLECTION>, --input <COLLECTION>

   Collections to search for input datasets; sets `collections.inputs` in ``.campaign.json``.
   May be passed multiple times (arguments are concatenated), and multiple collections may be passed together by separating them with commas.
   Order matters.
   If the campaign already includes inputs, new inputs are prepended if they are not already included in the existing inputs, and moved to the front if they are already included.

.. option:: --chain <NAME>

   Name of the ``CHAINED`` collection that combines input collections and all output collections; sets `collections.chain` in ``.campaign.json``.

.. option:: --no-chain

   Disable creation of the ``CHAINED`` collection by setting `collections.chain` to ``null`` in ``.campaign.json``.

.. option:: --next-run <NAME>

   Name for the RUN collection that will directly hold the outputs of the next ``RUN``-type collection created.
   Sets `collections.next_run` in ``.campaign.json``; see that for documentation on placeholders and defaults.

.. option:: --set-counter <INT>

   Manually set `collections.counter` in ``.campaign.json``.


Campaign Metadata Schema
========================

The schema for the ``.campaign.json`` file is presented as a flat list below; ``.``-separated names indicate hierarchies in the actual JSON form.
Options are ``str`` unless marked as some other type.

..
   We [ab]use the py:data directive to make a definition list we can link to easily from elsewhere in the document.

.. py:data:: version

   version triplet for the campaign format.
   Always present.

.. py:data:: name

   Name of the campaign.
   Always present; defaulted if necessary.

.. py:data:: doc

   Documentation for the campaign.
   Always present; defaults to ``""``.

.. py:data:: repo

   URI to the data repository.
   Always present, no default, never ``null``.

.. py:data:: collections.inputs

   :type: ``list[str]``

   List of input collections.
   May be absent, but is required to be present (or populated on-the-fly) by some subcommands.

.. py:data:: collections.chain

   Name of the ``CHAINED`` input/output collection.
   Always present; defaulted to `name` if not provided when campaign is created.
   May be set to ``null``, but does not default to ``null``.

   The child collections are set to the sequence ``(current_run, *past_runs, *inputs)`` whenever `~collections.current_run` is updated.

.. py:data:: collections.current_run

   Name for a current ``RUN``-type output collection that already exists and should generally be used by the next step that writes datasets.
   This option is often absent, to indicate that steps that write datasets should create a new ``RUN``-type collection instead.

.. py:data:: collections.next_run

   Name or name pattern used to set `collections.current_run` when needed.
   May contain placeholders, including ``%t`` to insert a timestamp, ``%n`` to insert a per-campaign counter value, and ``%c`` to insert the campaign name.
   Always present; defaults to ``%c/%t``.

.. py:data:: collections.past_runs

   :type: ``list[str]``

   Previous RUN-type collections created as part of this campaign, ordered from the most recent to the oldest.
   Always present; defaults to an empty list.

.. py:data:: collections.counter

   :type: ``int``

   Integer counter to insert into output run names with the ``%n`` placeholder.
   Always present; defaults to ``0``.

.. py:data:: pipeline

   URI to a pipeline YAML definition.
   May be absent, but is required to be present (or populated on-the-fly) by some subcommands.

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
