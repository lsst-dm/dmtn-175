
:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   The ``pipetask`` tool included in the Gen3 middleware for running `PipelineTasks<lsst.pipe.base.PipelineTask>` is often opaque and hard to use.  This is partly the result of history; the original interface was much more straightforward, but it assumed a very different collections model than what Butler has now, and the changes made to adapt it to the new collections system did not go deep enough.
   But it also suffers from the fact that many of its steps are "leaky"; information provided in one step (e.g. input collections used for `~lsst.pipe.base.QuantumGraph`) cannot be straightforwardly passed to later steps (e.g. execution of that `~lsst.pipe.base.QuantumGraph`), forcing it to rely on the user to provide consistent arguments and limiting the guarantees we can make about behavior in the presence of user error.
   Finally, the ``pipetask`` interface is currently only really usable from the command-line, making it hard to write unit tests for concrete `PipelineTasks<lsst.pipe.base.PipelineTask>`; even if we were to keep the command-line interface the same, we need to refactor the implementation to provide a usable Python API.
   This technote will provide a detailed design sketch for a new ``pipetask`` interface that attempts to address these problems, centered around a ``git``-like "campaign directory" concept that will be used to save state between different ``pipetask`` invocations.

Preliminaries
=============

The reader is assumed to be familiar with the Gen3 butler's main organizational concepts (dimensions, data IDs, datasets, dataset types, and collections), the types of collections, and the basic functionality of the current ``pipetask`` tool.


Conceptual Overview
===================

A "campaign" is a directory with (at least) a ``.campaign.json`` file that tracks the what *will be* run, and how.
As processing proceeds, this will also be the (default) directory where pipeline definitions, `QuantumGraphs<lsst.pipe.base.QuantumGraph>`, and logs are written.
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

      This method corresponds directly to the ``init`` subcommand:

      .. code-block:: sh

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

      This method corresponds directly to the ``edit`` subcommand:

      .. code-block:: sh

         pipetask edit <OPTIONS>

      This method can be used to set all campaign information that can be specified in `init`, but it can be used on existing campaigns.
      When used to modify input collections after a `~lsst.pipe.base.QuantumGraph` has already been built, `refresh_quantum_graph` should almost always been run afterwards.
      When used to modify the pipeline after a `~lsst.pipe.base.QuantumGraph` has already been built, `build_quantum_graph` must be run for the changes to have any effect.

      Option groups:

      - :ref:`opts-campaign`
      - :ref:`opts-pipeline`
      - :ref:`opts-collections`

   .. py:method:: status(**kwargs)

      Print information about the current state of the pipeline to STDOUT.

      This method corresponds directly to the ``status`` subcommand:

      .. code-block:: sh

         pipetask status <OPTIONS>

      Option Groups:

      - :ref:`opts-pipeline`: :option:`--pipeline-dot` only, and only if the campaign already contains a pipeline.
      - :ref:`opts-qg`: :option:`--qg-dot` only, and only if the campaign already contains a `~lsst.pipe.base.QuantumGraph`.

      Other Options: **TODO**

   .. py:method:: register_dataset_types(**kwargs)

      Register all intermediate and output dataset types that would be written by a pipeline, and check that all input dataset types are consistent with the definitions in the pipeline.

      This method corresponds directly to the ``register-dataset-types`` subcommand:

      .. code-block:: sh

         pipetask register-dataset-types <OPTIONS>

      The action of this method intentionally cannot be performed by providing options to any other method; registering dataset types is something that should be done only rarely, when they are first defined, and attempting to register them with every `pipetask` (as is all too easy to do now) is an antipattern that can lead to incorrectly-defined or typo'd dataset types that are hard to clean up.

      This command does not require the campaign to already have a `~lsst.pipe.base.QuantumGraph`, and does not create one.
      However, if a `~lsst.pipe.base.QuantumGraph` does exist, it is used instead of the pipeline to determine the tasks whose configurations should be saved, allowing campaigns to use imported `QuantumGraphs<lsst.pipe.base.QuantumGraph>` with no pipeline information at all.

      Option groups:

      - :ref:`opts-campaign`: :option:`campaign-dir` only, and only to find an existing campaign.

   .. py:method:: build_quantum_graph(**kwargs)

      Build a `~lsst.pipe.base.QuantumGraph` for the campaign.

      This method corresponds directly to the ``qg build`` subcommand:

      .. code-block:: sh

         pipetask qg build <OPTIONS>

      Option groups:

      - :ref:`opts-campaign`
      - :ref:`opts-pipeline`
      - :ref:`opts-collections`
      - :ref:`opts-qg`, except:

         - :option:`--allow-pruning` (pruning is a fundamental part of building a graph and cannot be disabled)
         - :option:`--refresh` (a graph is implicitly refreshed when it is built, so other options normally enabled by :option:`--refresh` are allowed).

   .. py:method:: import_quantum_graph(uri: ButlerURI, **kwargs)

      Import an existing `~lsst.pipe.base.QuantumGraph` into the campaign.

      This method corresponds directly to the ``qg import`` subcommand:

      .. code-block:: sh

         pipetask qg import <URI> <OPTIONS>

      Option groups:

      - :ref:`opts-campaign`
      - :ref:`opts-qg`, except :option:`--data-query`

      Passing :option:`--refresh` to this method/subcommand performs the refresh after the import, not before.

   .. py:method:: refresh_quantum_graph(**kwargs)

      Refresh the campaign's `~lsst.pipe.base.QuantumGraph` by querying again for its input and intermediate datasets.

      This method corresponds directly to the ``qg refresh`` subcommand:

      .. code-block:: sh

         pipetask qg refresh <OPTIONS>

      Refreshing a `~lsst.pipe.base.QuantumGraph` ensures that any embedded `~lsst.daf.butler.DatasetRef` objects are resolved if and only if they can be found in the `collections.inputs`, `collections.past_runs`, and `collections.current_run` collections.

      A campaign's `~lsst.pipe.base.QuantumGraph` should always be refreshed whenever the collections used to build it are changed.

      When an overall-input (i.e. non-intermediate) dataset cannot be resolved (by definition, these datasets must have been resolved when the `~lsst.pipe.base.QuantumGraph` was originally built) some aspects of the graph generation logic must be re-run, which can result in some quanta being dropped.
      The :option:`--drop-existing-in` option can also be used to drop quanta whose outputs already exist.

      Option groups:

      - :ref:`opts-campaign`: :option:`campaign-dir` only, and only to find an existing campaign.
      - :ref:`opts-qg`, except:

         - :option:`--data-query`
         - :option:`--extend-qg`
         - :option:`--refresh` (implied, so all options normally enabled by :option:`--refresh` are allowed).

   .. py:method:: prep(**kwargs)

      Register a new output ``RUN`` collection, write all "init output" datasets to it, including software versions and configuration for all tasks.

      This method corresponds directly to the ``prep`` subcommand:

      .. code-block:: sh

         pipetask prep <OPTIONS>

      This method creates the `collections.current_run` campaign entry if it does not exist and does not clear it when finished, indicating that the next dataset-writing step should write to that same collection.
      If `collections.current_run` does already exist, it writes init output datasets if they do not exist and checks them for consistency if they do.
      If `collections.chain` is not ``null``, it also [re]registers and [re]defines that collection.

      This command does not require the campaign to already have a `~lsst.pipe.base.QuantumGraph`, and does not create one.
      However, if a `~lsst.pipe.base.QuantumGraph` does exist, it is used instead of the pipeline to determine the tasks whose configurations should be saved, allowing campaigns to use imported `QuantumGraphs<lsst.pipe.base.QuantumGraph>` with no pipeline information at all.

      Option groups:

      - :ref:`opts-campaign`
      - :ref:`opts-pipeline`
      - :ref:`opts-collections`
      - :ref:`opts-execution`, except:

         - :option:`-j`, :option:`--processes`: irrelevant, because no quanta are executed.
         - :option:`--finish`, :option:`--no-finish`: :option:`--no-finish` is implied.

   .. py::method:: run(**kwargs)

      Run the campaign's `~lsst.pipe.base.QuantumGraph`, creating it if needed.

      This method corresponds directly to the ``run`` subcommand:

      .. code-block:: sh

         pipetask run <OPTIONS>

      This operation will create a `~lsst.pipe.base.QuantumGraph` if one does not exist, but does not require the campaign to have a pipeline if it has a `~lsst.pipe.base.QuantumGraph` (which thus must have been imported).

      High-level interfaces like this method and subcommand should always invoke `prep` before actually running any quanta (but after creating the `~lsst.pipe.base.QuantumGraph`, if one does not exist).
      This ensures that the output ``RUN``-type collection exists and that any provenance datasets it holds are consistent with the current configuration and environment.
      We also need a lower-level interface (at least in Python; *maybe* on the command-line, too, perhaps as a completely different executable) that instead *assumes* that `collections.current_run` exists and holds the right provenance datasets, for use by e.g. batch jobs that just want to run some already-exising quanta, but it's important that those interfaces are only called by higher-level code that itself ensures that `prep` is called appropriately.

      Option groups:

      - :ref:`opts-campaign`
      - :ref:`opts-pipeline`
      - :ref:`opts-collections`
      - :ref:`opts-execution`

   .. py::method:: pop(n: int = 0, **kwargs)

      Drop existing ``RUN``-type collections from the campaign and redefine its ``CHAINED`` collection (if one exists) accordingly.

      This method corresponds directly to the ``pop`` subcommand:

      .. code-block:: sh

         pipetask pop [INT] <OPTIONS>

      If ``n == 0`` (default), `collections.current_run` is cleared if it is set.
      If ``n > 0``, the first ``n`` collections in ``collections.past_runs`` are also removed.

      If `collections.chain` is not ``null``, the ``CHAINED``-type collection for this campaign is updated.

      Option groups:

      - :ref:`opts-campaign`: :option:`campaign-dir` only, and only to find an existing campaign.
      - : ref:opts-collections`:,: :option:`--unstore` and :option:`--purge` only.

   .. py::method:: clean(purge: bool = False)

      Remove datasets and possibly collections were created by this campaign but have since been dropped.

      This method corresponds directly to the ``clean`` subcommand:

      .. code-block:: sh

         pipetask clean <OPTIONS>

      This operation computes the "dropped" collections as those that are in `collections.created` but not (currently) in any of `collections.chain`, `collections.past_runs`, `collections.inputs`, or `collections.current_run`.

      If possible, we should make this remove directories that correspond to unstored ``RUN``-type collections, especially if those are in the campaign directory themselves.

      Option groups:

      - :ref:`opts-campaign`, :option:`campaign-dir` only, and only to find an existing campaign.
      - : ref:opts-collections`, :option:`--purge` only (:option:`--unstore` is the implied default behavior).


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
   If a collection that is already in `collections.past_runs` is included, it is automatically removed from `collections.past_runs`.

.. option:: --prepend-inputs

   Instead of replacing `collections.inputs` with the values given by all :option:`-i` arguments, prepend them if they are not already included in the existing inputs, and move them to the front if they are already included.

.. option:: --chain <NAME>

   Name of the ``CHAINED`` collection that combines input collections and all output collections; sets `collections.chain` in ``.campaign.json``.

.. option:: --no-chain

   Disable creation of the ``CHAINED`` collection by setting `collections.chain` to ``null`` in ``.campaign.json``.

.. option:: --next-run <NAME>

   Name for the RUN collection that will directly hold the outputs of the next ``RUN``-type collection created.
   Sets `collections.next_run` in ``.campaign.json``; see that for documentation on placeholders and defaults.

.. option:: --set-counter <INT>

   Manually set `collections.counter` in ``.campaign.json``.

.. _opts-qg:

QuantumGraphs
^^^^^^^^^^^^^

.. option:: --qg-dot <URI>

   Write a GraphViz dot diagram for the QuantumGraph to the given file.

.. option:: --write-qg [<URI>]

   Write the `~lsst.pipe.base.QuantumGraph` file to the given URI, and update the `quantum_graph` entry in ``.campaign.json`` to point to it.
   If invoked with no argument, or if not provided but other options require a local `~lsst.pipe.base.QuantumGraph` to be created, a default filename (using the campaign name) within the campaign directory is used.

.. option:: -d <QUERY>, --data-query <QUERY>

   Provide a SQL-like query expression that constrains the data IDs of the `~lsst.pipe.base.QuantumGraph`.

.. option:: --extend-qg

   If the campaign is already associated with a `~lsst.pipe.base.QuantumGraph`, extend it when building or importing a new one, instead of replacing it.

.. option:: --refresh

   Equivalent to running `refresh_quantum_graph` immediately before (usually) or after (where noted) some other method.

.. option:: --trim-existing-in [INPUTS|CAMPAIGN|RUN]

   Remove quanta from the `~lsst.pipe.base.QuantumGraph` when all of their outputs already exist in the given collection category:

   ``RUN``
      Trim a quantum if all of its outputs exist in `collections.current_run`; do nothing if `collections.current_run` is not set.
   ``CAMPAIGN``
      Trim a quantum if all of its outputs exist in either `collections.current_run` or `collections.past_runs`, i.e. any ``RUN``-type collection produced by this campaign that has not been discarded from it;
   ``INPUTS``
      Trim a quantum if all of its outputs exist in any of `collections.current_run`, `collections.past_runs`, or `collections.inputs`.

   Except where otherwise noted, :option:`--refresh` must also be passed for this option to be valid.

.. option:: --allow-pruning

   When refreshing a `~lsst.pipe.base.QuantumGraph`, allow a quantum to be removed if one or more of its input datasets cannot be resolved and the `~lsst.pipe.base.PipelineTask` indicates that the quantum is not viable without them.

   When this option is not given and an nonviable quantum is found, the refresh operation fails but the campaign and its `~lsst.pipe.base.QuantumGraph` are not modified.

   Except where otherwise noted, :option:`--refresh` must also be passed for this option to be valid.

.. option:: --allow-empty

   When building a `~lsst.pipe.base.QuantumGraph` or refreshing one with :option:`--allow-pruning` or `--drop-existing-in`, allow the graph to end up with no quanta.
   When this option is not given, an empty graph is treated as an error condition, and the campaign and its `~lsst.pipe.base.QuantumGraph` are not modified.

   Except where otherwise noted, :option:`--refresh` must also be passed for this option to be valid.

.. _opts-execution:

Execution
---------

These options control how quanta are executed and how ``RUN``-type collections are created and manipulated.

.. option:: -j <INT>, --processes <INT>

   Number of processes used for local (single-node) execution.
   Batch-execution extensions are encouraged to use this to control the total number of processes if they have a mode in which that is all that is provided.

.. option:: --finish, --no-finish

   Controls whether or not to clear `collections.current_run` after all requested quanta are executed successfully, and hence whether the *next* invocation ``pipetask`` that writes to a ``RUN``-type collection will use the same one.
   The default behavior depends on other options and the previous state of `collections.current_run`:

   - If `collections.current_run` was previously set and is being used (e.g. :option:`--push` was not passed), or if the full `~lsst.pipe.base.QuantumGraph` was not run, the default is to leave `collections.current_run` in place for the next invocation.

   - If `collections.current_run` was not previously set, or if other options (e.g. :option:`--push`) were used to create a new ``RUN``-type collection anyway, the default is to clear `collections.current_run` so the next invocation will create a new ``RUN``-type collection as well.

.. option:: --push

   Create a new ``RUN``-type collection for output datasets created by this method/subcommand.
   If `collections.current_run` is not set, this is the default behavior.
   If it is set, the value of `collections.current_run` is inserted at the front of `collections.past_runs`.

.. option:: --replace

   Create a new ``RUN``-type collection for output datasets created by this method/subcommand., dropping `collections.current_run`.
   It is an error to pass this option if `collections.current_run` is not set.

.. option:: --continue

   If `collections.current_run` is not set, remove the first entry from `collections.past_runs` (which must not be empty) and set `collections.current_run` to that.
   Does nothing if `collections.current_run` is already set.

.. option:: --restart

   Drop *all* runs in `collections.past_runs` and `collections.current_run` (if it exists), and create and prep a new one to contain all outputs.

.. option:: --unstore

   If an output collection is dropped by this action (via :option:`--replace`, :option:`--restart`, or `Campaign.pop`), remove its dataset artifacts from the datastore only.
   Not valid if no collections can be dropped by this operation.

.. option:: --purge

   If an output collection is dropped by this action (via :option:`--replace`, :option:`--restart`, or `Campaign.pop`), remove the collection and its datasets entirely from both the registry and the datastore.
   Supersedes :option:`--unstore`.
   Not valid if no collections can be dropped by this operation.

.. option:: --skip-existing-in [INPUTS|CAMPAIGN|RUN]

   Do not execute quanta for which all outputs already exist in the given collection category.

   Unlike :option:`--trim-existing-in`, this does not modify the `~lsst.pipe.base.QuantumGraph`, but the argument choices have the same definition.

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
   Setting it to ``null`` does not automatically delete the collection if it has already been created, but `Campaign.clean` will delete it.

   The child collections are set to the sequence ``(current_run, *past_runs, *inputs)`` whenever `~collections.current_run` is updated.

.. py:data:: collections.current_run

   Name for a current ``RUN``-type output collection that already exists and should generally be used by the next step that writes datasets.
   This entry is often absent or ``null`` (these are equivalent), to indicate that steps that write datasets should create a new ``RUN``-type collection instead.

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

.. py:data:: collections.created

   All collections created by this campaign.
   This includes ``CHAINED`` collections.

.. py:data:: pipeline

   URI to a pipeline YAML definition.
   May be absent, but is required to be present (or populated on-the-fly) by some subcommands.

.. py:data:: quantum_graph.uri

   URI to a saved `~lsst.pipe.base.QuantumGraph` object.
   May be absent, but is required to be present (or populated on-the-fly) by some subcommands.

.. py:data:: quantum_graph.dirty

   :type: ``bool``

   Status flag set if the input collections or pipeline definition have changed since the `~lsst.pipe.base.QuantumGraph` was built.

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
