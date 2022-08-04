# helmfile-release-environment-bug

## Issues

### Simple `list` with environment

Here we are listing releases for the `development` environment

Expectation: only the `redis` release is listed

```shell
helmfile --environment development list
```

With `Helmfile version 0.0.0-dev`, specifically `main` branch @ `9af8f1286a7d3804fffe5ee0fbf5e661f23c3693`, all releases
are returned even without a matching environment

```text
NAME            NAMESPACE       ENABLED INSTALLED       LABELS  CHART                   VERSION
database        my-app          true    true                    bitnami/postgres        11.6.22
cache           my-app          true    true                    bitnami/redis           17.0.7 

```

With `helmfile version v0.145.2`, we get the expected result

```text
NAME    NAMESPACE       ENABLED INSTALLED       LABELS  CHART           VERSION
cache   my-app          true    true                    bitnami/redis   17.0.7 

```

Similarly for the `test` release

Expectation: only the `postgres` release is listed

```shell
helmfile --environment test list
```

With `Helmfile version 0.0.0-dev`, specifically `main` branch @ `9af8f1286a7d3804fffe5ee0fbf5e661f23c3693`, all releases
are returned even without a matching environment

```text
NAME            NAMESPACE       ENABLED INSTALLED       LABELS  CHART                   VERSION
database        my-app          true    true                    bitnami/postgres        11.6.22
cache           my-app          true    true                    bitnami/redis           17.0.7 

```

With `helmfile version v0.145.2`, we get the expected result

```text
NAME            NAMESPACE       ENABLED INSTALLED       LABELS  CHART                   VERSION
database        my-app          true    true                    bitnami/postgres        11.6.22

```

NOTE: this is **not** restricted to the `list` command, other commands are also affected.

### `list` with Helmfile State Values

Similar to above, but supplying state values via `--state-values-set`, using the current release of Helmfile: `v0.145.2`

Expectation: only the `redis` release is listed

```shell
helmfile --environment development --state-values-set example=value list
```

With `helmfile version v0.145.2`

```text
NAME            NAMESPACE       ENABLED INSTALLED       LABELS  CHART                   VERSION
database        my-app          true    true                    bitnami/postgres        11.6.22
cache           my-app          true    true                    bitnami/redis           17.0.7 

```

Again, the situation with the `test` environment is the same, i.e. all releases are listed.

## Root Cause

The change to the behaviour where releases for environments other than the requested one has been `git bisect`ed down
and found to be a result of [Use cobra](https://github.com/helmfile/helmfile/pull/234)

Digging into this, [a.Set](https://github.com/helmfile/helmfile/blob/85ade797abf278cbb6acebe26c1b66b39c7a98ce/pkg/app/app.go#L988)
used to be `nil` without state values set, however the switch to Cobra changed this to an empty
`map[string]interface{}`, which is then always added to the `opts.Environment.OverrideValues`. This results in a
mismatching environment being entirely ignored, templating being applied to all helmfiles/releases using the specified
environment anyway, and all releases being returned.

This helps to confirm the reason for the other observed case: prior to the Cobra change, the same behaviour is observed
if any helmfile state values are supplied. In both cases, environment override values are applied, and this causes
non-applicable releases (based on environment) to be included.
