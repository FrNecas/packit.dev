---
title: "Testing Farm"
draft: false
disableToc: false
weight: 4
aliases: [/testing-farm]
---

# Testing Farm

Testing Farm is Packit's testing system.
It is a containerized service for running tests.
Test execution is configured via [Flexible Metadata Format][fmf],
according to the [Metadata Specification][spec].

## Enable Testing

In order to enable test execution simply include `tests` jobs in the `.packit.yaml` configuration:

```yaml
jobs:
- job: tests
  trigger: pull_request
  metadata:
    targets:
    - fedora-all
```
The test job by default requires Copr build to be built before running tests and then
it is installed into the testing environment.
To run tests that don't require Copr build to be built, test job needs to include `skip_build` (described below) option in the metadata:
```yaml
  jobs:
  - job: tests
    trigger: pull_request
    metadata:
      targets:
      - fedora-all
      skip_build: true
```

Required metadata:
* **targets** - Specify which "builds" you want to test.
[As with copr_build job](/docs/configuration#available-copr-build-targets) you can use
specific targets such as `fedora-34-x86_64`. Or just the distro part,
like `centos-stream-8`, in which case the architecture is `x86_64`. Or an alias,
like `fedora-development`, `fedora-stable`, `fedora-all` or `epel-all`.
Each target is then mapped to a [(tmt) distro](https://tmt.readthedocs.io/en/latest/spec/context.html#dimension)
and to a [Testing farm's compose](https://api.dev.testing-farm.io/v0.1/composes)
when submitting a test. You can override the default (target to distro) mapping by
specifying `targets` as a dictionary instead of as a list.
In the following example, the `epel-8-x86_64` build will be tested on `centos-8`
distro (otherwise the default would be `centos-stream-8`) and for
`epel-7-x86_64` build the default mapping (to `centos-7` distro) will be used:
```yaml
  metadata:
    targets:
      epel-8-x86_64:
        distros: [centos-8]
      epel-7-x86_64: {}
```

Optional metadata:
* **fmf_url** - Git repository containing the metadata (FMF) tree.
  Use any format acceptable by the git clone command.
* **fmf_ref** - Branch, tag or commit specifying the desired git revision.
  Defaults to "master" when **fmf_url** is specified and **fmf_ref** is not.
* **skip_build** - Whether to skip the build phase and only run tests (defaults to false). 
  Enabling this will cause no Copr build to be built and installed into the testing environment, 
  only submitting request to Testing Farm (the selected components to be installed should be part of the TMT definitions).

## Restart Testing

The testing will automatically start after an update to the pull request
(and successful Copr build if `skip_build` is false).
To trigger retesting manually (can come handy in case of infrastructure
issues for example), you can use the following comment in the pull request:

    /packit test

## Creating Tests

The easiest way to get started is to use the [tmt][tmt] tool
which will help you with the setup.
Please follow [Fedora tmt page][fedora-tmt].

## More Examples

Get inspiration for a quick start from a couple of real-life examples!

### Using Filters

Use a custom `filter` in the discover step in order to choose relevant tests only:

```yaml
    discover:
        how: fmf
        filter: "tier: 1"
        repository: https://src.fedoraproject.org/tests/selinux
```

### Prepare Step

The `prepare` step can be used to define how test environment should be prepared before testing.
Provide one or more paths to ansible playbooks:

```yaml
    prepare:
        how: ansible
        playbooks:
            - setup/packages.yml
```

### Apache Test

Here is an example of a simple integration test for the web server `httpd` and `curl` utility:

```yaml
    /apache/smoke:
        execute:
            script:
            - dnf -y install httpd curl
            - systemctl start httpd
            - echo foo > /var/www/html/index.html
            - curl http://localhost/ | grep foo
```

Plan `/apache/smoke` defines only the `execute` step.
Individual shell commands are provided as a list.
Testing will fail if any of the commands returns a non-zero exit status.

### Systemd Tests

Below you can find little bit more interesting example of a `systemd` test configuration:

```yaml
    /systemd/smoke:
        summary:
            Basic set of quick smoke tests for systemd.
        discover:
            how: fmf
            filter: "tier: 1 & distro: rhel-8"
            repository: "https://github.com/systemd-rhel/tests"
        prepare:
            how: ansible
            playbooks: [setup/packages.yml]
        execute:
            how: beakerlib
```

This plan enables a set of Tier 1 tests from the shared [systemd tests][systemd-tests] repository.
The meaning of individual attributes is as follows:

- Summary — an optional but useful attribute describing high-level purpose of the plan.
- Discover — instructs to fetch tests from given repository and select relevant ones by provided `filter`.
- Prepare — specifies which ansible playbook should be applied to prepare environment for testing.
- Execute — defines that the beakerlib framework should be used for running the tests.

### FMF Tests

Here's a real-life example of tests enabled for the [fmf][fmf] package.
There are several plans defined under the [plans](https://github.com/psss/fmf/tree/master/plans) directory.
The `smoke` plan enables a super basic test checking availability of the `fmf` command:

```yaml
    summary:
    	Just a basic smoke test
    execute:
    	how: shell
    	script: fmf --help
```

Plan `features` is used to execute all available beakerlib tests from the `fmf` repository:

```yaml
    summary:
    	Essential command line features
    discover:
    	how: fmf
    	repository: https://github.com/psss/fmf
    execute:
    	how: beakerlib
```

It is also possible to select only a subset of available tests.
This is demonstrated by the `docs` plan.
Use an fmf `filter` like `tier:1` to select tests for execution.
You can also reference a specific feature area instead:

```yaml
    summary:
    	Ensure that documentation is present
    discover:
    	how: fmf
    	repository: https://github.com/psss/fmf
    	filter: coverage:/stories/docs.*
    execute:
    	how: beakerlib
```

See the [stories](https://github.com/psss/fmf/tree/master/stories) directory to get some inspiration for organizing stories and requirements.

### Running linters

Running linters on your code is easy to setup using Testing Farm and tmt.
Linters are tools which you can install from the distribution and they usually
just require a path to files which they check. Here is a plan which you can use
to run `rpmlint` on your spec file:

```yaml
summary:
  Execute rpmlint on the spec file
prepare:
  - name: packages
    how: install
    package:
    - rpmlint
execute:
  script:
  - rpmlint fedora/python-ogr.spec
```

## Testing Farm API

Packit Service communicates with Testing Farm via its [API][testing-farm-api].

## Issues & RFEs

If you have found an issue or have an RFE, you can file an [issue in nucleus project][issues].

[fmf]: https://fmf.readthedocs.io/
[tmt]: https://tmt.readthedocs.io/
[fedora-tmt]: https://docs.fedoraproject.org/en-US/ci/tmt
[spec]: https://tmt.readthedocs.io/en/latest/spec.html
[systemd-tests]: https://github.com/systemd-rhel/tests
[testing-farm-api]: https://testing-farm.gitlab.io/api
[issues]: https://gitlab.com/testing-farm/general/-/issues
