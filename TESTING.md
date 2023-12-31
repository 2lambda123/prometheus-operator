
When contributing code to Prometheus-Operator, you'll notice that every Pull Request will run against an extensive test suite. Among an extensive list of benefits that tests brings to the Project's overall health and reliability, it can be the reviewer's and contributors's best friend during development:

* Test cases serve as documentation, providing insights into the expected behavior of the software.
* Testing can prevent regressions by verifying that new changes don't break existing functionality.
* Running tests locally accelerate the feedback loop, removing the dependency that contributors might have on CI when working on a Pull Request.

This document will focus on teaching you about the different test suites that we currently have and how to run different scenarios to help your development experience!

# Test categories

## Unit tests

Unit tests are used to test particular code snippets in isolation. They are your best ally when looking for quick feedback loops in a particular function. 

Imagine you're working on a PR that adds a new field to the Prometheus CRD and you want to test if your change is reflected to the statefulset. Instead of creating a full Kubernetes cluster, installing all the CRDs, running the Prometheus-Operator, deploying a Prometheus resource and finally check if your change made it to the live object, you could simply write or extend a unit test for the statefulset generation.

Here is an example test that checks if labels are present in the statefulset.

https://github.com/prometheus-operator/prometheus-operator/blob/aeceb0b4fadc8307a44dc55afdceca0bea50bbb0/pkg/prometheus/server/statefulset_test.go#L140-L167

Unit tests can be run with:

```
make test-unit
```

They can also be run for particular packages:

```
go test ./pkg/prometheus/server
```

Or even particular functions:

```
go test -run ^TestPodLabelsAnnotations$ ./pkg/prometheus/server
```

### Testing multiline string comparison - Golden files

Golden files are plaintext documents designed to facilitate the validation of lengthy strings. They come in handy when, for instance, you need to test a Prometheus configuration that's generated using Go structures. You can marshal this configuration into YAML and then compare it against a static reference to ensure a match. Golden files offer an elegant solution to this challenge, sparing you the need to hard-code the static configuration directly into your test code.

In the example below, we're generating the Prometheus configuration (which can easily have 100+ lines for each individual test) and comparing it against a golden file:

https://github.com/prometheus-operator/prometheus-operator/blob/aeceb0b4fadc8307a44dc55afdceca0bea50bbb0/pkg/prometheus/promcfg_test.go#L102-L277

If not for golden files, the test above, instead of ~150 lines, would easily require around ~1000 lines. The usage of golden files help us maintain test suites with several multiline strings comparison without sacrifing test readability.

### Updating Golden Files

There are contributions, e.g. adding a new required field to an existing configuration, that requires us to update several golden files at once. This can easily be done with the command below:

```
make test-unit-update-golden
```

## End-to-end tests

Sometimes, running tests in isolation is not enough and we really want test the behavior of Prometheus-Operator when running in a working Kubernetes cluster. For those occasions, end-to-end tests are our choice.

To run e2e-tests, first start a Kubernetes cluter. We recommend using [MiniKube](https://minikube.sigs.k8s.io/docs/start/) or [KinD](https://kind.sigs.k8s.io/) for development purposes since they are light weight distributions of kubernetes and can run on small notebooks.

For manual e2e tests, you can use the utility script [scripts/run-external.sh](./scripts/run-external.sh), it will make sure all the requirements are in place to run Prometheus-Operator without problems, but for automated end-to-end tests, we have the command:

```
make test-e2e
```

`make test-e2e` will run the complete end-to-end test suite. Those are the same tests we run in Pull Requests pipelines and it will make sure all features requirements amongst ***all*** controllers are working.

When working on a contribution though, it's rare that you'll need to make a change that impacts all controllers at once. Running the complete test suite takes a ***long time***, so you might want to run only the tests that are relevant to your change while developing it.

### Skipping test suites

https://github.com/prometheus-operator/prometheus-operator/blob/272df8a2411bcf877107b3251e79ae8aa8c24761/test/e2e/main_test.go#L46-L50

As shown above, particular test suites can be skipped with Environment Variables. You can also look at our [CI pipeline as example](https://github.com/prometheus-operator/prometheus-operator/blob/272df8a2411bcf877107b3251e79ae8aa8c24761/.github/workflows/e2e.yaml#L85-L94). Altough we always run all tests in CI, skipping irrelevant tests are great during development as they shorten the feedback loop.

Currently, the environment variables below can be used to skip particular end-to-end tests:

```
EXCLUDE_ALERTMANAGER_TESTS
EXCLUDE_PROMETHEUS_TESTS
EXCLUDE_PROMETHEUS_ALL_NS_TESTS
EXCLUDE_THANOSRULER_TESTS
EXCLUDE_OPERATOR_UPGRADE_TESTS
EXCLUDE_PROMETHEUS_UPGRADE_TESTS
```

Tests are skipped as long as their corresponding environment variable is not an empty string.

### Running just a particular end-to-end test

A few test suites can easily take more than an hour even when running in powerful notebooks. If you're debugging a particular test, it might be advantageous for you to comment code just to accelerate your tests.

```patch
// TestDenylist tests the Prometheus Operator configured not to watch specific namespaces.
func TestDenylist(t *testing.T) {
	skipPrometheusTests(t)
	testFuncs := map[string]func(t *testing.T){
+		// "Prometheus":     testDenyPrometheus,
+		// "ServiceMonitor": testDenyServiceMonitor,
-		"Prometheus":     testDenyPrometheus,
-		"ServiceMonitor": testDenyServiceMonitor,
		"ThanosRuler":    testDenyThanosRuler,
	}

	for name, f := range testFuncs {
		t.Run(name, f)
	}
}
```

In the example above we're commenting 2 tests, in combination with Environment Variables to skip other test suites, to make sure we focus on what really matters to us at the moment. Just don't forget to remove the comments once you're done!!