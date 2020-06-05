`flexible_pipeline` was a Capreolus branch that heavily modified sacred's initialization in order to support profane-style pipelines. This document briefly describes how `profane` differs from the `flexible_pipeline` approach.

## Changes from `flexible_pipeline`
- `Benchmark` now depends on `Collection`, so that the collection does not need to be specified by the user separately.
- `Dependency` declarations now have a `key` in addition to the `module_name`. This allows multiple dependencies on the same module (but with different config options for each). For example, an independently-configured `searcher1` and `searcher2`.
- Previously, each time a dependency was declared the object was added to `provide`. (When a module requests a dependency available in `provide`, the *object* in `provide` is used directly rather than instantiating a new object.) Dependencies are no longer provided by default. When `Dependency.provide_this=True`, the dependency will be provided to all of the declaring module's descendants. For example, say we have a `Rank` task that declares a `Benchmark` dependency with `provide_this=True` followed by a `Searcher` dependency. The `Benchmark` will be instantiated first and then provided to the `Searcher` class. However, the `Benchmark` would not be provided to any of the `Rank` task's parents.
- Similarly, a new `Dependency.provide_children` list allows a `Dependency` to provide some of its children to the declaring module's descendants. For example, in the previous example say the `Benchmark` class also declared `provide_children=["collection"]`. The benchmark's `Collection` object would then be passed to the `Rank` task's `Searcher` dependency.
- `Task` is now a true module and can be used as a dependency like any other module. Coupled with the `provide` changes, this allows tasks to be arbitrarily combined (e.g., a `WeirdRerankTask` that depends on two `RerankTask` modules that each run on separate benchmarks).
- Modules' `__init__` method now fully instantiates the module; previously, `instantiate_from_config` was required to instantiate depenencies. The `provide` argument can be used to specify existing dependency objects rather than creating them. For example, `index=Anserini(config=...); BM25(config=..., provide={"index": index})`.
- In place of the `config` class method, modules declare their config by providing a `config_spec` class attribute containing `ConfigOption` objects. 
- `registry.all_known_modules` has been replaced with a `ModuleRegistry` class, which is instantiated at `base.module_registry`.
- Modules can also be instantiated using `create` method of the module's base class (e.g., `Reranker` or `Benchmark`). By default, modules instantiated with `create` are cached based on their configs, so that identical module objects are re-used.

## DB Queue and Worker
I've revived the run queuing mechanism from before WSDM. To make this work, `EXAMPLE_DB` needs to be a URL pointing to a valid Postgres DB. e.g., `EXAMPLE_DB="postgresql+psycopg2://<user>:<pass>@<hostname>/<db name>"`.
- To queue runs, `run.py` accepts a `-q` option that will cause the specified run to be queued rather than run immediately. e.g., `python run.py -q rank.run with searcher.b=0.123`.
- To launch one of these queued runs, run `worker.py` with no arguments. This script will 1) clear any zombie runs on the current host (i.e., runs marked as running that have non-existent PIDs), and then 2) launch any QUEUED/FAILED run that has failed less than three times.
- To continuously launch available runs, `worker.py` can be put in a shell script or queued with slurm. Placing the loop inside Python isn't great, because past experiences revealed a lot of memory errors with this. Looping in Python until a new run is found would be okay though.