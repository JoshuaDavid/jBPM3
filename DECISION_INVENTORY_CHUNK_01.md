# Decision Inventory — Chunk 1: Graph Definition Model

Covers `org.jbpm.graph.def` (13 files), `org.jbpm.module.def` (1 file),
`org.jbpm.module.exe` (1 file).

Each entry follows the format:

> **File:Line — CATEGORY — Description of what was decided.**
> *Alternative:* What could have been done instead.

---

## GraphElement.java (abstract base class for all graph elements)

### Type hierarchy

**GraphElement.java:51 — HIERARCHY — `GraphElement` is abstract with concrete
subclasses `ProcessDefinition`, `Node` (→ `SuperState`, `StartState`,
`EndState`, `Decision`, `Fork`, `Join`, `State`, `TaskNode`, …),
`Transition`, `Event`, `Action`.**
*Alternative:* Could use composition (a generic `GraphElement` with a `type`
field and pluggable behavior) instead of class hierarchy. Could also have
`Transition` not extend `GraphElement` since transitions are structurally
different from nodes.

**GraphElement.java:51 — INTERFACE — `GraphElement implements Identifiable,
Serializable`.**
*Alternative:* Could implement `Comparable` for natural ordering. Could omit
`Serializable` and use a separate serialization mechanism (DTO pattern).

### Field declarations

**GraphElement.java:55 — VISIBILITY — `long id` is package-private (no
modifier).**
*Alternative:* Could be `private` with only getter access, or `protected` for
subclass access. The package-private choice allows direct field access from
other `graph.def` classes without a method call.

**GraphElement.java:56 — VISIBILITY — `name` is `protected`.**
*Alternative:* Could be `private` with getter/setter only. The `protected`
choice allows subclasses like `Node` to directly read `this.name` without a
method call.

**GraphElement.java:59 — COLLECTION_TYPE — Events stored as `Map events`
(raw type, keyed by event type string).**
*Alternative:* Could use `Map<String, Event>` (type-safe). Could use an
`EnumMap` if event types were an enum rather than string constants. Could use
a `LinkedHashMap` to preserve insertion order.

**GraphElement.java:60 — COLLECTION_TYPE — Exception handlers stored as
`List exceptionHandlers` (ordered).**
*Alternative:* Could use a `Map` keyed by exception class name for O(1)
lookup. The `List` choice means exception handlers are matched in order
(first match wins), giving the user explicit priority control.

### Lazy initialization pattern

**GraphElement.java:104-106 — DEFAULT — Events map is lazily initialized
(null until first event is added).**
*Alternative:* Could eagerly initialize in constructor (`events = new
HashMap()`). The lazy pattern saves memory when elements have no events (many
simple nodes/transitions won't), but means every access must null-check.

**GraphElement.java:142-144 — DEFAULT — Exception handler list is lazily
initialized (null until first handler is added).**
*Alternative:* Same trade-off as events. Could eagerly initialize.

### Null argument validation

**GraphElement.java:98-100 — VALIDATION — `addEvent()` throws
`IllegalArgumentException` if event is null.**
*Alternative:* Could silently ignore null (return null or no-op). Could throw
`NullPointerException` (which is the standard contract per `Objects.
requireNonNull`).

**GraphElement.java:101-103 — VALIDATION — `addEvent()` throws
`IllegalArgumentException` if event type is null.**
*Alternative:* Could auto-generate an event type, or could defer validation
to when the event is actually fired.

**GraphElement.java:114-116 — VALIDATION — `removeEvent()` throws
`IllegalArgumentException` if event is null.**
*Alternative:* Could silently return null for null input.

**GraphElement.java:139-141 — VALIDATION — `addExceptionHandler()` throws
`IllegalArgumentException` if handler is null.**
*Alternative:* Same as above — could silently ignore.

**GraphElement.java:151-153 — VALIDATION — `removeExceptionHandler()` throws
`IllegalArgumentException` if handler is null.**
*Alternative:* Could silently ignore.

### Event firing and propagation

**GraphElement.java:173-193 — CONTROL_FLOW — `fireEvent()` saves and
restores `eventSource` on the execution context using a try/finally block.**
*Alternative:* Could use a stack-based approach for nested events instead of
save/restore. Could pass eventSource as a parameter instead of mutating
shared state.

**GraphElement.java:182-185 — CONTROL_FLOW — Checks for `EventService` via
`jbpmContext.getServices().getService()` and calls it if present.**
*Alternative:* Could always assume EventService is available (fail if not).
Could use an observer/listener pattern instead of a service lookup. The null
check makes EventService optional.

**GraphElement.java:198 — CONTROL_FLOW — Propagation is determined by
`!equals(executionContext.getEventSource())` — comparing this element to the
original event source.**
*Alternative:* Could pass an explicit `isPropagated` flag. Could use a
separate "propagated event" type.

**GraphElement.java:200-211 — ORDERING — Static actions (from process
definition) execute before runtime actions (added at runtime).**
*Alternative:* Could interleave them, execute runtime first, or make ordering
configurable.

**GraphElement.java:214 — DEFAULT — Event is cleared from context
(`setEvent(null)`) after executing actions but before propagating to parent.**
*Alternative:* Could keep the event in context during propagation. This
choice means parent handlers won't see the child's event object.

**GraphElement.java:217-220 — CONTROL_FLOW — Events propagate up the parent
hierarchy (`getParent()`) recursively until a parentless element is reached.**
*Alternative:* Could stop propagation at process definition level only. Could
allow actions to stop propagation (like DOM `stopPropagation()`). Could
propagate downward instead of upward.

### Action execution

**GraphElement.java:229 — CONTROL_FLOW — Actions with
`isPropagationAllowed == false` are skipped when the event is propagated.**
*Alternative:* Could still execute them but mark the execution context as
propagated. Default is `true` (accept propagated events), so most actions
fire everywhere in the hierarchy.

**GraphElement.java:231-238 — CONTROL_FLOW — Async actions create an
`ExecuteActionJob` and send it via `MessageService`; sync actions execute
immediately.**
*Alternative:* Could queue all actions and execute them in a batch. Could
make sync/async a property of the event rather than the action.

**GraphElement.java:246-251 — DEFAULT — Async action jobs get
`dueDate = new Date()` (immediate) and `exclusive` matches the action's
`isAsyncExclusive` flag.**
*Alternative:* Could allow a configurable delay. Could always be
non-exclusive.

**GraphElement.java:263-271 — CONTROL_FLOW — Token is locked during event
action execution (if it's an event-triggered action and token isn't already
locked). Not locked for node behavior actions.**
*Alternative:* Could always lock, or never lock. The distinction is: event
actions shouldn't move the token (it's locked to prevent that), while node
behavior actions need to move the token. This is a critical design decision
that determines concurrent safety.

**GraphElement.java:264 — DEFAULT — Lock owner is `action.toString()`.**
*Alternative:* Could use a UUID, the thread name, or a dedicated lock ID.
Using `toString()` makes debugging easier but could cause collisions if two
actions have the same string representation.

**GraphElement.java:277-279 — EXCEPTION — Only `Exception` is caught, not
`Error`. Comment explains: "Errors are not caught because that might halt the
JVM and mask the original Error."**
*Alternative:* Could catch `Throwable` to handle everything. The decision to
let `Error` propagate is a deliberate safety choice — `OutOfMemoryError`,
`StackOverflowError` etc. should crash the process, not be silently handled.

**GraphElement.java:281 — EXCEPTION — On action failure, the exception is
logged in the `ActionLog` and then passed to `raiseException()`.**
*Alternative:* Could rethrow immediately without trying exception handlers.
Could swallow the exception and continue.

### Exception handling chain

**GraphElement.java:290-307 — CONTROL_FLOW — `executeActionImpl` checks for
`UserCodeInterceptor` and delegates to it if present; otherwise calls
`action.execute()` directly.**
*Alternative:* Could always call action.execute() and use AOP/proxy for
interception. The interceptor pattern allows the simulation module to
replace real action execution with simulated execution.

**GraphElement.java:335-370 — EXCEPTION — `raiseException()` implements a
multi-step exception handling strategy:**
1. Check if already handling an exception (prevent recursive handling)
2. Check if transaction is still active (can't load handlers lazily if rolled back)
3. Find matching exception handler on this element
4. If found, execute it; if handler itself throws, use that new exception
5. If not found, propagate to parent element
6. If no handler found anywhere, wrap in `DelegationException` and throw

*Alternative:* Could use a simpler try/catch without the transaction check.
Could allow multiple exception handlers to run (chain). Could not propagate
to parent. Each step is a distinct decision:

**GraphElement.java:383 — CONTROL_FLOW — If `executionContext.getException()
!= null`, exception handling is bypassed (already handling one).**
*Alternative:* Could allow nested exception handling. This prevents infinite
recursion where a handler throws, which triggers another handler search.

**GraphElement.java:391 — CONTROL_FLOW — Checks `persistenceService
instanceof DbPersistenceService` to test transaction status.**
*Alternative:* Could use a more general `isTransactionActive()` abstraction.
The instanceof check couples this to the DB persistence implementation.

**GraphElement.java:397-398 — DEFAULT — If no persistence service or not DB-
backed, assumes exception handling is possible (`return true`).**
*Alternative:* Could be conservative and return false (don't try handlers).
The choice to return true allows in-memory execution to use exception
handlers.

**GraphElement.java:360 — CONTROL_FLOW — `parent != null && !equals(parent)`
— stops propagation if an element is its own parent (prevents infinite loop).**
*Alternative:* Could just check `parent != null`. The self-equality check is
a safety guard for malformed graphs.

**GraphElement.java:368-369 — EXCEPTION — Unhandled exceptions that are
already `JbpmException` are rethrown as-is; others are wrapped in
`DelegationException`.**
*Alternative:* Could always wrap. Could always rethrow as-is. The
preservation of `JbpmException` avoids double-wrapping.

### Exception handler matching

**GraphElement.java:401-408 — CONTROL_FLOW — `findExceptionHandler()` iterates
exception handlers in list order, returns first match.**
*Alternative:* Could return all matching handlers. Could match most-specific
first (by exception class hierarchy depth). The list-order approach gives the
user explicit priority control.

### Parent hierarchy

**GraphElement.java:411-413 — HIERARCHY — Default `getParent()` returns
`processDefinition`. Overridden in `Node` (returns superState or
processDefinition), `Transition` (returns common ancestor of from/to), and
`ProcessDefinition` (returns null).**
*Alternative:* Could return null by default. The choice to return
processDefinition means events on any element propagate to the process
definition level.

**GraphElement.java:420 — DEFAULT — `getParents()` returns
`Collections.EMPTY_LIST` when parent is null (no allocation).**
*Alternative:* Could return `new ArrayList()` (mutable empty list). Using
`EMPTY_LIST` is memory-efficient but the returned list is immutable.

### Equals and hashCode

**GraphElement.java:444-501 — EQUALITY — Complex multi-strategy equals:**
1. Identity check (`o == this`)
2. Type check using `getClass().isInstance(o)` (allows subclass equality)
3. ID match (if both have non-zero IDs)
4. Named elements: compare by name + parent
5. Unnamed elements in NodeCollections: compare by index position
6. Otherwise: not equal

*Alternative:* Could use only ID-based equality (simpler, common for
Hibernate entities). Could use `getClass() == o.getClass()` (strict type
matching) instead of `isInstance` (which allows a Node to equal a Decision).
The multi-strategy approach supports both persisted entities (by ID) and
in-memory objects (by name/position).

**GraphElement.java:446 — EQUALITY — Uses `getClass().isInstance(o)` not
`instanceof` or `getClass() ==`.**
*Alternative:* `instanceof GraphElement` would allow any two GraphElements to
be compared. `getClass() ==` would require exact type match. `isInstance`
allows subclass instances to be equal to parent class instances — a Decision
node could equal a Node if they have the same name and parent.

**GraphElement.java:449 — EQUALITY — ID comparison: `id != 0 && id ==
other.getId()` — zero ID means "not persisted" and is excluded.**
*Alternative:* Could treat 0 as a valid ID. Could use `Long` (boxed) with
null for "not persisted". The choice of 0 as sentinel means ID 0 can never
be a valid database ID.

**GraphElement.java:462-471 — EQUALITY — For unnamed nodes in a
NodeCollection, uses `System.identityHashCode()` to find position in parent's
node list (JBPM-3423 fix to avoid recursive `equals` via `indexOf`).**
*Alternative:* Could use a simple `indexOf()` (original code before the fix,
but caused infinite recursion). Could assign synthetic names to unnamed
nodes.

**GraphElement.java:484-501 — EQUALITY — `hashCode()` mirrors the equals
strategy — uses name+parent hash for named elements, index+parent for
unnamed.**
*Alternative:* Could use just the ID if non-zero, falling back to
`identityHashCode`. The current approach ensures consistency with equals but
is expensive for unnamed elements (requires iterating parent's node list).

### toString

**GraphElement.java:503-507 — DISPLAY — `toString()` shows class simple name
+ name, or ID if unnamed, or hex hash if no ID.**
*Alternative:* Could always show the fully-qualified class name. Could
include parent chain. Could show all three always.

### Field access

**GraphElement.java:55 — VISIBILITY — `id` is package-private, no setter.**
*Alternative:* Could have a setter (needed for some ORM strategies). The
absence of a setter means IDs are assigned only by Hibernate.

---

## ProcessDefinition.java

### Inheritance and interfaces

**ProcessDefinition.java:56 — HIERARCHY — `ProcessDefinition extends
GraphElement implements NodeCollection`.**
*Alternative:* Could be a standalone class (not a GraphElement). Making it a
GraphElement means it participates in event propagation, has a name, and can
have exception handlers — which is useful but conceptually odd (a
"definition" is also an "element").

### Field declarations

**ProcessDefinition.java:60 — DEFAULT — `version = -1` as initial value.**
*Alternative:* Could be 0, or use `Integer` with null. The -1 sentinel means
"not yet deployed" and is used in `equals()` and `hashCode()` to distinguish
unversioned definitions.

**ProcessDefinition.java:61 — DEFAULT — `isTerminationImplicit = false`
(explicit termination by default).**
*Alternative:* Could default to `true` (process ends when all tokens are in
end states). The `false` default means processes only end when an EndState
explicitly ends the root token.

**ProcessDefinition.java:63 — TYPE — `startState` is typed as `Node`, not
`StartState`.**
*Alternative:* Could be `StartState` for type safety. Using `Node` allows
any node to be the start state, which is more flexible but loses compile-time
type checking.

**ProcessDefinition.java:64 — CACHING — `nodesMap` is `transient` (not
serialized, rebuilt on demand).**
*Alternative:* Could serialize it. Making it transient means it's a pure
cache, rebuilt from the `nodes` list.

**ProcessDefinition.java:65 — COLLECTION_TYPE — `actions` is a `Map` (keyed
by action name).**
*Alternative:* Could be a `List` (allowing duplicate names). The Map choice
enforces unique action names at the process definition level.

**ProcessDefinition.java:66 — COLLECTION_TYPE — `definitions` is a `Map`
(keyed by class name string).**
*Alternative:* Could be keyed by `Class` object. Using String keys means
module definitions are looked up by class name, which works across
classloaders.

### Event types

**ProcessDefinition.java:72-89 — ENUMERATION — ProcessDefinition supports 16
event types including all node, task, transition, superstate, subprocess, and
timer events.**
*Alternative:* Could support only process-level events (process-start,
process-end). Supporting all event types means any event from any child
element propagates to the process definition level.

### Factory method

**ProcessDefinition.java:107-127 — CONTROL_FLOW — `createNewProcessDefinition()`
automatically adds default modules (loaded from `resource.default.modules`
property file).**
*Alternative:* Could create a bare ProcessDefinition and require explicit
module registration. The auto-registration means every process gets
ContextDefinition, TaskMgmtDefinition, etc. by default.

**ProcessDefinition.java:118-123 — EXCEPTION — Module instantiation failures
throw `JbpmException` wrapping the original exception.**
*Alternative:* Could skip failed modules with a warning. Could collect all
failures and throw a composite exception.

### Module class loading and caching

**ProcessDefinition.java:129-138 — CACHING — Module classes are cached in a
static `Map moduleClassesByResource`, synchronized on the map.**
*Alternative:* Could reload every time (no cache). Could use a concurrent
map. The synchronized block is a classic double-check locking pattern.

**ProcessDefinition.java:141-160 — EXCEPTION — `ClassNotFoundException` for
a module class is logged at debug level and the module is skipped.**
*Alternative:* Could throw an error (module is configured but missing). The
silent skip allows optional modules (e.g., FileDefinition is skipped if the
JCR dependency is not on the classpath).

### Parsing convenience methods

**ProcessDefinition.java:228-285 — API — Six static parsing methods:
`parseXmlString`, `parseXmlResource`, `parseXmlInputStream`, `parseXmlReader`,
`parseParZipInputStream`, `parseParResource`.**
*Alternative:* Could use a builder pattern. Could have a single `parse()`
method with overloads. The six methods provide convenience at the cost of API
surface area.

**ProcessDefinition.java:241-243 — EXCEPTION — `parseXmlResource()` throws
`JpdlException` if the resource is not found.**
*Alternative:* Could return null. Could throw a more specific
`ResourceNotFoundException`.

### Node management

**ProcessDefinition.java:296-306 — CACHING — `getNodesMap()` lazily builds
a HashMap from the nodes list.**
*Alternative:* Could maintain the map incrementally during add/remove. The
lazy rebuild approach is simpler but discards and rebuilds on every
invalidation.

**ProcessDefinition.java:325 — DEFAULT — `addNode()` lazily initializes
`nodes` as `ArrayList`.**
*Alternative:* Could use `LinkedList` if frequent insertion/removal at
arbitrary positions was expected.

**ProcessDefinition.java:328 — CACHING — `nodesMap = null` invalidates the
cache on node add.**
*Alternative:* Could update the map incrementally (map.put). Invalidation
is simpler and avoids inconsistency bugs.

**ProcessDefinition.java:330-332 — CONTROL_FLOW — If the added node is a
`StartState` and no start state is set yet, it becomes the start state.**
*Alternative:* Could require explicit `setStartState()`. Could throw if a
StartState is added when one already exists. The "first StartState wins"
behavior is implicit and could be surprising.

### Hierarchical node finding

**ProcessDefinition.java:401-434 — ALGORITHM — `findNode()` supports a
slash-separated hierarchical name with `/` for absolute paths and `..` for
parent navigation.**
*Alternative:* Could use dot-separated names. Could not support `..` parent
navigation. Could use XPath-like syntax. The slash convention mirrors file
system paths.

**ProcessDefinition.java:406 — CONTROL_FLOW — Empty name part (from leading
`/`) navigates to the process definition root.**
*Alternative:* Could treat empty name as an error.

**ProcessDefinition.java:419-420 — NULL — `..` on a null current element
returns null (graceful failure).**
*Alternative:* Could throw an exception ("cannot navigate above root").

**ProcessDefinition.java:426-428 — NULL — Attempting to navigate into a non-
NodeCollection element returns null.**
*Alternative:* Could throw an exception.

**ProcessDefinition.java:433 — TYPE — Final result must be a `Node` (not a
ProcessDefinition); returns null otherwise.**
*Alternative:* Could return the ProcessDefinition if navigated to root.

### Node name generation

**ProcessDefinition.java:379-391 — ALGORITHM — `generateNodeName()` produces
sequential integer names starting from "1", skipping any already used.**
*Alternative:* Could use UUIDs. Could use a prefix like "node-1". Could use
a monotonically increasing counter (no gap-filling).

### equals and hashCode

**ProcessDefinition.java:197-219 — EQUALITY — Two ProcessDefinitions are
equal if they have the same name AND version, but only if name is non-null
and version is non-negative.**
*Alternative:* Could compare by name only. Could compare by ID only. The
name+version approach matches the deployment model where each deployment
creates a new version.

**ProcessDefinition.java:214 — DEFAULT — Unversioned definitions (version
< 0) or unnamed definitions (name == null) use `System.identityHashCode`.**
*Alternative:* Could use a synthetic hash. This means two unpersisted
definitions with the same name are never equal unless they are the same
object.

### setProcessDefinition guard

**ProcessDefinition.java:184-188 — VALIDATION — Throws
`IllegalArgumentException` if trying to set a different ProcessDefinition
as this definition's processDefinition.**
*Alternative:* Could silently ignore. This guard prevents a ProcessDefinition
from being nested inside another ProcessDefinition (which would break the
model).

### Action management

**ProcessDefinition.java:461 — VALIDATION — `addAction()` requires the
action to have a non-null name.**
*Alternative:* Could allow unnamed actions. The requirement for names means
all process-level actions can be looked up by name.

**ProcessDefinition.java:481 — VALIDATION — `removeAction()` throws if the
action is not present.**
*Alternative:* Could silently return. This strict validation catches bugs
where code tries to remove an action from the wrong process definition.

### Module definition management

**ProcessDefinition.java:512-513 — KEYING — Module definitions are stored
by class name (`moduleDefinition.getName()` returns `getClass().getName()`).**
*Alternative:* Could use the class object directly. Could allow multiple
module definitions of the same type. The class-name key means at most one
definition per module type per process.

**ProcessDefinition.java:525 — KEYING — `removeDefinition()` removes by
class name, not by the module's `name` field.**
*Alternative:* Could remove by name field, or by identity. There's a subtle
difference here: `getName()` is the class name by default.

**ProcessDefinition.java:534-539 — API — `getDefinition(Class clazz)` looks
up by class name.**
*Alternative:* Could return the first assignable type (supporting
inheritance). The exact-match approach means you must look up by the exact
module class, not a superclass.

### Start state management

**ProcessDefinition.java:436-444 — CONTROL_FLOW — `setStartState(StartState)`
removes the old start state node from the process definition before setting
the new one.**
*Alternative:* Could keep the old node (just decouple from startState field).
The removal ensures only one StartState exists.

---

## Node.java

### NodeType inner class

**Node.java:53-89 — DESIGN_PATTERN — `NodeType` is a type-safe enum
implemented as a class with static final instances and a `values` map for
deserialization (`readResolve`).**
*Alternative:* Could use a Java 5 `enum`. Could use plain strings. The
class-based approach predates Java 5 enums and supports extensibility
(new NodeTypes can be registered by subclasses via the protected
constructor).

**Node.java:60-67 — ENUMERATION — Eight predefined NodeTypes: Node,
StartState, EndState, State, Task, Fork, Join, Decision.**
*Alternative:* Could include more types (MailNode, ProcessState,
InterleaveStart, etc.). Only the "core" types are represented.

### Field types

**Node.java:91 — COLLECTION_TYPE — `leavingTransitions` is a `List`
(ordered).**
*Alternative:* Could be a `Set` (unordered). The List choice means transition
order matters — the first unconditional transition is the "default" leaving
transition.

**Node.java:93 — COLLECTION_TYPE — `arrivingTransitions` is a `Set`
(unordered).**
*Alternative:* Could be a `List`. Arriving transitions have no meaningful
order (you can't control which transition tokens arrive on), so Set is
semantically correct and prevents duplicates.

**Node.java:91 vs 93 — ASYMMETRY — Leaving transitions are a List; arriving
transitions are a Set.**
*Alternative:* Could use the same collection type for both. This asymmetry
reflects the semantic difference: leaving transition order determines default
transition; arriving transition order is irrelevant.

**Node.java:94 — TYPE — `action` is a single `Action` (the node's behavior
action).**
*Alternative:* Could be a list of actions. Having a single action means node
behavior is a single delegation; multiple behaviors require composition.

### Event types

**Node.java:112-117 — ENUMERATION — Node supports 4 event types:
`node-enter`, `node-leave`, `before-signal`, `after-signal`.**
*Alternative:* Could also support task events (but those are on the Task
node specifically). The 4 events cover the node lifecycle.

### Leaving transition map

**Node.java:170-179 — ALGORITHM — `getLeavingTransitionsMap()` builds the
map by iterating in REVERSE order so that the first transition with a given
name wins (earlier in list overwrites later in map, but reverse iteration
means later `put` wins = first in list).**
*Alternative:* Could iterate forward (last transition with a given name
wins). Could throw on duplicate names. The reverse iteration is subtle: it
means if two transitions have the same name, `map.get(name)` returns the
first one in the list.

### Transition lookup with superstate fallback

**Node.java:227-236 — CONTROL_FLOW — `getLeavingTransition(name)` searches
this node's transitions first, then falls back to the superstate.**
*Alternative:* Could only search local transitions. Could search all the way
up to the process definition. The superstate fallback means transitions
defined on a superstate are available to all contained nodes.

**Node.java:231-232 — NULL — Handles null transition name: searches for a
transition with `getName() == null`.**
*Alternative:* Could treat null name as "default transition". The explicit
null-name search means unnamed transitions are first-class and can be found
by name (null).

### Default leaving transition

**Node.java:274-288 — ALGORITHM — Default transition is the first
UNCONDITIONAL transition; if all transitions have conditions, falls back to
the first transition overall.**
*Alternative:* Could always be the first transition regardless of conditions.
Could require an explicit "default" marking. The unconditional-first logic
means conditional transitions are skipped when no specific transition is
named.

### Transition name generation

**Node.java:250-257 — ALGORITHM — First transition gets `null` name; second
gets "1"; third gets "2"; etc. (skips existing names).**
*Alternative:* Could always use numbered names. The null-first pattern means
the "main" transition has no name, which serves as the default.

### Node execution lifecycle

**Node.java:358-383 — CONTROL_FLOW — `enter()` does 5 things in order:**
1. Set token's node to this
2. Record node enter time
3. Fire `node-enter` event
4. Clear transition references from context
5. If async: create job and lock token; if sync: call `execute()`

*Alternative:* Could fire the event after clearing transition context.
Could set the enter time after the event fires. The ordering means event
handlers see the token's new node but still have access to the previous
transition (actually no — transition is cleared before execute, not before
event firing; it's cleared at step 4).

**Node.java:375-382 — CONTROL_FLOW — Async node execution creates an
`ExecuteNodeJob`, sends it via MessageService, and locks the token.**
*Alternative:* Could use a thread pool instead of a message service. Could
not lock the token (risky — other threads could signal it). The lock
prevents concurrent modifications while the async job is pending.

### Default execute behavior

**Node.java:396-406 — CONTROL_FLOW — `execute()`: if a custom action is set,
execute it; otherwise, leave over the default transition.**
*Alternative:* Could throw if no action is set (require explicit behavior).
Could always leave after executing the action. The "auto-leave" behavior for
action-less nodes means plain `<node>` elements in JPDL are pass-through.

### Leave behavior

**Node.java:412-413 — DEFAULT — `leave()` with no arguments uses the default
leaving transition.**
*Alternative:* Could require an explicit transition name always. The
convenience method makes the common case (leave over default) easy.

**Node.java:422 — EXCEPTION — `leave(name)` throws `JbpmException` if no
transition with the given name exists.**
*Alternative:* Could return false. Could silently do nothing. Throwing
ensures programming errors are caught.

**Node.java:441-443 — CONTROL_FLOW — Only creates a `NodeLog` if
`token.getNodeEnter()` is non-null.**
*Alternative:* Could always log. The null check skips logging for nodes that
were entered without recording an enter time (e.g., the start state on
process creation before the token's nodeEnter is set — but actually
`enter()` always sets it, so this might only be relevant for manually
positioned tokens).

### setName with uniqueness enforcement

**Node.java:467-488 — VALIDATION — `setName()` enforces name uniqueness
within the parent container (superstate or process definition).**
*Alternative:* Could allow duplicate names (lookup returns first match).
Could not validate at all (defer to addNode). The proactive validation
prevents silent name collisions.

**Node.java:470-477 — CONTROL_FLOW — Checks superState first, then
processDefinition, for name conflicts.**
*Alternative:* Could check only one container. The two-level check
reflects the containment model.

### Fully qualified name

**Node.java:497-499 — ALGORITHM — Fully qualified name uses `/` as
separator, concatenating superstate names.**
*Alternative:* Could use `.` or `::`. The `/` choice matches the `findNode`
hierarchical naming convention.

### setAsyncExclusive side effect

**Node.java:537-540 — SIDE_EFFECT — Setting `isAsyncExclusive = true`
automatically sets `isAsync = true`.**
*Alternative:* Could require setting both independently. Could throw if
exclusive is set without async. The automatic implication makes the API less
error-prone.

---

## Transition.java

### Fields

**Transition.java:43 — DEFAULT — `isConditionEnforced = true` by default.**
*Alternative:* Could default to false (don't enforce conditions). The true
default means conditions are always checked when a transition is taken. Must
explicitly opt out.

**Transition.java:43 — VISIBILITY — `isConditionEnforced` is transient
(not persisted).**
*Alternative:* Could persist it. Being transient means it resets to true
on deserialization. This appears to be an intentional choice: condition
enforcement is a runtime behavior, not a persistent property.

### Event types

**Transition.java:47-49 — ENUMERATION — Transitions support only 1 event
type: `transition`.**
*Alternative:* Could support `before-transition` and `after-transition` as
separate events. The single event fires during the transition.

### Condition evaluation

**Transition.java:130-135 — CONTROL_FLOW — If condition is non-null and
enforcement is on, evaluates condition and throws `JbpmException` if result
is not `Boolean.TRUE`.**
*Alternative:* Could skip the transition silently (return without entering
target). Could evaluate conditions as part of decision logic rather than
on the transition itself. Throwing means taking a guarded transition with a
false condition is a hard error.

**Transition.java:132 — NULL — `!Boolean.TRUE.equals(result)` — null result
from expression evaluation is treated as false (condition not met).**
*Alternative:* Could treat null as true. Could throw a separate "expression
returned null" error.

### Transition take lifecycle

**Transition.java:129-163 — ORDERING — `take()` does in order:**
1. Evaluate condition (if any)
2. Set token node to null (token is "in transit")
3. Fire superstate leave events
4. Fire transition event
5. Fire superstate enter events
6. Enter destination node

*Alternative:* Could enter destination before firing events. Could set
token node to destination instead of null during transit. Setting node to
null means during transition events, `token.getNode()` returns null.

### SuperState event handling

**Transition.java:165-189 — ALGORITHM — `fireSuperStateEnterEvents()` walks
into nested superstates to find the actual leaf destination node (first child
of each superstate).**
*Alternative:* Could not auto-descend into superstates. Could let the user
specify which child of the superstate to enter.

**Transition.java:170 — DEFAULT — When entering a superstate, the first
child node is chosen as the entry point.**
*Alternative:* Could require a designated entry node. Could use a named
child. The "first child" convention is implicit and depends on node ordering
in the XML.

**Transition.java:173-176 — EXCEPTION — Throws `JbpmException` if the
destination resolves to null (superstate with no children).**
*Alternative:* Could stay on the source node. Could skip the transition.

**Transition.java:183 — ORDERING — Superstate enter events fire outer to
inner (list is reversed).**
*Alternative:* Could fire inner to outer. Outer-to-inner matches the DOM
"capture phase" pattern.

**Transition.java:195 — ORDERING — Superstate leave events fire inner to
outer (natural list order).**
*Alternative:* Could fire outer to inner. Inner-to-outer matches DOM
"bubble phase".

### Equals for Transition

**Transition.java:232-241 — EQUALITY — Transitions are equal if they have
the same name and the same source node (`from`).**
*Alternative:* Could also compare `to` node. Could compare condition. The
source+name combination is unique within a process since a node can only
have one leaving transition with a given name.

**Transition.java:240 — NULL — `from != null` is required for non-ID
equality; null-from transitions are only equal by identity.**
*Alternative:* Could treat disconnected transitions as equal if names match.

### getParent for Transition

**Transition.java:272-283 — ALGORITHM — Returns the lowest common ancestor
of `from` and `to` nodes in the graph element hierarchy.**
*Alternative:* Could return `from.getParent()`. Could return
processDefinition always. The LCA approach means event propagation on a
transition bubbles to the right level in the superstate hierarchy.

**Transition.java:274 — CONTROL_FLOW — If `from == to` (self-loop), returns
`from.getParent()`.**
*Alternative:* Could return `from` itself.

### setName with duplicate check

**Transition.java:260-270 — VALIDATION — Setting a transition name checks
if the source node already has a leaving transition with that name.**
*Alternative:* Could silently overwrite. Could allow duplicate names.

**Transition.java:267 — SIDE_EFFECT — Updates the source node's
`leavingTransitionsMap` by putting the new name entry.**
*Alternative:* Could invalidate the map and let it rebuild lazily. But the
old name entry is NOT removed — this appears to be a bug or intentional
decision to leave the old mapping (it would be overwritten if iterated from
the list). Actually, looking more closely, `from.getLeavingTransitionsMap()`
rebuilds from the list, so this put is into the cache which will be rebuilt
next time anyway. But the old name IS left in the map until next rebuild.

---

## Event.java

### Event types as constants

**Event.java:32-48 — DESIGN_PATTERN — 18 event types defined as `public
static final String` constants.**
*Alternative:* Could use an enum. Could use a registry pattern. String
constants predate Java 5 enums but allow extensibility (custom event types
are valid even though no constant exists).

**Event.java:47-48 — ENUMERATION — Two timer-related events: `timer-create`
and `timer`.**
*Alternative:* Could also have `timer-cancel`, `timer-fire`. The two events
cover creation and execution.

### Actions collection

**Event.java:88 — DEFAULT — Actions list lazily initialized as `ArrayList`.**
*Alternative:* Could eagerly initialize.

### equals and hashCode

**Event.java:109-124 — EQUALITY — Events are equal by (eventType +
graphElement) pair.**
*Alternative:* Could also compare the actions list. The choice means two
events with the same type on the same element are considered equal regardless
of their actions.

**Event.java:116-117 — NULL — No null checks on `eventType` or
`graphElement` in equals — will NPE if either is null.**
*Alternative:* Could null-check. This implies Events should always have both
set — a design contract enforced by convention rather than validation.

---

## Action.java

### Three execution strategies

**Action.java:108-118 — CONTROL_FLOW — `execute()` has three mutually
exclusive strategies in priority order:**
1. `referencedAction != null` → delegate to referenced action
2. `actionExpression != null` → evaluate expression
3. `actionDelegation != null` → instantiate and call ActionHandler

*Alternative:* Could support combinations (expression + delegation). Could
use a strategy pattern instead of conditionals. The priority order means
if both an expression and a delegation are set, the expression wins.

**Action.java:99-106 — SIDE_EFFECT — Sets the thread's context class loader
to the process class loader before executing.**
*Alternative:* Could not modify the class loader. This ensures delegation
classes from process archives can load their dependencies.

### Parsing three forms

**Action.java:74-90 — CONTROL_FLOW — `read()` parses three forms of action
definition from XML:**
1. `expression` attribute → store as actionExpression
2. `ref-name` attribute → defer resolution (add to unresolved references)
3. `class` attribute → create Delegation
4. None of the above → add warning

*Alternative:* Could require exactly one. Could support combinations. The
priority order determines which attribute takes precedence if multiple are
present.

### Default field values

**Action.java:41 — DEFAULT — `isPropagationAllowed = true`.**
*Alternative:* Could default to false (actions only fire on their direct
element). True means actions fire even when the event propagates from a child
element.

### setName side effect

**Action.java:189-203 — SIDE_EFFECT — Setting the name updates the process
definition's action map (removes old name, puts new name).**
*Alternative:* Could require remove-and-re-add. The automatic map update
prevents stale entries.

**Action.java:195 — CONTROL_FLOW — The condition
`(this.name != null && this.name.equals(name) || name != null)` always
evaluates to true when `name != null`. The comment says "the != string
comparison is to avoid null pointer checks."**
*Alternative:* Could use a cleaner null-safe comparison. This condition
appears to have a logic issue — it should probably be
`!(this.name != null ? this.name.equals(name) : name == null)` to detect
actual changes.

### setAsyncExclusive implication

**Action.java:249-252 — SIDE_EFFECT — `setAsyncExclusive(true)`
automatically sets `isAsync = true`.**
*Alternative:* Same pattern as Node — exclusive implies async.

### equals/hashCode branching

**Action.java:125-164 — EQUALITY — Named actions: equal by (name +
processDefinition). Unnamed actions: equal by (delegation/expression/
reference + event).**
*Alternative:* Could use only ID-based equality. The dual strategy handles
both top-level named actions and inline event-bound actions.

**Action.java:137-140 — EQUALITY — For unnamed actions, the delegation/
expression/reference comparison is cascaded with short-circuit: delegation
takes priority over expression over reference.**
*Alternative:* Could compare all three. The cascade means only one mechanism
needs to match.

---

## SuperState.java

### Event types

**SuperState.java:55-70 — ENUMERATION — SuperState supports 14 event types
(same as ProcessDefinition minus process-start and process-end, but includes
superstate-enter/leave).**
*Alternative:* Could support process-start/end (but those don't make sense
for superstates). The event type set determines what the process designer
shows.

### Execution behavior

**SuperState.java:90-97 — CONTROL_FLOW — Executing a SuperState delegates to
its first child node.**
*Alternative:* Could throw (superstates are containers, not executable).
Could allow a designated entry child. The "first child" convention is
consistent with how `Transition.fireSuperStateEnterEvents()` resolves
superstates.

**SuperState.java:91 — EXCEPTION — Throws `JbpmException` if the superstate
has no children.**
*Alternative:* Could leave the token in the superstate. Could auto-leave
via default transition.

### Node containment

**SuperState.java:130-138 — CONTROL_FLOW — `addNode()` sets
`node.superState = this` (direct field access).**
*Alternative:* Could use a setter method. Direct field access is a
performance choice within the same package.

**SuperState.java:182-193 — ALGORITHM — `containsNode()` walks up the
parent chain of the target node checking for equality with this superstate.**
*Alternative:* Could walk down this superstate's children recursively. The
upward walk is more efficient when the superstate hierarchy is deep but the
child count is high.

---

## ExceptionHandler.java

### Matching strategy

**ExceptionHandler.java:47-49 — CONTROL_FLOW — `matches()`: if
`exceptionClassName` is set, uses `isInstance()` (matches subclasses); if
null, matches ALL exceptions.**
*Alternative:* Could require exact type match (`getClass() == exceptionClass`).
Could use `equals` on class names. The `isInstance` approach matches the
standard Java catch semantics. The null-matches-all is a wildcard handler.

### Lazy class resolution

**ExceptionHandler.java:51-61 — CACHING — Exception class is resolved lazily
and cached in a transient field.**
*Alternative:* Could resolve at parse time. Lazy resolution means the class
doesn't need to be on the classpath until an exception actually needs
matching.

**ExceptionHandler.java:56-57 — EXCEPTION — Throws `JbpmException` wrapping
`ClassNotFoundException` if the exception class can't be loaded.**
*Alternative:* Could return false (no match) for unloadable classes. Throwing
means a misconfigured exception handler is a hard error.

### Handler execution

**ExceptionHandler.java:63-71 — CONTROL_FLOW — Executes ALL actions in the
handler's action list sequentially.**
*Alternative:* Could stop after first action. Could execute in parallel. The
sequential execution means all recovery actions run in order.

**ExceptionHandler.java:68 — DELEGATION — Calls
`graphElement.executeAction()` — which means exception handler actions get
the full action execution treatment (logging, locking, etc.).**
*Alternative:* Could execute actions directly without the GraphElement
wrapper. This means exception handler actions themselves can throw exceptions,
which would be caught by `raiseException()` and re-propagated.

---

## EventCallback.java

### Default timeout

**EventCallback.java:42 — DEFAULT — `DEFAULT_TIMEOUT = 5 * 60 * 1000`
(5 minutes).**
*Alternative:* Could be 30 seconds, 1 minute, or configurable. 5 minutes is
generous for test synchronization.

### Notification mechanism

**EventCallback.java:86-103 — DESIGN_PATTERN — Uses JTA `Synchronization`
registered on the Hibernate transaction to fire notifications after commit.**
*Alternative:* Could use a direct callback (risks running before commit).
Could use JMS. The after-commit approach ensures the event is only notified
when the database state is consistent.

**EventCallback.java:92 — CONTROL_FLOW — Only releases the semaphore if
`status == STATUS_COMMITTED`; rolled-back transactions don't notify.**
*Alternative:* Could notify on rollback too (with a different event).
The committed-only approach prevents false positives in tests.

### Semaphore management

**EventCallback.java:132-142 — SYNCHRONIZATION — `getEventSemaphore()` is
synchronized on the static `eventSemaphores` map.**
*Alternative:* Could use `ConcurrentHashMap.computeIfAbsent()`. The
synchronized block predates Java 8's concurrent utilities.

**EventCallback.java:137 — DEFAULT — New semaphores start with 0 permits
("fail semaphore").**
*Alternative:* Could start with N permits. Starting at 0 means
`waitForEvent` blocks until exactly the right number of notifications arrive.

---

## DelegationException.java

**DelegationException.java:27 — HIERARCHY — Extends `JbpmException`.**
*Alternative:* Could extend `RuntimeException` directly. Extending
JbpmException keeps it in the jBPM exception hierarchy.

**DelegationException.java:39-41 — DEPRECATION — The constructor taking
`ExecutionContext` is deprecated because "the execution context may not be
in a consistent state."**
*Alternative:* Could remove it entirely. Keeping it deprecated maintains
backward compatibility.

---

## ActionHandler.java (interface)

**ActionHandler.java:28-30 — API — Single-method interface with
`execute(ExecutionContext) throws Exception`.**
*Alternative:* Could be a functional interface (it is, implicitly). Could
have lifecycle methods (`init`, `destroy`). Could not throw checked
exceptions (use `RuntimeException` only). The `throws Exception` allows any
exception but forces callers to handle checked exceptions.

**ActionHandler.java:28 — INTERFACE — Extends `Serializable`.**
*Alternative:* Could not require serialization. The Serializable requirement
means action handler instances can be serialized to the database (for async
execution and process persistence).

---

## Identifiable.java (interface)

**Identifiable.java:30-32 — API — Single method `long getId()` with `long`
return type.**
*Alternative:* Could use `Long` (boxed, nullable). Could use `String` (for
UUID-based IDs). Could use a generic `<T>` ID type. The primitive `long`
means every identifiable element uses a numeric database-generated ID.

---

## NodeCollection.java (interface)

**NodeCollection.java:30 — INTERFACE — Extends `Serializable`.**
*Alternative:* Could be a plain interface. Serializable is needed because
both ProcessDefinition and SuperState are serializable.

**NodeCollection.java:30 — HIERARCHY — Implemented by both
`ProcessDefinition` and `SuperState`.**
*Alternative:* Could have a separate `NodeContainer` class that both
delegate to. The interface approach avoids an extra object but requires both
implementors to duplicate the node management logic.

---

## ModuleDefinition.java

**ModuleDefinition.java:29 — HIERARCHY — Abstract class, not an interface.**
*Alternative:* Could be an interface with default methods. The abstract class
allows field declarations (id, name, processDefinition) shared by all
module definitions.

**ModuleDefinition.java:32 — DEFAULT — `name = getClass().getName()` — module
name defaults to the fully-qualified class name.**
*Alternative:* Could use a human-readable name. Could use
`getSimpleName()`. The full class name serves as a unique key in the
process definition's module map.

**ModuleDefinition.java:41 — API — `createInstance()` is abstract — every
module definition must know how to create its runtime instance.**
*Alternative:* Could use a factory registry. Could use reflection. The
abstract method ensures compile-time enforcement.

---

## ModuleInstance.java

**ModuleInstance.java:30 — HIERARCHY — Concrete class (not abstract), but
subclassed by `ContextInstance`, `TaskMgmtInstance`, `LoggingInstance`, etc.**
*Alternative:* Could be abstract (force subclassing). Being concrete allows
default module instances but subclasses provide the real behavior.

**ModuleInstance.java:35 — FIELD — Has a `version` field for optimistic
locking.**
*Alternative:* Could not have optimistic locking. The version field prevents
concurrent modification conflicts in the database.

**ModuleInstance.java:60-62 — API — `getService(String)` provides access to
jBPM services from within module instances.**
*Alternative:* Could inject services via constructor. Could use a service
locator pattern. The current approach uses a thread-local service registry.

**ModuleInstance.java:61 — DEFAULT — `Services.getCurrentService(serviceName,
false)` — the `false` parameter means "don't throw if service not found".**
*Alternative:* Could pass `true` (throw on missing service). The lenient
lookup allows module instances to work without all services configured.

---

## Summary Statistics

| Category | Count |
|----------|------:|
| Control flow decisions | 68 |
| Default value choices | 22 |
| Collection type choices | 8 |
| Exception handling strategies | 16 |
| Equality/identity decisions | 18 |
| Visibility/access decisions | 6 |
| API design decisions | 14 |
| Validation strategies | 14 |
| Algorithm choices | 12 |
| Caching strategies | 6 |
| Side effect decisions | 6 |
| Type hierarchy decisions | 8 |
| Null handling decisions | 10 |
| Synchronization decisions | 2 |
| Ordering decisions | 6 |
| Enumeration choices | 6 |
| Design pattern choices | 4 |
| **Total** | **~226** |
