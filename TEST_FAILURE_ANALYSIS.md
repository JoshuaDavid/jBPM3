# Test Failure Root Cause Analysis

## Summary

103 of 1118 tests (10 failures + 93 errors) fail when running the full test suite,
but all pass when run individually. The root cause is a **test-ordering contamination
bug** in `JbpmConfigurationTest.testSingleton()`.

## Root Cause

`JbpmConfigurationTest.testSingleton()` (line 44 of `JbpmConfigurationTest.java`)
poisons the static `JbpmConfiguration.instances` cache, and the test's `tearDown()`
method fails to clean it up.

### The Poisoning Sequence

`JbpmConfiguration` maintains a static singleton cache:

```java
// JbpmConfiguration.java line ~150
private static final Map instances = new HashMap(); // key: resource name, value: JbpmConfiguration
```

`JbpmConfigurationTest` has this lifecycle:

```java
protected void setUp() {
    JbpmConfiguration.clearInstances();       // clears the cache
}

protected void tearDown() {
    JbpmConfiguration.setDefaultObjectFactory(null); // clears defaultObjectFactory
    // BUG: does NOT call clearInstances()
}
```

`testSingleton()` is the **last** method to execute within the class (confirmed via the
Surefire XML report). It does:

```java
public void testSingleton() {
    JbpmConfiguration.setDefaultObjectFactory(new ObjectFactoryImpl()); // EMPTY factory
    JbpmConfiguration instance = JbpmConfiguration.getInstance();       // caches it
    // ...assertions...
}
```

Step by step:

1. `setUp()` clears the cache
2. `setDefaultObjectFactory(new ObjectFactoryImpl())` — sets an **empty** ObjectFactory
3. `getInstance()` → cache miss → `defaultObjectFactory != null` → creates
   `new JbpmConfiguration(emptyFactory)` → caches under key `"jbpm.cfg.xml"`
4. `tearDown()` → `setDefaultObjectFactory(null)` — but **does NOT clear the cache**

After `JbpmConfigurationTest` completes, the cache contains:

```
"jbpm.cfg.xml" → JbpmConfiguration(ObjectFactoryImpl{ namedObjectInfos: {} })
```

### How Subsequent Tests Fail

Any test that calls `JbpmConfiguration.getInstance()` (directly or via
`Configs.getString()`) gets the cached poisoned instance with an empty ObjectFactory.

**For `AbstractDbTestCase` tests** (e.g., `TaskTimerExecutionDbTest`):
```
setUp() → getJbpmConfiguration() → getInstance("jbpm.cfg.xml")
       → cache hit: poisoned config
       → createJbpmContext() → looks up "default.jbpm.context" → NOT FOUND
       → ConfigurationException: no info for object 'default.jbpm.context'; defined objects: []
```

**For `AbstractJbpmTestCase` tests** (e.g., `VariableTypeTest`, `XmlSchemaTest`):
```
ProcessDefinition.parseXmlString(...)
  → createNewProcessDefinition()
    → Configs.getString("resource.default.modules")
      → Configs.getObjectFactory()    [no JbpmContext on thread]
        → JbpmConfiguration.getInstance()
          → cache hit: poisoned config
          → "resource.default.modules" NOT FOUND in empty factory
          → ConfigurationException: no info for object 'resource.default.modules'; defined objects: []
```

### Why the Failures Stop

`JbpmContextTest` (test #50 in execution order) has:
```java
protected void setUp() {
    JbpmConfiguration.clearInstances(); // CLEANS the poisoned cache
}
```

After `JbpmContextTest` runs, the cache is empty. Subsequent tests that call
`getInstance()` load the real `jbpm.cfg.xml` from the classpath, which triggers
`parseObjectFactory()` to read `org/jbpm/default.jbpm.cfg.xml` with all the proper
configuration entries.

### Additional Detail: `close()` Can't Fix It Either

The poisoned `JbpmConfiguration` instance was created via the public constructor
`new JbpmConfiguration(objectFactory)` which sets `resourceName = null`. The `close()`
method (line 572) only removes from cache when `resourceName != null`:

```java
if (resourceName != null) {
    instances.remove(resourceName);
}
```

So even calling `close()` on the poisoned instance won't clear the cache entry (it was
cached under `"jbpm.cfg.xml"` but the instance's `resourceName` is `null`).

## Actual Test Execution Order (Verified)

| # | Test Class | Result |
|---|-----------|--------|
| 1-24 | Various (JBPM2959Test, EventPropagationTest, ..., ExceptionHandlerDbTest) | All PASS |
| **25** | **JbpmConfigurationTest** | **PASS (but poisons cache)** |
| 26 | VariableTypeTest | 19 errors |
| 27 | ActionValidatingXmlTest | 16 errors |
| 28 | NodeActionTest | 4 failures |
| 29 | TaskTimerExecutionDbTest | 6 errors |
| 30 | AsyncTimerAndSubProcessDbTest | 1 error |
| 31 | JBPM3423Test | 1 error |
| 32 | TimerXmlTest | 9 errors |
| 33 | XmlSchemaTest | 16 errors |
| 34 | SignalLogTest | 1 error |
| 35 | InitialNodeTest | 3 errors |
| 36 | StateDbTest | 1 error |
| 37 | ObjectFactoryUserGuideTest | PASS (doesn't use JbpmConfiguration) |
| 38 | ProcessStateTest | 2 errors |
| 39 | JbpmContextGetDbTest | 6 errors |
| 40 | SignalLogDbTest | 1 error |
| 41 | JBPM2787Test | 1 error |
| 42 | JBPM2828Test | PASS (overrides getJbpmConfiguration() with custom config) |
| 43 | Wfp18InterleavedParallelRoutingTest | 6 failures |
| 44 | JBPM3235Test | PASS (uses custom config resource) |
| 45 | SOA2010Test | 4 errors |
| 46 | JBPM2608Test | 1 error |
| 47 | JBPM1778Test | 2 errors |
| 48 | JBPM2375Test | 2 errors |
| 49 | ProcessClassLoaderTest | 1 error |
| **50** | **JbpmContextTest** | **PASS (clears poisoned cache in setUp)** |
| 51-269 | Remaining 219 tests | All PASS |

**Total failing tests: 103** (10 failures + 93 errors across 20 test classes)

## The Fix

Add `clearInstances()` to `JbpmConfigurationTest.tearDown()`:

```java
protected void tearDown() throws Exception {
    JbpmConfiguration.setDefaultObjectFactory(null);
    JbpmConfiguration.clearInstances();  // ADD THIS LINE
    super.tearDown();
}
```

This is a one-line fix in the test infrastructure, not a product code change.
