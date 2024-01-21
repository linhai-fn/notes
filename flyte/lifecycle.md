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
execution = remote.execute(task, inputs=inputs)
```

## Resources

- [Understand the Lifecycle of a Flyte Workflow](https://docs.flyte.org/en/latest/concepts/workflow_lifecycle.html)
