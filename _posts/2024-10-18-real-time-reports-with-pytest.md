---
title: Real-time reports with Pytest
description: >-
  Using custom Pytest hooks to generate real-time reports
date: 2024-10-18 06:40:00 +0100
categories: [Python]
tags: [python,pytest,hooks]
tok: true
-image:
  path: /assets/img/illustrations/realtimestreaming.jpg
  alt: Realtime Streaming
---

---
One remarkable characteristic of Pytest is that it is very easy to extend its functionality with minimal effort.

The way fixtures and hooks are designed to work is really noteworthy from a design perspective.

The framework by default offers the bare minimum, which is actually what's necessary 90% of the times.

But where it really shines is that it's batteries included to perform much more complicated stuff if you want it to.


## Custom report generation
I've used Pytest a few times in scheduled, long running integration tests in CI/CD pipelines and what I've noticed was that generating some sort of progress status was necessary.

As it turned out, this was quite easy to implement with regular fixtures and a custom hook.

The key to the implementation was the standard `pytest_runtest_makereport` hook, which allows you to tap into the metadata of a running test item.

## The implementation

The actual implementation was seamless!

I only had to override the `pytest_runtest_makereport` hook and create a [StashKey](https://docs.pytest.org/en/7.1.x/reference/reference.html#stash).

More information regarding StashKeys can be found [here](https://docs.pytest.org/en/stable/how-to/writing_hook_functions.html#storing-data-on-items-across-hook-functions).

```python
from datetime import datetime
from enum import Enum
from typing import Any, AsyncIterator, Callable, Dict, Generator

import pytest
import pytest_asyncio


PHASE_REPORT_KEY: pytest.StashKey = (
    pytest.StashKey[Dict[str, pytest.CollectReport]]()
)


class Status(Enum):
    PASSED = 'passed'
    FAILED = 'failed'
    IN_PROGRESS = 'in-progress'
    CANCELLED = 'cancelled'
    SKIPPED = 'skipped'


class Report:
    """Pytest plugin class used to stream test reports at runtime"""
    def __init__(self) -> None:
        self.session_timestamps = {}
        self.session_summary = {}
        self.session_results = {}

    @pytest.hookimpl(tryfirst=True, hookwrapper=True)
    def pytest_runtest_makereport(
        self,
        item: Callable,
        call,
    ) -> Generator[None, Any, None]:
        """Pytest hook that wraps the standard pytest_runtest_makereport
        function and grabs the results for the 'call' phase of each test.
        """
        outcome = yield
        report = outcome.get_result()

        test_cls_docstring = item.parent.obj.__doc__ or ''
        test_fn_docstring = item.obj.__doc__ or ''
        report.description = test_fn_docstring or test_cls_docstring

        report.exception = None
        if report.outcome == "failed":
            exception = call.excinfo.value
            report.exception = exception

        item.stash.setdefault(PHASE_REPORT_KEY, {})[report.when] = report
```

Now, I had access to the metadata of each test item and I could stash them in order to be able to use them in my fixtures.

The only thing that was left, was to generate my custom report in 3 phases:
- Once at the beginning of the session
- Once after every test was completed
- Once in the end of the session

For example, given 5 tests, I would generate 7 reports, essentially streaming the progress of the whole session in real time.

Creating the fixtures was simple too.

A `first_last_report` fixture scoped as `session` would generate the first and last reports of the session.

```python
    @pytest_asyncio.fixture(scope="session", autouse=True)
    async def first_last_report(
        self,
        request,
    ) -> AsyncIterator[None]:
        """Pytest fixture that publishes the first and last report"""
        started = datetime.now()

        self.session_timestamps["started"] = started
        self.session_summary["passed"] = 0
        self.session_summary["failed"] = 0
        self.session_summary["status"] = Status.IN_PROGRESS.value

        report = dict(
            timestamps=self.session_timestamps,
            summary=self.session_summary,
            results=self.session_results,
        )
        print(f"\nReport before all tests: {report}")

        # Wait until all the tests are completed
        yield

        total_tests = request.session.testscollected
        failed_tests = request.session.testsfailed
        passed_tests = total_tests - failed_tests

        finished = datetime.now()
        duration = (finished - started).total_seconds()

        self.session_timestamps["finished"] = finished
        self.session_timestamps["duration"] = duration
        self.session_summary["passed"] = passed_tests
        self.session_summary["failed"] = failed_tests
        self.session_summary["status"] = (
            Status.FAILED.value
            if request.session.testsfailed
            else Status.PASSED.value
        )

        report = dict(
            timestamps=self.session_timestamps,
            summary=self.session_summary,
            results=self.session_results,
        )
        print(f"\nReport after all tests: {report}")
```

A `report` fixture scoped as `function` (default) would generate a report after each test was completed.

```python
    @pytest_asyncio.fixture(autouse=True)
    async def report(
        self,
        request,
    ) -> AsyncIterator[None]:
        """Pytest fixture that publishes a report every
        time an individual unit test has been completed.
        """

        # Wait until the test is completed
        yield

        # Gather test results
        report = request.node.stash[PHASE_REPORT_KEY]

        test_module = report["call"].fspath.split(".")[0]
        test_name = report["call"].head_line.replace(".", "::")
        test_description = report["call"].description
        passed = report["call"].passed
        failed = report["call"].failed
        skipped = report["call"].skipped
        exception = repr(report["call"].exception)

        status = None
        if passed:
            status = Status.PASSED.value
            self.session_summary["passed"] += 1
        elif failed:
            status = Status.FAILED.value
            self.session_summary["failed"] += 1
        elif skipped:
            status = Status.SKIPPED.value

        started = self.session_timestamps["started"]
        self.session_timestamps["duration"] = (
            (datetime.now() - started).total_seconds()
        )

        self.session_results.setdefault(test_module, {})[test_name] = dict(
            name=test_name,
            description=test_description,
            status=status,
            error=exception,
        )

        report = dict(
            timestamps=self.session_timestamps,
            summary=self.session_summary,
            results=self.session_results,
        )
        print(f"\nReport after test '{test_name}': {report}")
```

## Extra bits
I also added a new pytest flag `--stream-reports` and conditionally registered the plugin

```python
def pytest_addoption(parser):
    group = parser.getgroup('report-stream')
    group.addoption(
        '--stream-reports',
        dest='stream_reports',
        action='store_true',
        help='Enable reports'
    )


@pytest.hookimpl(tryfirst=True)
def pytest_configure(config):
    if config.option.stream_reports:
        config._stream_reports = Report()
        config.pluginmanager.register(config._stream_reports)
```

## The result
Let's assume we have the following tests
```python
import pytest


@pytest.mark.asyncio
async def test_reports_1():
    assert False

@pytest.mark.asyncio
async def test_reports_2():
    assert True
```

If we run them without the flag, the result is as expected:
```bash
asdf:~/reporting $ poetry run pytest -s tests/
======================================================== test session starts ========================================================
collected 2 items

tests/test_reports.py F.

============================================================= FAILURES ==============================================================
___________________________________________________________ test_reports_1 ___________________________________________________________

    @pytest.mark.asyncio
    async def test_reports_1():
>       assert False
E       assert False

tests/test_reports.py:6: AssertionError
====================================================== short test summary info ======================================================
FAILED tests/test_reports.py::test_reports_1 - assert False
==================================================== 1 failed, 1 passed in 0.04s ====================================================
```

If we run them with the flag, we should also see the reports being printed in the STDOUT:
```bash
asdf:~/reporting $ poetry run pytest -s --stream-reports tests/
======================================================== test session starts ========================================================
collected 2 items

tests/test_reports.py
Report before all tests: {'timestamps': {'started': datetime.datetime(2024, 10, 18, 14, 0, 43, 938273)}, 'summary': {'passed': 0, 'failed': 0, 'status': 'in-progress'}, 'results': {}}
F
Report after test 'test_reports_1': {'timestamps': {'started': datetime.datetime(2024, 10, 18, 14, 0, 43, 938273), 'duration': 0.018298}, 'summary': {'passed': 0, 'failed': 1, 'status': 'in-progress'}, 'results': {'tests/test_reports': {'test_reports_1': {'name': 'test_reports_1', 'description': '', 'status': 'failed', 'error': "AssertionError('assert False')"}}}}
.
Report after test 'test_reports_2': {'timestamps': {'started': datetime.datetime(2024, 10, 18, 14, 0, 43, 938273), 'duration': 0.020047}, 'summary': {'passed': 1, 'failed': 1, 'status': 'in-progress'}, 'results': {'tests/test_reports': {'test_reports_1': {'name': 'test_reports_1', 'description': '', 'status': 'failed', 'error': "AssertionError('assert False')"}, 'test_reports_2': {'name': 'test_reports_2', 'description': '', 'status': 'passed', 'error': 'None'}}}}

Report after all tests: {'timestamps': {'started': datetime.datetime(2024, 10, 18, 14, 0, 43, 938273), 'duration': 0.020318, 'finished': datetime.datetime(2024, 10, 18, 14, 0, 43, 958591)}, 'summary': {'passed': 1, 'failed': 1, 'status': 'failed'}, 'results': {'tests/test_reports': {'test_reports_1': {'name': 'test_reports_1', 'description': '', 'status': 'failed', 'error': "AssertionError('assert False')"}, 'test_reports_2': {'name': 'test_reports_2', 'description': '', 'status': 'passed', 'error': 'None'}}}}

============================================================= FAILURES ==============================================================
___________________________________________________________ test_reports_1 ___________________________________________________________

    @pytest.mark.asyncio
    async def test_reports_1():
>       assert False
E       assert False

tests/test_reports.py:6: AssertionError
====================================================== short test summary info ======================================================
FAILED tests/test_reports.py::test_reports_1 - assert False
==================================================== 1 failed, 1 passed in 0.04s ====================================================
```

We can clearly see the following 4 reports printed.

The first one when the session starts:
```python
{
    'timestamps': {
        'started': datetime.datetime(2024, 10, 18, 14, 0, 43, 938273),
    },
    'summary': {
        'passed': 0,
        'failed': 0,
        'status': 'in-progress'
    },
    'results': {}
}
```

The second when the `test_reports_1` test completes:
```python
{
    'timestamps': {
        'started': datetime.datetime(2024, 10, 18, 14, 0, 43, 938273),
        'duration': 0.018298
    },
    'summary': {
        'passed': 0,
        'failed': 1,
        'status': 'in-progress'
    },
    'results': {
        'tests/test_reports': {
            'test_reports_1': {
                'name': 'test_reports_1',
                'description': '',
                'status': 'failed',
                'error': "AssertionError('assert False')"
            }
        }
    }
}
```

The third when the `test_reports_2` completes:
```python
{
    'timestamps': {
        'started': datetime.datetime(2024, 10, 18, 14, 0, 43, 938273),
        'duration': 0.020047
    },
    'summary': {
        'passed': 1,
        'failed': 1,
        'status': 'in-progress'
    },
    'results': {
        'tests/test_reports': {
            'test_reports_1': {
                'name': 'test_reports_1',
                'description': '',
                'status': 'failed',
                'error': "AssertionError('assert False')"
            },
            'test_reports_2': {
                'name': 'test_reports_2',
                'description': '',
                'status': 'passed',
                'error': 'None'
            }
        }
    }
}
```

Finally, the last when the session ends:
```python
{
    'timestamps': {
        'started': datetime.datetime(2024, 10, 18, 14, 0, 43, 938273),
        'duration': 0.020318,
        'finished': datetime.datetime(2024, 10, 18, 14, 0, 43, 958591),
    },
    'summary': {
        'passed': 1,
        'failed': 1,
        'status': 'failed'
    },
    'results': {
        'tests/test_reports': {
            'test_reports_1': {
                'name': 'test_reports_1',
                'description': '',
                'status': 'failed',
                'error': "AssertionError('assert False')"
            },
            'test_reports_2': {
                'name': 'test_reports_2',
                'description': '',
                'status': 'passed',
                'error': 'None'
            }
        }
    }
}
```

Each test progressively adds a few more details in our report, which can be streamlined, leading to the final state of our session.

## Conclusion
Pytest is awesome! Period..

You can easily dump these snippets in your conftest.py and even ship it as a plugin, which is what I've basically done in [pytest-report-stream](https://pypi.org/project/pytest-report-stream/).

The source code is available in my github repo [pytest-report-stream](https://github.com/kolitiri/pytest-report-stream).
