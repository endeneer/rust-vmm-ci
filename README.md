# rust-vmm-ci

The `rust-vmm-ci` repository contains [integration tests](#integration-tests)
and [Buildkite pipeline](#buildkite-pipeline) definitions that are used for
running the CI for all rust-vmm crates.

Having a centralized place for the tests is one of the enablers for keeping the
same quality standard for all crates in rust-vmm.

## Getting Started with rust-vmm-ci

To run the integration tests defined in the pipeline as part of the CI:

1. Add rust-vmm-ci as a git submodule to your repository

```bash
# Add rust-vmm-ci as a submodule. This will point to the latest rust-vmm-ci
# commit from the main branch. The following command will also add a
# `.gitmodules` file and the `rust-vmm-ci` to the index.
git submodule add https://github.com/rust-vmm/rust-vmm-ci.git
# Commit the changes to your repository so that the CI can run using the
# rust-vmm-ci pipeline and tests.
git commit -s -m "Added rust-vmm-ci as submodule"
```

2. Create the coverage test configuration file named
`coverage_config_ARCH.json` in the root of the repository, where `ARCH` is the
architecture of the machine.
There are two coverage test configuration files, one per each platform.
The example of the configuration file for the `x86_64` architecture can be
found in
[coverage_config_x86_64.json.sample](coverage_config_x86_64.json.sample),
and the example of the configuration file for the `aarch64` architecture can be
found in
[coverage_config_aarch64.json.sample](coverage_config_aarch64.json.sample).

The json must have the following fields:

- `coverage_score`: The coverage of the repository.
- `exclude_path`: This field is used for excluding files from the report. It
  should be used to exclude autogenerated files. Files in `exclude_path` are
  separated by one comma. If the repository does not have any autogenerated
  files, `exclude_path` should be an empty string.
- `crate_features`: `cargo kcov` does not build crate features by default. To
  get the coverage report including optional features, these need to be
  specified in `crate_features` separated by comma. If the crate does not have
  any features, this field should be empty.

This file is required for the coverage integration so it needs to be added
to the repository as well.

3. Create a new pipeline definition in Buildkite. For this step ask one of the
rust-vmm Buildkite [admins](CODEOWNERS) to create one for you. The process is explained
[here](https://github.com/rust-vmm/community/blob/main/docs/setup_new_repo.md#set-up-ci).

4. There is a script that autogenerates a dynamic Buildkite pipeline. Each step
in the pipeline has a default timeout of 5 minutes. To run the CI using this dynamic pipeline,
you need to add a step that is uploading the rust-vmm-ci pipeline:
```bash
./rust-vmm-ci/.buildkite/autogenerate_pipeline.py | buildkite-agent pipeline upload
```
This allows overriding some values and extending others through environment 
variables. 
- `X86_LINUX_AGENT_TAGS`: overrides the tags by which the x86_64 linux agent is
  selected; the default values are

  `{"os": "linux", "platform": "x86.metal"}`
- `AARCH64_LINUX_AGENT_TAGS`: overrides the tags by which the aarch64 linux
  agent is selected. The default values are

  `{"os": "linux", "platform": "arm.metal"}`
- `DOCKER_PLUGIN_CONFIG`: specifies additional configuration for the docker
  plugin. For available configuration, please check the
  https://github.com/buildkite-plugins/docker-buildkite-plugin.
- `TESTS_TO_SKIP`: specifies a list of tests to be skipped.
- `TIMEOUTS_MIN`: overrides the timeout value for specific tests.
- `DEFAULT_AGENT_TAG_HYPERVISOR`: sets the hypervisor on which all the tests in
  the pipeline run. By default, the selected hypervisor is KVM because the
  hosts running KVM at the time of this change showed better performance and
  experienced timeouts less often. NOTE: This will not override the hypervisor
  defined at the test step level. If a test already defines a hypervisor tag
  that will remain intact.

The variable `TESTS_TO_SKIP` is specified as a JSON list with the names
of the tests to be skipped. The variable `TIMEOUTS_MIN` is a dictionary where
each key is the name of a test and each value is the number of minutes for the
timeout. The other variables are specified as dictionaries, where the first key
is `tests` and its value is a list of test names where the configuration should
be applied; the second key is `cfg` and its value is a dictionary with the
actual configuration.

For example, we can skip the test `commit-format`, have a timeout of 30 minutes
for the test `style` and extend the docker plugin specification as follows:
```shell
TESTS_TO_SKIP='["commit-format"]' TIMEOUTS_MIN='{"style": 30}' DOCKER_PLUGIN_CONFIG='{
    "tests": ["coverage"],
    "cfg": {
        "devices": [ "/dev/vhost-vdpa-0" ],
        "privileged": true
    }
}' ./rust-vmm-ci/.buildkite/autogenerate_pipeline.py | buildkite-agent pipeline upload
```

For most use cases, overriding or extending the configuration is not necessary. We may
want to do so if, for example, the platform needs a custom device that is not available
on the existing test instances or if we need a specialized hypervisor.

5. The code owners of the repository will have to setup a WebHook for
triggering the CI on
[pull request](https://developer.github.com/v3/activity/events/types/#pullrequestevent)
and [push](https://developer.github.com/v3/activity/events/types/#pushevent)
events.

## Buildkite Pipeline

The [Buildkite](https://buildkite.com) pipeline is the definition of tests to
be run as part of the CI. It includes steps for running unit tests and linters
(including coding style checks), and computing the coverage.

Currently the tests can run on Linux `x86_64` and `aarch64` hosts.

Example of step that checks the build:

```yaml
steps:
- label: build-gnu-x86_64
  command: cargo build --release
  retry:
    automatic: false
  agents:
    os: linux
    platform: x86_64.metal
  plugins:
  - docker#v3.8.0:
      image: rustvmm/dev:v16
      always-pull: true
  timeout_in_minutes: 5
```

To see all steps in the pipeline check the output of the 
[.buildkite/autogenerate_pipeline.py](.buildkite/autogenerate_pipeline.py) script.

### Custom Pipeline

Some crates might need to test functionality that is specific to that
particular component and thus cannot be added to the common pipeline.

In this situation, the repositories need to create a JSON file with a custom
test configuration. The preferred path is `.buildkite/custom-tests.json`.

For example to test the build with one non-default
[feature](https://doc.rust-lang.org/1.19.0/book/first-edition/conditional-compilation.html)
enabled, the following configuration can be added:

```json
{
  "tests": [
    {
      "test_name": "build-bzimage",
      "command": "cargo build --release --features bzimage",
      "platform": [
        "x86_64"
      ]
    }
  ]
}
```

To run this custom pipeline, you need to add a step that is uploading it in Buildkite. The same
script that autogenerates the main pipeline can be used with the option 
`-t PATH_TO_CUSTOM_CONFIGURATION`:

```bash
./rust-vmm-ci/.buildkite/autogenerate_pipeline.py -t .buildkite/custom-tests.json | buildkite-agent pipeline upload
```

## Integration Tests

In addition to the one-liner tests defined in the
[Buildkite Pipeline](#buildkite-pipeline), the rust-vmm-ci also has more
complex tests defined in [integration_tests](integration_tests).

### Test Profiles

The integration tests support two test profiles:

- **devel**: this is the recommended profile for running the integration tests
  on a local development machine.
- **ci** (default option): this is the profile used when running the
  integration tests as part of the the Continuous Integration (CI).

The test profiles are applicable to
[`pytest`](https://docs.pytest.org/en/latest/), the integration test framework
used with rust-vmm-ci. Currently only the
[coverage test](tests/test_coverage.py) follows this model as all the other
integration tests are run using the Buildkite pipeline.

The difference between is declaring tests as passed or failed:

- with the **devel** profile the coverage test passes if the current coverage
  is equal or higher than the upstream coverage value. In case the current
  coverage is higher, the coverage file is updated to the new coverage value.
- with the **ci** profile the coverage test passes only if the current coverage
  is equal to the upstream coverage value.

Further details about the coverage test can be found in the
[Adaptive Coverage](#adaptive-coverage) section.

### Adaptive Coverage

The line coverage is saved in [tests/coverage](tests/coverage). To update the
coverage before submitting a PR, run the coverage test:

```bash
CRATE="kvm-ioctls"
# NOTE: This might not be the latest container version, you can check which one we're using
# by looking into the .buildkite/autogenerate_pipeline.py file.
LATEST=16
docker run --device=/dev/kvm \
           -it \
           --security-opt seccomp=unconfined \
           --volume $(pwd)/${CRATE}:/${CRATE} \
           rustvmm/dev:v${LATEST}
cd ${crate}
pytest --profile=devel rust-vmm-ci/integration_tests/test_coverage.py
```

If the PR coverage is higher than the upstream coverage, the coverage file
needs to be manually added to the commit before submitting the PR:

```bash
git add tests/coverage
```

Failing to do so will generate a fail on the CI pipeline when publishing the
PR.

**NOTE:** The coverage file is only updated in the `devel` test profile. In
the `ci` profile the coverage test will fail if the current coverage is higher
than the coverage reported in [tests/coverage](tests/coverage).

### Performance tests

`rust-vmm-ci` includes an integration test that can run a battery of
benchmarks at every pull request, comparing the results with the tip of the
upstream `main` branch. The test is not included in the default Buildkite
pipeline. Each crate that requires the test to be run as part of the CI must
add a [custom pipeline](#custom-pipeline).

An example of a pipeline that runs the test for ARM platforms and prints the
results:

```yaml
steps:
- label: bench-aarch64
  command: pytest rust-vmm-ci/integration_tests/test_benchmark.py -s
  retry:
    automatic: false
  agents:
    os: linux
    platform: arm.metal
  plugins:
  - docker#v3.8.0:
      image: rustvmm/dev:v16
      always-pull: true
```

The test requires [`criterion`](https://github.com/bheisler/criterion.rs)
benchmarks to be exported by the crate. The test expects the entry point
into the performance benchmarks to be named `main`. In other words, the
following configuration is expected in `Cargo.toml`:

```toml
[[bench]]
name = "main"
```

All benchmarks need to be collected in a main.rs file placed in `benches/`.

`criterion` collects performance results by running a function for a
user-configured number of iterations, timing the runs, and applying statistics.
The individual benchmark tests must be added in the crate. They can be run
outside the CI with:

```bash
cargo bench [--all-features] OR [--features <features>]
```

`rust-vmm-ci` uses [`critcmp`](https://github.com/BurntSushi/critcmp) to
compare the results yielded by `cargo bench --all-features` on the PR being
tested with those from the tip of the upstream `main` branch. The test
runs `cargo bench` twice, once on the current `HEAD`, then again after
`git checkout origin/main`. `critcmp` takes care of the comparison, making
use of `criterion`'s stable format for
[output files](https://bheisler.github.io/criterion.rs/book/user_guide/csv_output.html).
The results are printed to `stdout` and can be visually inspected in the
pipeline output. In its present form, the test cannot fail.

To run the test locally:

```bash
docker run --device=/dev/kvm \
           -it \
           --security-opt seccomp=unconfined \
           --volume $(pwd)/${CRATE}:/${CRATE} \
           rustvmm/dev:v${LATEST}
cd ${CRATE}
pytest rust-vmm-ci/integration_tests/test_benchmark.py -s
```

Note that performance is highly dependent on the underlying platform that the
tests are running on. The raw numbers obtained are likely to differ from their
counterparts on a CI instance.

### Running the tests locally
To run the integration tests locally, you can run the following from the crate you need to test.
You can find the latest container version in the 
[script](.buildkite/autogenerate_pipeline.py)
that autogenerates the pipeline. For example:
```bash
cd ~/vm-superio
CRATE="vm-superio"
# NOTE: This might not be the latest container version, you can check which one we're using
# by looking into the .buildkite/autogenerate_pipeline.py file.
LATEST=16
docker run -it \
           --security-opt seccomp=unconfined \
           --volume $(pwd):/${CRATE} \
           --volume ~/.ssh:/root/.ssh \
           rustvmm/dev:v${LATEST}
cd vm-superio
./rust-vmm-ci/test_run.py
```

Known issues:
- When running the `cargo-audit` test, the following error may occur:
```
test_cargo-audit (__main__.TestsContainer) ... error: couldn’t fetch advisory database: git operation failed: reference ‘refs/heads/main’ not found; class=Reference (4); code=NotFound (-3)
```
  A fix for this is to remove `~/.cargo/advisory-db` in the container, and then rerun `test_run.py`:
```
rm -rf ~/.cargo/advisory-db
./rust-vmm-ci/test_run.py
```
