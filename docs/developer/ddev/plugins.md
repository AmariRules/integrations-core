# Plugins

-----

## tox

Our [tox plugin](https://github.com/DataDog/integrations-core/blob/master/datadog_checks_dev/datadog_checks/dev/plugin/tox.py)
dynamically adds environments based on the presence of options defined in the `[testenv]` section of each integration's
`tox.ini` file.

### Style

Setting `dd_check_style` to `true` will enable 2 environments for enforcing our [style conventions](../guidelines/style.md):

1. `style` - This will check the formatting and will error if any issues are found. You may use the `-s/--style` flag
   of `ddev test` to execute only this environment.
2. `format_style` - This will format the code for you, resolving the most common issues caught by `style` environment.
   You can run the formatter by using the `-fs/--format-style` flag of `ddev test`.

## pytest

Our [pytest plugin](https://github.com/DataDog/integrations-core/blob/master/datadog_checks_dev/datadog_checks/dev/plugin/pytest.py)
makes a few [fixtures](https://docs.pytest.org/en/latest/fixture.html) available globally for use during tests. Also, it's responsible
for managing the control flow of E2E environments.

### Fixtures

#### Agent stubs

The stubs provided by each fixture will automatically have their state reset before each test.

- [aggregator](../base/api.md#aggregator)
- [datadog_agent](../base/api.md#datadog-agent)

#### Check execution

Most tests will execute checks via the `run` method of the [AgentCheck interface](../base/api.md#agentcheck)
(if the check [is stateful](../guidelines/conventions.md#stateful-checks)).

A consequence of this is that, unlike the `check` method, exceptions are not propagated to the caller meaning not only can an exception
not be asserted, but also errors are silently ignored.

The `dd_run_check` fixture takes a check instance and executes it while also propagating any exceptions like normal.

```python
def test_metrics(aggregator, dd_run_check):
    check = AwesomeCheck('awesome', {}, [{'port': 8080}])
    dd_run_check(check)
    ...
```

You can use the `extract_message` option to condense any exception message to just the original message rather than the full traceback.

```python
def test_config(dd_run_check):
    check = AwesomeCheck('awesome', {}, [{'port': 'foo'}])

    with pytest.raises(Exception, match='^Option `port` must be an integer$'):
        dd_run_check(check, extract_message=True)
```

#### E2E

##### Agent check runner

The `dd_agent_check` fixture will run the integration with a given configuration on a live Agent and return a populated
[aggregator](../base/api.md#aggregator). It accepts a single `dict` configuration representing either:

- a single instance
- a full configuration with top level keys `instances`, `init_config`, etc.

Internally, this is a wrapper around `ddev env check` and you can pass through any supported options or flags.

This fixture can only be used from tests [marked](http://doc.pytest.org/en/latest/example/markers.html) as `e2e`. For example:

```python
@pytest.mark.e2e
def test_e2e_metrics(dd_agent_check, instance):
    aggregator = dd_agent_check(instance, rate=True)
    ...
```

##### State

Occasionally, you will need to persist some data only known at the time of environment creation (like a generated token)
through the test and environment tear down phases.

To do so, use the following fixtures:

- `dd_save_state` - When executing the necessary steps to spin up an environment you may use this to save any
  object that can be serialized to JSON. For example:

    ```python
    dd_save_state('my_data', {'foo': 'bar'})
    ```

- `dd_get_state` - This may be used to retrieve the data:

    ```python
    my_data = dd_get_state('my_data', default={})
    ```

### Environment manager
