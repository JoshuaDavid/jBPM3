# Developer Decision Inventory Plan

## Purpose

Inventory every place in the jBPM3 codebase where an intentional developer
decision was made and an alternative could have been chosen. This inventory
serves as the basis for building an extremely comprehensive test suite.

## Codebase Summary

| Category | Files | LOC |
|----------|------:|--------:|
| Core module main source | 407 | 52,731 |
| Core module test source | 299 | 38,018 |
| Non-core module source | ~269 | ~15,000 |
| Hibernate mapping files | 105 | XML |
| XML configuration files | 60+ | XML |
| **Total** | **~1,140** | **~106,000** |

## Decision Categories to Identify Per Chunk

Every chunk should catalog decisions in these categories:

1. **Control flow** — if/else branches, switch/case, ternary operators, early returns.
   Each branch represents "the developer decided X should happen when condition Y
   is true; alternatively Z could happen." (~2,925 total across core)
2. **Null handling** — == null / != null checks, what happens on null vs non-null.
   (~1,244 null comparisons across core)
3. **Type discrimination** — instanceof checks, discriminator values, class
   hierarchy choices. (~165 instanceof + 296 type hierarchy decisions)
4. **Exception strategy** — which exceptions are caught vs propagated, what
   recovery action is taken, custom exception types thrown. (~989 try/catch/throws)
5. **Collection type** — HashMap vs TreeMap, ArrayList vs LinkedList, Set vs List,
   ordering guarantees. (~271 collection instantiations)
6. **Default values** — hardcoded constants, magic numbers, default parameters,
   initial field values. (~865 static final declarations)
7. **Visibility** — public vs protected vs package-private vs private; what's
   exposed as API vs hidden as implementation. (~1,237 method visibility decisions)
8. **Synchronization** — synchronized blocks, volatile fields, thread-safety
   assumptions. (~61 sync points)
9. **ORM mapping** — inheritance strategy, fetch mode, cascade type, collection
   type, key strategy, optimistic locking. (105 mapping files)
10. **Configuration defaults** — service implementations, timeouts, retry counts,
    thread pool sizes, polling intervals. (60+ config files)
11. **Algorithm/logic** — specific formulas, iteration strategies, ordering
    choices, string handling approaches.
12. **API contract** — method signatures, return types, parameter types,
    checked vs unchecked exceptions.

---

## The 20 Chunks

### Chunk 1: Graph Definition Model

**Scope:** The static process graph structure — how processes are defined before
execution.

| Item | Value |
|------|-------|
| Packages | `graph.def`, `module.def`, `module.exe` |
| Files | 15 |
| LOC | ~3,182 |
| Key classes | `ProcessDefinition`, `Node`, `Transition`, `Event`, `Action`, `SuperState`, `ExceptionHandler`, `GraphElement`, `NodeCollection` |

**Decision types to inventory:**
- Graph structure: How nodes, transitions, and events relate (parent-child,
  collections, ordering). Why a list for leaving transitions but a set for
  arriving transitions?
- Event model: Which lifecycle points have events (node-enter, node-leave,
  transition, before-signal, after-signal, etc.). Why these and not others?
- ExceptionHandler chain: How exceptions propagate up the graph element
  hierarchy. Why check parent before process definition?
- SuperState nesting: How nested states resolve transitions and events.
- Action attachment: Actions bound to events vs directly to nodes.
- GraphElement as abstract base: What's in the base class vs subclasses.

**Estimated decision points:** ~400

---

### Chunk 2: Graph Node Types and Actions

**Scope:** Every concrete node type — the behavioral building blocks of
processes.

| Item | Value |
|------|-------|
| Packages | `graph.node`, `graph.node.advanced`, `graph.action` |
| Files | 25 |
| LOC | ~2,726 |
| Key classes | `Decision`, `Fork`, `Join`, `Merge`, `State`, `EndState`, `StartState`, `TaskNode`, `ProcessState`, `MailNode`, `InterleaveStart`/`End`, `Script`, `MailAction` |

**Decision types to inventory:**
- Per-node execution semantics: What happens when a token arrives at each node
  type? When does it auto-leave vs wait?
- Fork: How child tokens are created. Naming strategy. What happens with
  script-based forking?
- Join: Token merging logic. When is a join "complete"? Discriminator join
  (n-of-m) vs all-must-arrive. Reactivation of parent token.
- Decision: Expression evaluation vs delegation handler vs condition-based
  transitions. Priority ordering of conditions.
- TaskNode: Blocking behavior (signal=first, signal=last, signal=never,
  signal=unsynchronized). How task completion maps to node leaving.
- EndState: Process termination vs token termination. Whether
  `isTerminationImplicit`.
- ProcessState: Subprocess binding. Variable mapping in/out.
- MailNode: Template resolution, recipient resolution, BCC/CC handling.

**Estimated decision points:** ~350

---

### Chunk 3: Process Execution and Token System

**Scope:** Runtime execution — process instances, tokens, execution context,
and execution logging.

| Item | Value |
|------|-------|
| Packages | `graph.exe`, `graph.log`, `signal` |
| Files | 15 |
| LOC | ~2,400 |
| Key classes | `ProcessInstance`, `Token`, `ExecutionContext`, `RuntimeAction`, `NodeLog`, `TransitionLog`, `SignalLog`, `TokenCreateLog`, `ProcessInstanceCreateLog` |

**Decision types to inventory:**
- Token tree: Parent-child relationships, token naming, locking semantics.
- Signal propagation: How `Token.signal()` triggers node execution. Default
  transition vs named transition.
- Execution context: What state is available during execution (token, node,
  transition, process instance, task instance).
- Process lifecycle: Start → active → suspended → ended state machine.
- Token lifecycle: Active → suspended → ended. What "ended" means for child
  tokens.
- RuntimeAction: How runtime-added actions interact with definition-time
  actions.
- Logging granularity: What gets a log entry (node enter, transition take,
  signal, token create/end, process create/end). What data is captured in
  each log type.

**Estimated decision points:** ~300

---

### Chunk 4: JPDL XML Parsing and Process Archives

**Scope:** How process definitions are loaded from XML and PAR archives.

| Item | Value |
|------|-------|
| Packages | `jpdl.xml`, `jpdl.par`, `jpdl`, `jpdl.convert`, `jpdl.exe` |
| Files | 15 |
| LOC | ~2,207 |
| Key classes | `JpdlXmlReader`, `JpdlXmlWriter`, `ProcessArchive`, `ProcessArchiveParser`, `FileArchiveParser`, `ConfigurableParser`, `JpdlException` |

**Decision types to inventory:**
- XML element handling: For each element (process-definition, node, transition,
  action, event, timer, task, swimlane, variable, etc.), what attributes are
  parsed, what defaults are applied, what's optional vs required.
- Error recovery: What happens on malformed XML? Missing attributes?
  Unknown elements?
- Schema versioning: How JPDL 3.0 vs 3.1 vs 3.2 differences are handled.
- Archive structure: How PAR/ZIP entries are resolved. Class loading from
  archives. The `classes/` directory convention.
- Parser extension points: The configurable parser registry (`jbpm.parsers.xml`).
  How custom parsers are discovered and invoked.
- Write format: What gets serialized back to XML and what's lost in
  round-tripping.

**Estimated decision points:** ~350

---

### Chunk 5: Expression Language — Grammar and Parser

**Scope:** The JPDL expression language parser and core interfaces.

| Item | Value |
|------|-------|
| Packages | `jpdl.el`, `jpdl.el.parser` |
| Files | 13 |
| LOC | ~4,260 |
| Key classes | `ExpressionEvaluator`, `Expression`, `VariableResolver`, `FunctionMapper`, `ELException`, `ELParseException`, `ELParser` (generated) |

**Decision types to inventory:**
- Grammar rules: What expressions are valid? Operator precedence. Supported
  literals (string, integer, floating point, boolean, null).
- Parser error handling: What constitutes a parse error vs a runtime error.
- Token types: What lexical tokens are recognized.
- API contracts: The separation of parsing (compile-time) from evaluation
  (runtime). Why `ExpressionEvaluator` and `Expression` are separate.
- Variable resolution: How `${variable}` references are resolved. Scoping rules.
- Function mapping: How `${fn:name()}` functions are resolved.

**Note:** Much of `jpdl.el.parser` may be JavaCC-generated code. Focus on the
grammar definition decisions (what the language supports) rather than the
generated parser mechanics.

**Estimated decision points:** ~200

---

### Chunk 6: Expression Language — Evaluator and Type System

**Scope:** Runtime expression evaluation — operators, type coercion, literals.

| Item | Value |
|------|-------|
| Packages | `jpdl.el.impl` |
| Files | 51 |
| LOC | ~9,822 |
| Key classes | `ExpressionEvaluatorImpl`, `Coercions`, `BooleanLiteral`, `IntegerLiteral`, `FloatingPointLiteral`, `StringLiteral`, `ArithmeticOperator`, `RelationalOperator`, `AndOperator`, `OrOperator`, `NotOperator`, etc. |

**Decision types to inventory:**
- Type coercion rules: How does `String + Integer` evaluate? What about
  `null + 5`? Each coercion rule is a decision (51 files × multiple coercion
  paths). The `Coercions` class is particularly dense with decisions.
- Operator semantics: For each operator (+, -, *, /, %, ==, !=, <, >, <=, >=,
  &&, ||, !, empty, not empty), what types are supported and how are mixed
  types handled?
- Null propagation: How does null interact with each operator? Is `null == null`
  true? Is `null + 1` an error or null?
- String-to-number conversion: When and how strings are auto-converted to
  numbers.
- Boolean semantics: What values are truthy vs falsy beyond true/false.
- Property access: How `object.property` is resolved (reflection, Map access,
  List index, etc.).

**Note:** This is the largest chunk by LOC but each class is small (~193 LOC
avg) and represents a focused decision surface. The decision density per class
is high — each operator/literal/coercion class embodies specific behavioral
choices.

**Estimated decision points:** ~600

---

### Chunk 7: Task Management

**Scope:** Human task lifecycle — definition, assignment, execution, and
logging.

| Item | Value |
|------|-------|
| Packages | `taskmgmt.def`, `taskmgmt.exe`, `taskmgmt.impl`, `taskmgmt.log` |
| Files | 20 |
| LOC | ~2,730 |
| Key classes | `Task`, `TaskInstance`, `TaskMgmtDefinition`, `TaskMgmtInstance`, `Swimlane`, `SwimlaneInstance`, `PooledActor`, `TaskController`, `Assignable`, `AssignmentHandler` |

**Decision types to inventory:**
- Task lifecycle: Create → assign → start → end. What transitions are valid?
  Can a task be re-assigned after starting? Can it be cancelled?
- Assignment strategy: Direct actor assignment vs swimlane-based vs pooled
  actors. Priority between these when multiple apply.
- Swimlane binding: When does a swimlane instance get its actor? First task
  assignment propagates to swimlane, or swimlane dictates to tasks?
- Blocking/signaling: The 4 signal modes (first, last, never, unsynchronized)
  and how they interact with multiple tasks on a single TaskNode.
- Task controller: Variable mapping between process context and task form.
  Default controller behavior vs custom `TaskControllerHandler`.
- Priority levels: HIGHEST, HIGH, NORMAL, LOW, LOWEST — how they're used.
- Pooled actors: How actor selection works from a pool. Locking semantics.
- Logging: What task events get logged (create, assign, start, end) and
  what data is captured (old actor, new actor, etc.).

**Estimated decision points:** ~350

---

### Chunk 8: Variable and Context System

**Scope:** Process variables — storage, type handling, scoping, and conversion.

| Item | Value |
|------|-------|
| Packages | `context.def`, `context.exe`, `context.exe.variableinstance`, `context.exe.converter`, `context.exe.matcher`, `context.log`, `context.log.variableinstance` |
| Files | 49 |
| LOC | ~3,194 |
| Key classes | `ContextDefinition`, `ContextInstance`, `VariableInstance` (abstract), `StringInstance`, `LongInstance`, `DoubleInstance`, `DateInstance`, `ByteArrayInstance`, `HibernateLongInstance`, `HibernateStringInstance`, `NullInstance`, `TokenVariableMap` |

**Decision types to inventory:**
- Variable type resolution: How the system decides which `VariableInstance`
  subtype to use for a given Java object. The matcher chain
  (`jbpm.varmapping.xml`) and its ordering.
- Type conversion: Each converter (BooleanToStringConverter,
  DateToLongConverter, etc.) is a decision about how to persist a Java type
  in a database column.
- Scope hierarchy: Variables at process level vs token level. How
  `TokenVariableMap` creates per-token variable namespaces.
- Null handling: `NullInstance` as an explicit type vs just not persisting nulls.
- Hibernate types: `HibernateLongInstance` and `HibernateStringInstance` — the
  decision to support arbitrary Hibernate entities as process variables using
  `any` mappings.
- Variable lifecycle logging: What changes get logged, what data is captured
  in create/update/delete logs.

**Estimated decision points:** ~350

---

### Chunk 9: Database and Persistence Layer

**Scope:** Hibernate integration, session management, queries, and transaction
handling.

| Item | Value |
|------|-------|
| Packages | `db`, `db.hibernate`, `db.compatibility`, `persistence`, `persistence.db`, `persistence.jta` |
| Files | 28 |
| LOC | ~5,156 |
| Key classes | `GraphSession`, `TaskMgmtSession`, `ContextSession`, `JobSession`, `LoggingSession`, `JbpmSchema`, `DbPersistenceService`, `DbPersistenceServiceFactory`, `StaleObjectLogConfigurer`, `JtaDbPersistenceService` |

**Decision types to inventory:**
- Session architecture: Why separate sessions per domain (GraphSession,
  TaskMgmtSession, etc.) rather than a single DAO? What queries are in which
  session?
- HQL query design: Each query's filter criteria, ordering, parameter binding,
  pagination. Why certain queries use HQL vs Criteria API.
- Stale object handling: `StaleObjectLogConfigurer` — when to log vs throw on
  optimistic lock failures.
- Schema management: `JbpmSchema` — DDL generation, schema export/update
  decisions.
- Connection management: Single session vs multiple sessions. Session flushing
  strategy (auto vs manual).
- JTA integration: When to use JTA transactions vs local JDBC transactions.
  `JtaDbPersistenceService` vs `DbPersistenceService`.
- Hibernate dialect handling: MySQL custom dialect, compatibility layer for
  different Hibernate versions.
- Save operations: The `svc.save` package — what gets saved, in what order,
  what cascade operations are relied upon vs explicit saves.

**Estimated decision points:** ~500

---

### Chunk 10: Configuration and Object Factory

**Scope:** The lightweight IoC container and jBPM bootstrap configuration.

| Item | Value |
|------|-------|
| Packages | `configuration`, root `org.jbpm` (JbpmConfiguration, JbpmContext) |
| Files | 29 |
| LOC | ~3,437 |
| Key classes | `JbpmConfiguration`, `JbpmContext`, `ObjectFactory`, `ObjectFactoryImpl`, `ObjectFactoryParser`, `BeanInfo`, `ConstructorInfo`, `FieldInfo`, `PropertyInfo`, `JbpmContextInfo`, `JbpmTypeObjectInfo` |

**Decision types to inventory:**
- IoC design: Why build a custom IoC container instead of using Spring? The
  ObjectFactory pattern — XML → ObjectInfo → instance.
- Type system: The `*Info` class hierarchy (BooleanInfo, IntegerInfo, LongInfo,
  DoubleInfo, FloatInfo, CharacterInfo, StringInfo, ListInfo, MapInfo, SetInfo,
  NullInfo, RefInfo, BeanInfo). Why these types and not others?
- Bean lifecycle: Constructor injection vs field injection vs property
  (setter) injection. When each is used.
- Singleton vs prototype: `isSingleton` flag on beans. Which beans are singletons
  by default.
- JbpmConfiguration: Singleton management, thread-local JbpmContext stack,
  close/dispose semantics.
- JbpmContext: Service lookup, convenience methods, the decision about which
  operations are "first-class" (newProcessInstance, getTaskList, etc.).
- Resource loading: How configuration resources are located (classpath,
  file system, input stream).

**Estimated decision points:** ~300

---

### Chunk 11: Command Pattern Layer

**Scope:** All command implementations for encapsulating jBPM operations.

| Item | Value |
|------|-------|
| Packages | `command`, `command.impl` |
| Files | 36 |
| LOC | ~3,550 |
| Key classes | 35 command classes including `StartProcessInstanceCommand`, `SignalCommand`, `CancelProcessInstanceCommand`, `GetTaskListCommand`, `DeployProcessCommand`, `ChangeProcessInstanceVersionCommand`, `CompositeCommand`, `AsynchronousCommand` |

**Decision types to inventory:**
- Command granularity: What operations get their own command vs what's combined?
  Why is `SignalCommand` separate from `TaskInstanceEndCommand`?
- Abstraction hierarchy: `AbstractBaseCommand` → `AbstractGetObjectBaseCommand`
  → `AbstractProcessInstanceBaseCommand` → concrete commands. What logic lives
  at which level?
- Parameter validation: What's checked upfront vs lazily? Required vs optional
  parameters.
- Transaction scoping: Each command executes in its own JbpmContext. What
  happens when commands are composed (`CompositeCommand`)?
- Async execution: `AsynchronousCommand` — how commands are serialized to
  messages and re-executed.
- Query commands: What data is returned, what filtering is supported, what
  ordering is applied. Why certain query shapes were chosen.
- Error handling: What exceptions are thrown for invalid operations (cancel
  already-ended process, signal already-ended token, etc.)?

**Estimated decision points:** ~400

---

### Chunk 12: Job Execution and Scheduling

**Scope:** Asynchronous job processing, timers, and the job executor thread pool.

| Item | Value |
|------|-------|
| Packages | `job`, `job.executor`, `scheduler.def`, `scheduler.db` |
| Files | 17 |
| LOC | ~2,084 |
| Key classes | `Job`, `Timer`, `ExecuteActionJob`, `ExecuteNodeJob`, `SignalTokenJob`, `CleanUpProcessJob`, `JobExecutor`, `JobExecutorThread`, `DispatcherThread`, `LockMonitorThread`, `CreateTimerAction`, `CancelTimerAction` |

**Decision types to inventory:**
- Job type hierarchy: Why these 6 job types? What determines whether an
  operation becomes a Timer vs ExecuteNodeJob vs SignalTokenJob?
- Thread architecture: Dispatcher + Executor + LockMonitor as separate threads.
  Why this decomposition?
- Lock acquisition: How jobs are locked for execution. Optimistic locking
  via version column. The `lockOwner` field and `lockTime` semantics.
- Retry logic: Failed jobs get retried up to N times (default 3). What
  constitutes a retryable failure? How `retries` is decremented.
- Timer execution: How timer due dates are calculated. Business calendar
  integration. Repeat intervals.
- Job ordering: How jobs are prioritized for execution (by dueDate, then by
  creation order).
- Cleanup: `CleanUpProcessJob` — what gets cleaned up when a process ends.
- Concurrency: Multiple job executor instances. How they coordinate via
  database locks to avoid duplicate execution.

**Estimated decision points:** ~250

---

### Chunk 13: Instantiation, Security, Services, and Cross-cutting

**Scope:** Object instantiation/delegation, security model, service
abstraction, and transaction management.

| Item | Value |
|------|-------|
| Packages | `instantiation`, `security`, `security.authentication`, `security.authorization`, `security.permission`, `svc`, `svc.save`, `tx`, `logging`, `logging.db`, `logging.exe`, `logging.log` |
| Files | 47 |
| LOC | ~3,034 |
| Key classes | `Delegation`, `ProcessClassLoaderFactory`, `DefaultProcessClassLoaderFactory`, `SharedProcessClassLoaderFactory`, `DefaultAuthenticationService`, `IdentityAuthorizationService`, `Services`, `ServiceFactory`, `SaveLogsOperation`, `CascadeProcessInstance` |

**Decision types to inventory:**
- Instantiation strategies: How action handlers, assignment handlers, and
  decision handlers are instantiated from class names in process definitions.
  Field injection, bean injection, constructor injection, configuration
  injection.
- Class loading: `ProcessClassLoaderFactory` — why two implementations
  (default vs shared)? When each is used. How process archive classes are
  isolated from the system classpath.
- Authentication: Pluggable authentication via `AuthenticationServiceFactory`.
  `DefaultAuthenticationService` using thread-local actor ID.
- Authorization: `IdentityAuthorizationService` — which operations are
  authorized (deploy, create instance, assign task, etc.).
- Permission model: 6 concrete permissions (CreateProcessInstance,
  DeployProcess, EndTask, SubmitTaskParameters, TaskAssign,
  ViewTaskParameters). Why these and not others?
- Service abstraction: The `Services` registry and `ServiceFactory` pattern.
  How services are discovered and instantiated.
- Save operations: The ordered chain of save operations when a JbpmContext
  closes (SaveLogsOperation, CascadeProcessInstance, etc.). Why this
  ordering?
- Transaction management: `TxService` — when to commit vs rollback. How
  transaction boundaries map to JbpmContext lifecycle.
- Logging infrastructure: `LoggingService` — database-backed vs in-memory.
  `ProcessLog` hierarchy (what's the base class for all audit logs).

**Estimated decision points:** ~400

---

### Chunk 14: Utilities, Calendar, Mail, and Infrastructure

**Scope:** Supporting libraries and cross-cutting infrastructure.

| Item | Value |
|------|-------|
| Packages | `util`, `bytes`, `calendar`, `mail`, `web`, `ant`, `msg`, `msg.db`, `file.def`, `jcr`, `jcr.impl`, `jcr.jackrabbit`, `jcr.jndi` |
| Files | 45 |
| LOC | ~4,076 |
| Key classes | `ClassLoaderUtil`, `IoUtil`, `XmlUtil`, `EqualsUtil`, `Clock`, `ByteArray`, `BusinessCalendar`, `Duration`, `Holiday`, `Day`, `MailAction`, `MailNode`, `FileDefinition`, `JCRService` |

**Decision types to inventory:**
- Business calendar: Working hours definition, holiday handling, duration
  parsing ("2 business days", "3 hours"). How weekends are determined. The
  `jbpm.business.calendar.properties` configuration.
- ByteArray: Block-based storage (1024-byte blocks). Why not a single BLOB?
  How blocks are assembled/disassembled.
- Mail: Template resolution (`jbpm.mail.templates.xml`), recipient resolution
  (actor ID → email address via `AddressResolver`), SMTP configuration.
  MailAction vs MailNode — when to use each.
- Class loading utilities: `ClassLoaderUtil` — classpath scanning, resource
  loading, thread context class loader vs system class loader.
- File storage: `FileDefinition` — how process-associated files (images,
  forms) are stored and retrieved.
- JCR integration: Content repository access patterns. Jackrabbit vs JNDI
  lookup.
- Web integration: Servlet listeners, context initialization.
- Ant tasks: Build-time operations (deploy process, generate schema).
- Clock: `Clock.getCurrentTime()` — the decision to abstract time for
  testability.

**Estimated decision points:** ~350

---

### Chunk 15: Hibernate ORM Mappings

**Scope:** All 105 `.hbm.xml` mapping files — every ORM decision.

| Item | Value |
|------|-------|
| Location | `core/src/main/resources/org/jbpm/**/*.hbm.xml` |
| Files | 105 |
| Format | Hibernate XML mappings |
| Key tables | JBPM_PROCESSDEFINITION, JBPM_NODE, JBPM_TRANSITION, JBPM_ACTION, JBPM_TOKEN, JBPM_TASKINSTANCE, JBPM_VARIABLEINSTANCE, JBPM_JOB, JBPM_LOG, etc. |

**Decision types to inventory:**
- **Inheritance strategy:** Consistent use of table-per-class-hierarchy with
  single-character discriminators. Why not table-per-subclass? The
  discriminator value assignments (P, N, D, F, J, etc.).
- **Cascade settings:** `cascade="all"` on parent→child collections,
  `cascade="lock"` on Job references, `cascade="save-update"` on `any`
  elements. Why different cascade strategies for different relationships?
- **Fetch modes:** Almost everything is `lazy="true"` (default). Only
  `ProcessLog` has explicit `lazy="false"`. Why?
- **Collection types:** Maps (35+), lists (25+), sets (15+). Why a map for
  `events` but a list for `nodes`? Why a set for `arrivingTransitions` but
  a list for `leavingTransitions`?
- **Optimistic locking:** Version columns on ProcessInstance, Token, Comment,
  TaskInstance, SwimlaneInstance, PooledActor, TokenVariableMap, Job. Why
  these entities and not others (e.g., ProcessDefinition has no version)?
- **Polymorphic associations:** The `any` element for Event.graphElement,
  ExceptionHandler.graphElement, RuntimeAction.graphElement — supporting 11
  possible types. Why `any` instead of a common base table?
- **Key strategy:** All entities use surrogate keys (native generator). No
  composite keys or natural keys.
- **Foreign key naming and indexing:** Explicit `foreign-key` and `index`
  attributes for performance.
- **Bidirectional relationships:** `insert="false" update="false"` on inverse
  sides. Which side is the owner?
- **Custom types:** `AccessType` for variable access modes.

**Estimated decision points:** ~500

---

### Chunk 16: XML Configuration and Runtime Defaults

**Scope:** All configuration files that determine runtime behavior —
jbpm.cfg.xml, hibernate configs, properties, node/action type registries.

| Item | Value |
|------|-------|
| Location | `core/src/main/resources/`, `core/src/test/resources/`, non-core `src/main/resources/` |
| Files | 60+ |
| Format | XML (cfg.xml, properties) |

**Decision types to inventory:**
- **Service defaults** (from `default.jbpm.cfg.xml`):
  - Authentication: `DefaultAuthenticationServiceFactory`
  - Logging: `DbLoggingServiceFactory` (DB-backed)
  - Messaging: `DbMessageServiceFactory` (DB-backed)
  - Persistence: `DbPersistenceServiceFactory` (DB-backed)
  - Scheduler: `DbSchedulerServiceFactory` (DB-backed)
  - Transactions: `TxServiceFactory`
- **Job executor defaults:** 1 thread, 60s idle interval, 4s retry interval,
  10min max lock time, 3 retries. Why these specific values?
- **Database defaults:** HSQLDB in-memory, schema auto-update, no production
  database out of the box.
- **Cache strategy:** `HashtableCacheProvider` with `nonstrict-read-write`.
  Why not a more sophisticated cache?
- **Node type registry** (`node.types.xml`): Which node types are registered
  and their XML element names. Extensibility mechanism.
- **Action type registry** (`action.types.xml`): Which action types are
  registered.
- **Parser registry** (`jbpm.parsers.xml`): Which parsers handle which parts
  of a process archive.
- **Variable mapping** (`jbpm.varmapping.xml`): The ordered list of
  variable-type-to-VariableInstance mappings. Matcher priority.
- **Default modules** (`jbpm.default.modules.properties`): Which module
  definitions are automatically added to every process definition.
- **Test configuration variants** (19 test configs): What each variant changes
  and why (no-logging, async-subprocess, custom job executor, different
  databases, custom expression evaluator, Gmail SMTP, etc.).
- **Database-specific configurations:** 7 database dialects (HSQLDB, MySQL,
  Oracle, PostgreSQL, DB2, Derby, MSSQL, Sybase) — what differs between them
  (isolation levels, type mappings, schema strategies).
- **Mail templates** (`jbpm.mail.templates.xml`): Default email templates
  for task-assign and task-reminder.
- **Business calendar** (`jbpm.business.calendar.properties`): Working hours
  and holiday definitions.

**Estimated decision points:** ~400

---

### Chunk 17: Identity Module

**Scope:** User/group/membership identity management — a self-contained module.

| Item | Value |
|------|-------|
| Location | `/home/user/jBPM3/identity/` |
| Main files | 21 |
| Test files | 6 |
| LOC | ~2,000 |
| Key classes | `User`, `Group`, `Membership`, `Entity`, `IdentitySession`, `IdentitySessionFactory`, `IdentitySchema`, `ExpressionAssignmentHandler`, `IdentityLoginModule`, `IdentityAddressResolver`, `IdentityXmlParser` |

**Decision types to inventory:**
- Entity model: `Entity` → `User` / `Group` hierarchy with discriminator.
  `Membership` as a join entity with role. Why not a simpler user-role
  model?
- Expression assignment: The `ExpressionAssignmentHandler` mini-language for
  task assignment (`user(fred)`, `group(managers).member(boss)`, etc.). What
  expressions are supported?
- Authentication: `IdentityLoginModule` (JAAS). What credentials are checked,
  how password hashing works.
- Authorization: `IdentityPolicy` — permission checking against identity data.
- Mail resolution: `IdentityAddressResolver` — how user entities map to
  email addresses.
- XML import: `IdentityXmlParser` — the identity data XML format.
- Hibernate mapping: `hibernate.identity.hbm.xml` — discriminator-based
  inheritance, permission sets with custom `PermissionUserType`.

**Estimated decision points:** ~200

---

### Chunk 18: Simulation Module

**Scope:** Process simulation engine using Discrete Event Simulation (DES).

| Item | Value |
|------|-------|
| Location | `/home/user/jBPM3/simulation/` |
| Main files | 66 |
| Test files | 11 |
| Tutorial files | 10 |
| LOC | ~5,000 |
| Key classes | `JbpmSimulationModel`, `JbpmSimulationExperiment`, `JbpmSimulationScenario`, `ResourcePool`, `DistributionDefinition`, `TokenEntity`, `TaskInstanceEntity`, `ExperimentReader`, `SimulationJpdlXmlReader`, `ExperimentReport`, `ScenarioReport` |

**Decision types to inventory:**
- Simulation architecture: Why DES (Discrete Event Simulation)? How DESMO-J
  framework is integrated.
- Node overrides: `SimDecision`, `SimState`, `SimTaskNode` — how each node
  type's behavior is modified for simulation (replacing real execution with
  statistical models).
- Resource modeling: `ResourcePool` and `ResourceRequirement` — how human
  resources are modeled with capacity constraints and utilization tracking.
- Distribution definitions: What probability distributions are supported
  (normal, exponential, uniform, etc.) for task durations and arrival rates.
- Transition probabilities: How branching decisions are modeled statistically
  instead of deterministically.
- Data sources: `ProcessDataSource` and `ProcessDataFilter` — how historical
  data feeds the simulation.
- KPI calculation: `BusinessFigure` — how business metrics are computed from
  simulation runs.
- Reporting: What statistics are collected (queue lengths, utilization rates,
  cycle times) and how they're aggregated across experiment runs.
- Experiment configuration: The XML format for defining simulation scenarios,
  experiments, and resource pools.
- User code interception: `SimulationUserCodeInterceptor` — how real action
  handlers are replaced with simulation stubs.

**Estimated decision points:** ~400

---

### Chunk 19: JSF Console Component Library

**Scope:** The jBPM4JSF tag library — JSF components for process management.

| Item | Value |
|------|-------|
| Location | `/home/user/jBPM3/jsf-console/jbpm4jsf/` |
| Main files | 131 |
| Test files | 0 |
| LOC | ~8,000 |
| Key classes | 37 ActionListeners, 39 Handlers, `JbpmActionListener`, `JbpmJsfContext`, `JbpmPhaseListener`, `UITaskForm`, `TaskFormLayout`, plus 12 identity action/handler pairs |

**Decision types to inventory:**
- Component architecture: The ActionListener + Handler pattern. Why this
  two-layer design? What logic lives in each layer?
- JSF lifecycle integration: `JbpmPhaseListener` — which JSF phases trigger
  jBPM operations. How JbpmContext is opened/closed relative to JSF phases.
- Operation coverage: 37 distinct operations exposed as JSF tags (list
  processes, load process, start process, signal, complete task, assign task,
  suspend, resume, cancel, deploy, get variable, set variable, add comment,
  move token, etc.). Why these operations and not others?
- Identity operations: 12 identity-specific operations (create/delete/load
  user/group, add/delete membership). Separate tag library for identity.
- Task form system: `UITaskForm`, `TaskFormLayout` — how task forms are
  rendered from process-defined variable mappings.
- Error handling: How each action handles jBPM exceptions. What feedback is
  given to the user.
- Configuration: `jbpm4jsf-config.xml` — JSF authentication mapping to jBPM
  actors, process image URL patterns.
- Tag library design: 6 tag libraries (core, identity, task form, task form
  layout, plus compatibility versions). Why this separation?

**Estimated decision points:** ~500

---

### Chunk 20: Enterprise Integration and Examples

**Scope:** JEE5 integration, Tomcat integration, web console configuration,
and example process definitions.

| Item | Value |
|------|-------|
| Location | `enterprise-jee5/`, `tomcat/`, `jsf-console/console*/`, `jsf-console/gpd-deployer/`, `examples/` |
| Main files | 9 |
| Test/example files | 30 |
| Config files | 20+ |
| Key classes | `CommandServiceBean`, `CommandListenerBean`, `JobListenerBean`, `JmsConnectorService`, `DataSourceRealm`, `ProcessUploadServlet` |

**Decision types to inventory:**
- EJB architecture: Stateless session bean for synchronous commands, MDBs
  for async commands and jobs. Why MDB with maxSession=150? DLQ
  configuration.
- JMS integration: `JmsConnectorService` — how async messages are sent.
  Queue naming (`queue/JbpmCommandQueue`, `queue/JbpmJobQueue`).
- JNDI resource mapping: DataSource, ConnectionFactory, Timer entity bean
  lookups. JBoss-specific JNDI names.
- Tomcat integration: `DataSourceRealm` — custom Tomcat Realm for
  authenticating against jBPM identity tables.
- Web console configuration: Form-based auth vs BASIC auth, security
  constraints, role definitions, Facelets configuration.
- Deployment variants: Console-JEE5 vs Console-Tomcat — what differs
  between deployment targets.
- Process upload: `ProcessUploadServlet` — how process archives are deployed
  via HTTP.
- Example processes (9 process definitions): Each example embodies specific
  design patterns:
  - Simple action handling
  - Rules-based assignment (Drools integration)
  - Business trip workflow with forms
  - Custom task instances and factories
  - E-commerce websale workflow with reminders
  - Door process (state machine)

**Estimated decision points:** ~300

---

## Summary Table

| # | Chunk | Files | LOC | Est. Decisions |
|--:|-------|------:|--------:|---------------:|
| 1 | Graph Definition Model | 15 | 3,182 | 400 |
| 2 | Graph Node Types and Actions | 25 | 2,726 | 350 |
| 3 | Process Execution and Token System | 15 | 2,400 | 300 |
| 4 | JPDL XML Parsing and Process Archives | 15 | 2,207 | 350 |
| 5 | EL Grammar and Parser | 13 | 4,260 | 200 |
| 6 | EL Evaluator and Type System | 51 | 9,822 | 600 |
| 7 | Task Management | 20 | 2,730 | 350 |
| 8 | Variable and Context System | 49 | 3,194 | 350 |
| 9 | Database and Persistence Layer | 28 | 5,156 | 500 |
| 10 | Configuration and Object Factory | 29 | 3,437 | 300 |
| 11 | Command Pattern Layer | 36 | 3,550 | 400 |
| 12 | Job Execution and Scheduling | 17 | 2,084 | 250 |
| 13 | Instantiation, Security, Services | 47 | 3,034 | 400 |
| 14 | Utilities, Calendar, Mail, Infra | 45 | 4,076 | 350 |
| 15 | Hibernate ORM Mappings | 105 | XML | 500 |
| 16 | XML Configuration and Defaults | 60+ | XML | 400 |
| 17 | Identity Module | 27 | 2,000 | 200 |
| 18 | Simulation Module | 87 | 5,000 | 400 |
| 19 | JSF Console Component Library | 131 | 8,000 | 500 |
| 20 | Enterprise Integration and Examples | 39 | 2,000 | 300 |
| | **Totals** | **~854** | **~67,000+** | **~7,400** |

## Suggested Execution Order

Work chunks in dependency order — later chunks reference concepts from
earlier ones:

1. **Phase 1 (Foundation):** Chunks 1, 10, 15, 16
   Graph model, configuration system, ORM mappings, and config defaults.
   These define the vocabulary and structure everything else builds on.

2. **Phase 2 (Core Engine):** Chunks 2, 3, 4, 5, 6
   Node types, execution runtime, JPDL parsing, and expression language.
   The behavioral core of the workflow engine.

3. **Phase 3 (Domain Features):** Chunks 7, 8, 12
   Task management, variable system, and job scheduling.
   The user-facing features built on the core engine.

4. **Phase 4 (Infrastructure):** Chunks 9, 11, 13, 14
   Persistence, commands, security/services, and utilities.
   Cross-cutting concerns that support the domain features.

5. **Phase 5 (Modules):** Chunks 17, 18, 19, 20
   Identity, simulation, JSF console, and enterprise integration.
   Self-contained modules built on the core.

## Output Format Per Chunk

Each chunk's inventory should produce a structured catalog:

```
File: org/jbpm/graph/node/Decision.java
Line 45: BRANCH — if (decisionDelegation != null) → uses delegation handler;
         else → evaluates conditions on transitions.
         Alternative: could always require delegation, or always use conditions.
Line 67: DEFAULT — condition evaluation order is list-index order.
         Alternative: could be alphabetical, priority-based, or random.
Line 82: NULL — if no condition matches and no default transition, throws
         JbpmException. Alternative: could take first transition, or stay in node.
...
```

This format captures: file, line, decision category, what was decided, and
what alternative(s) existed.
