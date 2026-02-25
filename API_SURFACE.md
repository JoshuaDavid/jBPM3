# jBPM 3 — API Surface Documentation

## What Is This?

**jBPM 3** (Java Business Process Management, version 3.2.14.SP1) is a workflow and business process management engine for Java. It provides an embeddable, lightweight framework for defining, executing, and managing long-running business processes using a graph-based execution model.

This repository is based on the source of version 3.2.14 shipped with JBoss Enterprise SOA-P 5.3.1. jBPM 3 is **no longer under active community development** — the public SVN source is frozen and sustaining work happens internally.

**Maven coordinates:** `org.jbpm.jbpm3:jbpm:3.2.14.SP1` (parent POM)
**License:** LGPL
**Java target:** 1.5
**Persistence:** Hibernate 3.2

---

## Module Structure

| Module | Artifact | Description |
|--------|----------|-------------|
| `core` | jbpm-jpdl | Core process engine, graph model, task management, persistence, services |
| `identity` | jbpm-identity | User/group/membership identity model with Hibernate persistence |
| `enterprise-jee5` | jbpm-enterprise-jee5 | JEE5 integration (EJBs, JMS connector) |
| `simulation` | jbpm-simulation | Process simulation using DES (Discrete Event Simulation) |
| `jsf-console` | jsf-console, jbpm4jsf, gpd-deployer | Web console for process management (JSF-based) |
| `tomcat` | jbpm-tomcat | Tomcat integration |
| `examples` | jbpm-examples | Example process definitions and usage |
| `userguide` | jbpm-userguide | DocBook user guide |
| `db` | jbpm-db | Database schema scripts for supported databases |
| `distribution` | jbpm-distribution | Distribution packaging |

---

## Core API Surface

### 1. Bootstrap & Configuration (`org.jbpm`)

| Class | Role |
|-------|------|
| `JbpmConfiguration` | **Entry point.** Thread-safe singleton factory. Loads `jbpm.cfg.xml`, creates `JbpmContext` instances, manages the `JobExecutor`. |
| `JbpmContext` | **Unit-of-work handle.** Wraps a Hibernate session + transaction. Provides convenience methods for deploying processes, creating instances, loading tasks, signaling tokens. Used in a try/finally pattern. |
| `JbpmException` | Base runtime exception for all jBPM errors. |

**Key `JbpmConfiguration` methods:**
- `getInstance()` / `getInstance(String resource)` — obtain singleton from XML config
- `parseXmlString(String xml)` — create from inline XML
- `createJbpmContext()` / `createJbpmContext(String name)` — create a unit-of-work context
- `getJobExecutor()` — get the asynchronous job executor
- `close()` — shut down and release resources

**Key `JbpmContext` methods:**
- `deployProcessDefinition(ProcessDefinition)` — persist a process definition
- `newProcessInstance(String processDefName)` / `newProcessInstanceForUpdate(String)` — start a new instance
- `loadProcessInstance(long id)` / `loadProcessInstanceForUpdate(long id)` — retrieve existing instance
- `loadTaskInstance(long id)` / `loadTaskInstanceForUpdate(long id)` — retrieve a task
- `getTaskList(String actorId)` / `getGroupTaskList(List actorIds)` — query tasks by actor
- `save(ProcessInstance)` / `save(Token)` / `save(TaskInstance)` — explicitly persist objects
- `getGraphSession()` / `getTaskMgmtSession()` / `getLoggingSession()` / `getContextSession()` / `getJobSession()` — access specialized DAO sessions
- `getSession()` / `getSessionFactory()` / `getConnection()` — direct Hibernate/JDBC access
- `setActorId(String)` — set the authenticated user for this context
- `close()` — commit/rollback and release resources

### 2. Process Definition Model (`org.jbpm.graph.def`)

These classes define the **static structure** of a process (the "blueprint").

| Class/Interface | Role |
|----------------|------|
| `ProcessDefinition` | Top-level container. Parses from jPDL XML (`parseXmlString`, `parseXmlResource`, `parseParResource`). Contains nodes, transitions, actions, module definitions. |
| `Node` | A step in the process graph. Subclassed by node types. |
| `Transition` | Directed edge between nodes. Has a name and optional guard condition. |
| `Action` | Executable behavior attached to events (via delegation to user `ActionHandler`). |
| `Event` | Named lifecycle event on a graph element (e.g., `node-enter`, `node-leave`, `transition`). |
| `SuperState` | Composite node containing nested nodes (hierarchical process). |
| `ExceptionHandler` | Maps exception types to actions for error handling. |
| `GraphElement` | Abstract base for all graph elements (nodes, transitions, process def). |
| **`ActionHandler`** | **User-implemented interface:** `void execute(ExecutionContext)` — custom logic triggered by actions. |
| `NodeCollection` | Interface for containers of nodes (ProcessDefinition, SuperState). |

### 3. Node Types (`org.jbpm.graph.node`)

| Class | Description |
|-------|-------------|
| `StartState` | The single entry point of a process. |
| `EndState` | Terminal node that ends the process (or a token path). |
| `State` | Wait state — execution pauses until an external signal. |
| `TaskNode` | Creates task instances for human participants. Configurable signal behavior (last, first, never, unsynchronized). |
| `Decision` | Routing node. Uses a `DecisionHandler` or jPDL expression to pick an outgoing transition. |
| `Fork` | Splits a token into concurrent child tokens (parallel execution). |
| `Join` | Synchronizes concurrent tokens back into one. |
| `ProcessState` | Sub-process invocation — delegates to another `ProcessDefinition`. |
| `MailNode` | Sends email as part of process flow. |
| `Merge` | Merges multiple incoming paths (no synchronization). |

**User-implemented interfaces:**
- **`DecisionHandler`**: `String decide(ExecutionContext)` — return the name of the outgoing transition.
- **`SubProcessResolver`**: `ProcessDefinition findSubProcess(Element)` — custom sub-process lookup.

### 4. Process Execution Model (`org.jbpm.graph.exe`)

These are the **runtime instances** created when a process runs.

| Class | Role |
|-------|------|
| `ProcessInstance` | A running instance of a `ProcessDefinition`. Has a root token, context variables, module instances. |
| `Token` | A pointer to a current node in the graph. Supports hierarchical tokens for concurrent paths. |
| `ExecutionContext` | Passed to all handler callbacks. Provides access to the current token, node, process instance, task instance, and variables. |
| `RuntimeAction` | An action added at runtime (not defined in the process definition). |
| `Comment` | Text comment attachable to tokens or task instances. |

**Key `ProcessInstance` methods:**
- `signal()` / `signal(String transitionName)` — advance the root token
- `end()` — terminate the process
- `getContextInstance()` — get the variable scope
- `getTaskMgmtInstance()` — get task management runtime
- `getRootToken()` — get the main execution path
- `isSuspended()` / `suspend()` / `resume()`

**Key `Token` methods:**
- `signal()` / `signal(String transitionName)` — advance this token along a transition
- `getNode()` — current node
- `getProcessInstance()` — owning process instance
- `getParent()` / `getChildren()` — token hierarchy (for Fork/Join)
- `suspend()` / `resume()` / `lock(String)` / `unlock(String)`

### 5. Task Management (`org.jbpm.taskmgmt`)

#### Definition (`org.jbpm.taskmgmt.def`)

| Class/Interface | Role |
|----------------|------|
| `Task` | A human task definition. Has a name, assignment, due date expression, controller. |
| `Swimlane` | A role-based assignment. Tasks in the same swimlane are assigned to the same actor. |
| `TaskController` | Maps process variables to/from task form variables. |
| `TaskMgmtDefinition` | Module definition holding all tasks and swimlanes for a process. |
| **`AssignmentHandler`** | **User-implemented interface:** `void assign(Assignable, ExecutionContext)` — assign a task or swimlane to an actor. |
| **`TaskControllerHandler`** | **User-implemented interface:** custom variable marshaling for task forms. |

#### Execution (`org.jbpm.taskmgmt.exe`)

| Class/Interface | Role |
|----------------|------|
| `TaskInstance` | A runtime task assigned to an actor. Can be started, completed, cancelled. |
| `TaskMgmtInstance` | Runtime module managing task instances for a process instance. |
| `SwimlaneInstance` | Runtime swimlane with its current actor assignment. |
| `PooledActor` | Represents a candidate (group member) for pooled task assignment. |
| **`Assignable`** | Interface: `setActorId(String)`, `setPooledActors(String[])`. |
| **`TaskInstanceFactory`** | User-implementable factory for custom task instance creation. |

**Key `TaskInstance` methods:**
- `start()` — mark task as started
- `end()` / `end(String transitionName)` — complete the task, optionally choosing a transition
- `cancel()` — cancel without completing
- `setActorId(String)` / `setPooledActors(String[])` — assignment
- `getVariable(String)` / `setVariable(String, Object)` — task-scoped variables
- `addComment(String)` — attach a comment
- `isOpen()` / `isCancelled()` / `isSignalling()`

### 6. Context & Variables (`org.jbpm.context`)

| Class | Role |
|-------|------|
| `ContextInstance` | Runtime variable scope for a process instance. Variables are scoped to tokens. |
| `VariableInstance` | Abstract base for persisted variable values (Long, String, Date, ByteArray, HibernateLong, etc.). |
| `VariableAccess` | Declares a variable mapping (name, access mode: read/write/required) used by task controllers. |
| `VariableContainer` | Abstract base for objects that hold variables (ContextInstance, TaskInstance). |

**Key `ContextInstance` methods:**
- `setVariable(String name, Object value)` / `setVariable(String, Object, Token)`
- `getVariable(String name)` / `getVariable(String, Token)`
- `hasVariable(String)` / `deleteVariable(String)`
- `getVariables()` / `getVariableLocally(String, Token)`
- `setTransientVariable(String, Object)` / `getTransientVariable(String)` — non-persistent variables

### 7. Database Sessions (DAOs) (`org.jbpm.db`)

| Class | Role |
|-------|------|
| `GraphSession` | CRUD and queries for `ProcessDefinition` and `ProcessInstance`. |
| `TaskMgmtSession` | Queries for task instances by actor, group, or process instance. |
| `ContextSession` | (Minimal) persistence operations for context/variable data. |
| `LoggingSession` | Queries for process logs. |
| `JobSession` | CRUD for asynchronous jobs (timers, async continuations). |
| `JbpmSchema` | DDL operations: create, drop, update, clean the jBPM database schema. |

**Key `GraphSession` methods:**
- `deployProcessDefinition(ProcessDefinition)`
- `findLatestProcessDefinition(String name)` / `findProcessDefinition(String name, int version)`
- `findAllProcessDefinitions()` / `findAllProcessDefinitionVersions(String name)`
- `loadProcessInstance(long id)` / `findProcessInstances(long processDefinitionId)`
- `deleteProcessDefinition(long id)` / `deleteProcessInstance(long id)`
- `saveProcessInstance(ProcessInstance)` / `lockProcessInstance(long id)`

**Key `TaskMgmtSession` methods:**
- `findTaskInstances(String actorId)`
- `findTaskInstancesByIds(List taskInstanceIds)`
- `findPooledTaskInstances(String actorId)` / `findPooledTaskInstances(List actorIds)`
- `findTaskInstancesByProcessInstance(ProcessInstance)`

### 8. Command Pattern (`org.jbpm.command`)

A command-based API for remote/transactional invocation.

| Interface/Class | Role |
|----------------|------|
| **`Command`** | `Object execute(JbpmContext)` — a serializable unit of work. |
| **`CommandService`** | `Object execute(Command)` — executes commands (local or remote). |
| `DeployProcessCommand` | Deploy a process definition from XML or a PAR. |
| `NewProcessInstanceCommand` | Create a new process instance. |
| `StartProcessInstanceCommand` | Create and signal a new process instance. |
| `SignalCommand` | Signal a token or process instance. |
| `GetProcessInstanceCommand` | Retrieve a process instance by ID. |
| `GetProcessDefinitionCommand` / `GetProcessDefinitionsCommand` | Retrieve process definition(s). |
| `GetTaskListCommand` / `GetTaskInstanceCommand` | Query tasks. |
| `TaskInstanceEndCommand` | Complete a task instance. |
| `CancelProcessInstanceCommand` / `CancelTokenCommand` | Cancellation commands. |
| `SuspendProcessInstanceCommand` / `ResumeProcessInstanceCommand` | Suspend/resume. |
| `ChangeProcessInstanceVersionCommand` | Migrate a running instance to a new process version. |
| `DeleteProcessDefinitionCommand` | Delete a process definition. |
| `VariablesCommand` | Get/set variables on a token or task. |
| `BatchSignalCommand` | Signal multiple tokens at once. |
| `CompositeCommand` | Execute multiple commands as a batch. |

### 9. Asynchronous Execution (`org.jbpm.job`, `org.jbpm.job.executor`)

| Class | Role |
|-------|------|
| `Job` | Abstract base for asynchronous work items (persisted in DB). |
| `Timer` | A scheduled job that fires at a specific time, with optional repeat. |
| `ExecuteActionJob` | Async execution of an action. |
| `ExecuteNodeJob` | Async continuation at a node. |
| `SignalTokenJob` | Async signal of a token. |
| `JobExecutor` | Multi-threaded job executor. Polls the DB for due jobs and dispatches them. Configured via `JbpmConfiguration`. |

### 10. Services SPI (`org.jbpm.svc`, `org.jbpm.persistence`, etc.)

| Interface | Role |
|-----------|------|
| `Service` | Base interface for all pluggable services (`close()`). |
| `ServiceFactory` | Creates service instances. Configured in `jbpm.cfg.xml`. |
| `PersistenceService` | SPI for persistence operations (default: Hibernate-based `DbPersistenceService`). |
| `AuthenticationService` | SPI for resolving the current actor ID. |
| `AuthorizationService` | SPI for authorization checks. |
| `MessageService` | SPI for async messaging (default: DB-based; JMS in enterprise module). |
| `SchedulerService` | SPI for timer scheduling. |
| `LoggingService` | SPI for process event logging. |
| `EventService` | SPI for event publication. |

### 11. Identity Module (`org.jbpm.identity`)

A simple, optional identity component.

| Class | Role |
|-------|------|
| `Entity` | Abstract base (id + name). |
| `User` | A user with email and password. |
| `Group` | A named group (type: e.g., "team", "department"). |
| `Membership` | Associates a user with a group and a role. |
| `IdentitySession` | Hibernate DAO for identity CRUD and queries. |
| `ExpressionAssignmentHandler` | `AssignmentHandler` that resolves assignments via identity expressions (e.g., `previous -> group(manager) -> member`). |

### 12. Enterprise / JEE5 Integration (`org.jbpm.ejb`, `org.jbpm.jms`)

| Class | Role |
|-------|------|
| `CommandServiceBean` | Stateless session EJB that implements `CommandService` — executes jBPM commands within a managed JTA transaction. |
| `LocalCommandService` | Local EJB interface for `CommandServiceBean`. |
| `CommandListenerBean` | MDB that receives JMS messages containing `Command` objects and executes them. |
| `JobListenerBean` | MDB for processing async jobs via JMS. |
| `JmsConnectorService` | `MessageService` implementation that sends async jobs to a JMS queue. |

### 13. Web Integration (`org.jbpm.web`)

| Class | Role |
|-------|------|
| `JbpmContextFilter` | Servlet filter that creates/closes a `JbpmContext` per request. |
| `JobExecutorLauncher` | `ServletContextListener` that starts/stops the `JobExecutor` with the web app. |
| `CloseJbpmConfigurationServlet` | Servlet that closes `JbpmConfiguration` on `destroy()`. |

### 14. Process Definition Parsing (`org.jbpm.jpdl`)

| Class/Interface | Role |
|----------------|------|
| `JpdlXmlReader` | Parses jPDL XML into a `ProcessDefinition` object graph. |
| `ProcessArchive` | Reads a PAR (Process Archive — a ZIP containing `processdefinition.xml` + resources). |
| **`ProcessArchiveParser`** | User-implementable SPI for custom PAR parsing. |
| `Parsable` | Interface for objects that can be initialized from XML elements. |

### 15. Scripting & Expressions

| Feature | Class |
|---------|-------|
| jPDL EL (Expression Language) | `org.jbpm.jpdl.el.*` — custom EL implementation for expressions in process definitions. |
| BeanShell scripting | `org.jbpm.graph.action.Script` — execute BeanShell scripts as actions. |
| Business Calendar | `org.jbpm.calendar.BusinessCalendar` — resolve business-time durations (e.g., "3 business days"). |

---

## Key User-Implementable Interfaces

These are the extension points where application developers write custom logic:

| Interface | Package | Method to Implement |
|-----------|---------|-------------------|
| `ActionHandler` | `graph.def` | `void execute(ExecutionContext)` |
| `DecisionHandler` | `graph.node` | `String decide(ExecutionContext)` |
| `AssignmentHandler` | `taskmgmt.def` | `void assign(Assignable, ExecutionContext)` |
| `TaskControllerHandler` | `taskmgmt.def` | Variable marshaling for task forms |
| `TaskInstanceFactory` | `taskmgmt` | Custom task instance creation |
| `SubProcessResolver` | `graph.node` | Custom sub-process lookup |
| `ProcessArchiveParser` | `jpdl.par` | Custom process archive parsing |
| `Command` | `command` | `Object execute(JbpmContext)` |
| `ProcessClassLoaderFactory` | `instantiation` | Custom class loading for process classes |
| `AddressResolver` | `mail` | Resolve actor IDs to email addresses |
| `UserCodeInterceptor` | `instantiation` | Intercept delegation class instantiation and execution |

---

## Typical Usage Pattern

```java
// 1. Bootstrap (once per application)
JbpmConfiguration config = JbpmConfiguration.getInstance();

// 2. Deploy a process definition
JbpmContext ctx = config.createJbpmContext();
try {
    ProcessDefinition pd = ProcessDefinition.parseXmlResource("myprocess/processdefinition.xml");
    ctx.deployProcessDefinition(pd);
} finally {
    ctx.close();
}

// 3. Start a process instance
ctx = config.createJbpmContext();
try {
    ProcessInstance pi = ctx.newProcessInstanceForUpdate("myprocess");
    pi.getContextInstance().setVariable("applicant", "John");
    pi.signal(); // advance past start state
} finally {
    ctx.close();
}

// 4. Query and complete tasks
ctx = config.createJbpmContext();
try {
    List tasks = ctx.getTaskList("manager");
    TaskInstance task = (TaskInstance) tasks.get(0);
    task.setVariable("approved", Boolean.TRUE);
    task.end("approve"); // complete task, follow "approve" transition
} finally {
    ctx.close();
}

// 5. Shutdown
config.close();
```
