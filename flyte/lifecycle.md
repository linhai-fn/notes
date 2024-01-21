# Workflow Lifecycle

## Register

```bash
pyflyte register <workflows> --image <image>
```

1. `flytekit.clis.sdk_in_container.register.register()` passes `workflows` and `image` to `flytekit.tools.repo.register()`.
2. `flytekit.tools.repo.register()` serializes the `workflows` together with `image` to `registrable_entities`
   by calling `load_packages_and_modules()` -> `serialize()` -> `get_registrable_entities()`
3. Each entity in `registrable_entities` is registered to the remote by calling `flytekit.remote.remote.FlyteRemote.raw_register()`.
4. The `flytekit.remote.remote.FlyteRemote.raw_register()` calls `create_task()` or `create_workflow()` or `create_launch_plan()`
   in `flytekit.clients.friendly.SynchronousFlyteClient`. The `SynchronousFlyteClient` is a low-level client that users can use
   to make direct gRPC service calls to the control plane. But it's more user-friendly interface than the
   `flytekit.clients.raw.RawSynchronousFlyteClient`.
5. The `create_task()` or `create_workflow()` or `create_launch_plan()` in the `SynchronousFlyteClient` then convert everything
   to protobuf by calling each entity's `.to_flyte_idl()` method. And then send the `TaskCreateRequest` or
   `WorkflowCreateRequest` or `LaunchPlanCreateRequest` through gRPC calls to the control plane.
6. Flyte Admin creates task/workflow/launch plan definition in the Admin database.

## Execute

```python
execution = remote.execute(entity, inputs=inputs)
```

### Create Execution

1. Depending on the entity, the `flytekit.remote.remote.FlyteRemote.execute()` will call different methods.
   For `PythonTask`/`WorkflowBase`/`LaunchPlan`, the `execute_local_task()`/`execute_local_workflow()`/`execute_local_launch_plan()`
   will be invoked. Those methods will register task/workflow/launch plan first.
2. Eventually, all types of entities will be converted to either `FluteTask` or `FlyteLaunchPlan`. And the
   `FlyteRemote.execute_remote_task_lp()` will be invoked.
3. `FlyteRemote.execute_remote_task_lp()` will call `FlyteRemote._execute()`.
4. All the `inputs` will be translated to `Literal`s with `flytekit.core.type_engine.TypeEngine.to_literal()`.
    1. For each input values, the `TypeEngine` will `get_transformer()` and invoke their `transformer.to_literal()`.
    2. For example, if a `path` (with `str` type) is in the `inputs`. The `FlyteFilePathTransformer.to_literal()`
       will convert the `path` to a `Literal` of `Blob`. If the `path` is a remote path (e.g. starts with `gs://`),
       the `Blob.uri` will be set to `path`. If the `path` is a local path, the file will be uploaded to a
       random `remote_path` (obtained from `flytekit.core.data_persistence.FileAccessProvider.put_raw_data()`). And the
       `Blob.uri` will be set to the `remote_path`.
5. The `Literal`s will then be converted to `flytekit.models.literals.LiteralMap`.
6. An `ExecutionSpec` will be generated using the entity.
7. The `flytekit.clients.friendly.SynchronousFlyteClient.creat_execution()` is then invoked.
    1. This will convert both `ExecutionSpec` and `inputs` (a `LiteralMap`) to protobuf using their `.to_flyte_idl()` methods.
    2. It will then send the `ExecutionCreateRequest` through gRPC calls.

### Run Execution

1. The FlytePropeller will call the method `BuildResource` first.
2. The FlytePropeller will run a pod with `pyflyte-execute` as entrypoint.
3. `pyflyte-execute` calls `flytekit.bin.entrypoint.execute_task_cmd()` -> `_execute_task()` -> `_handle_annotated_task()`
   -> `_dispatch_execute()`. The main parameters are `inputs` (a remote path point to the input protobuf), `output_prefix`
   (where to write primitive outputs), and `raw_output_data_prefix` (where to write offload data such as files,
   directories, dataframes).
4. `_dispatch_execute()` performs the following steps.
    1. Download `inputs` and load into a `LiteralMap`. It will download the input protobuf `inputs.pb` to a random local path
       using `flytekit.core.data_persistence.FileAccessProvider.get_data()`. Then load the input protobuf using
       `flytekit.core.utils.load_proto_from_file()`. The convert it to `LiteralMap` using
       `flytekit.models.literals.LiteralMap.from_flyte_idl()`.
    2. Invoke task by calling `flytekit.core.base_task.PythonTask.dispatch_execute()`. The `dispatch_execute()` performs the
       following steps.
        1. Convert `input` `Literal`s to python types by calling `flytekit.core.type_engine.TypeEngine.literal_map_to_kwargs()`.
        2. The `TypeEngine.literal_map_to_kwargs()` calls each `Literal`'s `transformer.to_python_value()`.
        3. For example, if the `input` has a python type `FlyteFile`, the `FlyteFilePathTransformer.to_python_value()` gets
           the `Blob.uri`. If the `Blob.uri` is a remote path, it will get a random `local_path` using
           `flytekit.core.data_persistence.FileAccessProvider.get_random_local_path()`. Then create a `FlyteFile` by setting
           the `path` to `local_path`, `_downloader` to `flytekit.core.data_persistence.FileAccessProvider.get_data()`
           , `_remote_source` to the `uri`.
        4. Call `execute()` to execute the actual code we want to run and generate `output`. For example, a `PythonFunctionTask`
           will just execute the `_task_function`.
        5. Convert `outputs` to `LiteralMap` using `PythonTask._output_to_literal_map()`. It uses `TypeEngine.to_literal()`
           and `flytekit.models.literals.LiteralMap`.
    3. Convert `output` (a `LiteralMap`) to protobuf using its `.to_flyte_idl()` method. Then write the protobuf using
       `flytekit.core.utils.write_proto_to_file()`. Put the protobuf file to remote `output_prefix` using
       `flytekit.core.data_persistence.FileAccessProvider.put_data()`.

## Resources

- [Understand the Lifecycle of a Flyte Workflow](https://docs.flyte.org/en/latest/concepts/workflow_lifecycle.html)
