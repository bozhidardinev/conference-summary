### [Know your Java? - Venkat Subramaniam](https://www.youtube.com/watch?v=v5Q7TC5u5Co&list=PLX8CzqL3ArzVV1xRJkRbcM2tOgVwytJAi&index=4&pp=iAQB)

### 1. **Side Effects in Stream Operations:** 
A significant pitfall is performing side effects within stream operations, particularly terminal operations like forEach. This often involves modifying a collection or state that exists outside the stream pipeline. While this might appear to work with sequential streams, it becomes a "ticking time bomb" when using parallel streams because the operations are not guaranteed to execute in a predictable order or on a single thread. This can lead to non-deterministic behavior and incorrect results, such as losing elements when adding to a collection.

- Key Concept: Pure Functions: Lambdas used in streams, especially intermediate operations, should ideally be pure functions. A pure function has two main qualities:
	1. It does not modify anything outside its scope or make internal mutations visible externally.
	2. It does not depend on anything outside that may possibly change.


#### 1.1. Lazy Evaluation and Dependencies on Mutable External State: 
Stream operations employ lazy evaluation. This means intermediate operations (like map) are not executed when the stream pipeline is set up, but only when a terminal operation is called. If a lambda within a stream depends on external mutable state (like a value in a map), and that state changes before the terminal operation is executed, the lambda will use the changed value when it finally runs.

- Lesson: This violates the second rule of pure functions – depending on something outside that can change.

- Why Purity and Immutability Matter: Functional programming relies on lazy evaluation and parallel execution for efficiency, and these mechanisms rely on the purity of functions and immutability for correctness. Code that produces incorrect results, regardless of how well-written it appears, is not useful.

### 2. Type Inference (var) vs. Intended Type:
While type inference (var) is a useful feature, it infers the type based on the right-hand side of the assignment. This can lead to the variable being typed as a more specific type (e.g., ArrayList) than what was intended (e.g., Collection or List). Using var for a collection initialized as new ArrayList<>() will result in the variable being typed as ArrayList, not Collection or List. This can subtly change behavior, such as which overloaded method is called (e.g., remove(int) on ArrayList vs. remove(Object) potentially resolved via Collection or List interfaces depending on context).

Lesson: Be cautious when using var for collections, especially when you intend for the variable to be of a more general interface type (List, Collection, etc.) rather than a specific implementation type (ArrayList). Check the inferred type (e.g., using an IDE) and avoid var if it's not the desired type.

### 3. Ambiguity with remove on Collections vs. Lists: 
 When using an ArrayList, calling remove(1) typically removes the element at index. However, if the variable is declared as a Collection, calling remove(1) (with 1) can, in certain cases like a ```
```java
 Collection<Integer>
 ```
, trigger autoboxing of 1 into an Integer object. This means the method called is remove(Object obj), which removes the first occurrence of the object with the value 1, not the element at index. This difference led to surprising results in an example, removing the value 1 rather than the element at index.
 
Lesson: API design can be difficult, and overloading (like having remove(int index) and remove(Object obj)) can lead to confusion, especially with autoboxing and type differences due to inheritance. The code does what you type, not necessarily what you mean.


[Stream Gatherers (Java 24 Feature) - Deep Dive with the Expert - Viktor Klang - Software Architect (Java Platform Group - Oracle)](https://www.youtube.com/watch?v=J1YH_GsS-e0&list=PLX8CzqL3ArzVV1xRJkRbcM2tOgVwytJAi&index=7&ab_channel=Java)

The talk introduces **Stream Gatherers**, a new concept added to the Java Streams API to address limitations with existing intermediate operations and the `Collector` interface. The goal is to enable developers to create custom, flexible intermediate stream operations.

Stream Anatomy:
![[JavaOne25/Deeper into Java/screenshots/1.png]]


**Background: Java Stream Pipelines**

A typical Java Stream pipeline consists of a **source** of elements, zero or more **intermediate operations**, and a single **terminal operation**. Intermediate operations transform or filter the stream and are usually lazy, while the terminal operation triggers the processing and produces a final result or side effect. Existing intermediate operations include `filter`, `map`, `flatMap`, etc..

**The Problem with Existing Operations and Collectors**

There was a need for intermediate operations that could do more complex things, such as remembering previous elements, performing windowing (grouping elements into sub-lists), or calculating running aggregations (scan operations).

While the **`Collector`** interface is used for **terminal operations** to accumulate elements into a result (like collecting to a list or summing), it has significant limitations when considered for _intermediate_ operations:

- It's an **N to 1 operation**, meaning it consumes many elements and produces a single result, which doesn't fit the many-to-many nature of intermediate operations.
- It **cannot support infinite streams** because it must consume all elements before producing a result.
- It **cannot end early** or be "frugal" by deciding it doesn't need more upstream elements (like `limit` or `takeWhile`).
- Handling **inherently sequential operations** (like windowing based on encounter order) correctly in a parallel `Collector` (which requires a `combiner` for arbitrary merging) is not possible.
- The `accumulator` in a `Collector` has nowhere to send intermediate results; it can only update internal state.

**Introducing Stream Gatherers**

**Gatherers** are introduced as the solution to define custom **intermediate stream operations**. They were previewed in Java 22 and 23 and finalized in Java 24. The `Gatherer` interface is inspired by `Collector` but designed for intermediate steps.

Key components of the `Gatherer` interface:

- **Initializer**: A supplier to create the state for the gatherer.
- **Integrator**: Receives the current state, the next input element, and a handle to the **downstream**. This is the core processing logic for each element.
- **Combiner**: For parallel streams, defines how to merge the state from different processing branches.
- **Finisher**: Called once at the end of processing. It receives the final state and the **downstream** handle.

The crucial difference from `Collector`'s `accumulator` and `finisher` is the presence of a **`downstream` handle**. This handle has a **`push` method** that sends an element to the next operation in the pipeline and, importantly, **returns a boolean** indicating whether the downstream wants more elements. This return value enables **flow control** and **short-circuiting**; if the downstream returns `false`, the gatherer can propagate this signal upstream, potentially stopping further processing.

**Implementing Operations with Gatherers**

The talk demonstrates implementing various stream operations using `Gatherer`:

- **`map`**: A simple stateless gatherer that applies a function and pushes the result downstream.
- **`mapMulti`**: A stateless gatherer that can push zero or more elements per input using the downstream handle.
- **`limit`**: A stateful gatherer that keeps a count of elements processed and short-circuits by returning `false` from the integrator when the limit is reached. This also illustrates that `limit` is an **inherently sequential operation**.

Gatherers can be defined using inline lambda expressions via factory methods like `Gatherer.of(...)`, as method-local classes, or as separate classes for reusability.

**Gatherer Evaluation (Sequential and Parallel)**

The talk explains the abstract recipe for evaluating a gatherer:

1. Define the destination (downstream).
2. Create the initial state using the initializer.
3. Loop: While there are elements and the integrator (with state, element, and downstream) returns `true`, process the next element.
4. Call the finisher with the final state and downstream handle.

**Parallel evaluation** is more complex:

1. The stream source is **split recursively** into smaller chunks.
2. A sequential evaluation is run on each "leaf" chunk.
3. Partial results (states) are combined up the tree using the combiner.
4. The finisher is called once at the root of the tree.

**Short-circuiting in Parallel** is handled by cancelling tasks dealing with elements downstream in encounter order from where the short-circuit signal originated. If a left-hand side task short-circuits, the corresponding right-hand side task can potentially be discarded.

**Built-in Gatherers (`java.util.stream.Gatherers`)**

The standard library includes some useful out-of-the-box gatherers:

- **Folding**: Similar to reduce but can produce a different output type, maintaining state across elements.
- **Scanning**: Like folding, but emits every intermediate state/result.
- **Windowing**: Groups elements into lists of a fixed size (`windowFixed`) or based on other criteria. These are typically sequential due to their reliance on encounter order.
- **`mapConcurrent`**: Evaluates a function on elements concurrently using **virtual threads**, with a configurable limit on simultaneous tasks. It supports short-circuiting, interrupting virtual threads if the downstream no longer needs elements. This is currently not directly tied to the structured concurrency JEP.

**Gatherer Composition**

Gatherers can be composed using the **`andThen`** method, similar to functions. This allows chaining gatherers, where the output of one becomes the input of the next, enabling the construction of more complex intermediate operations from simpler ones.

**Conclusion**

Gatherers provide the necessary building blocks for implementing **any intermediate stream operation**, supporting statefulness, any-to-any input/output ratio, sequential or parallel execution control, composition, and crucial **frugal/short-circuiting** behavior. They fit into the stream pipeline model alongside sources (Spliterators) and terminal operations (Collectors).



[How Netflix Uses Java - Paul Bakker - Java Champion & Staff Software Engineer (Netflix)](https://www.youtube.com/watch?v=J1YH_GsS-e0&list=PLX8CzqL3ArzVV1xRJkRbcM2tOgVwytJAi&index=7&ab_channel=Java)

Netflix builds various types of applications, including the well-known **Netflix streaming backend** and more traditional **enterprise applications** for areas like movie production.

The **Netflix streaming backend** is characterized by extremely **high Requests Per Second (RPS)** due to millions of users. It is **multi-region**, operating in four different Amazon regions to provide low latency globally. This multi-region setup makes communication between regions expensive and slow, and complicates the use of relational data stores. When a request comes in, there is a **huge fan-out** to many different microservices. The system can often rely on **retries on failure** due to aggressive timeouts. It can also tolerate some failures, like missing non-critical data in a response. Relational data stores are typically not used in the streaming path; instead, in-memory distributed data stores like EV cache, Kafka, and Cassandra are more common.

The **traditional enterprise apps** (like those for movie production) are different. They are generally **low RPS** with fewer concurrent users. They can typically operate in a **single region**. Data often fits well into **relational databases**, although they aren't always used. Crucially, for these apps, **failure is often not an option**; data must be saved reliably, meaning the retry-on-failure mechanism used in streaming is not as applicable.

![[JavaOne25/Deeper into Java/screenshots/2.png]]

Despite these differences, both types of applications utilize a similar **GraphQL-based architecture**. Requests from devices or UIs first hit a **GraphQL API Gateway**. This gateway handles **federated GraphQL queries**, fanning out to various backend services, which Netflix calls **Domain Graph Services (DGS)**. These DGS services are primarily built using **Spring Boot** and the **DGS framework**, and are all Java. For **server-to-server communication**, Netflix often uses **gRPC**. gRPC is preferred for this purpose because it's a fast, binary protocol and aligns with a method-call mental model. For UI-to-backend communication, **GraphQL** is favored due to its flexible schema, schema-based collaboration between teams, and thinking-in-data model. The talk states that **REST is generally not used** for new development at Netflix due to a lack of schema, lack of flexibility, and often sending too much data.

![[JavaOne25/Deeper into Java/screenshots/3.png]]


Regarding **how Netflix uses Java**, the talk details their journey and technologies:

- They were previously stuck on **JDK 8** due to an old internal application framework and incompatible libraries.
- To break this cycle, they patched a handful of unmaintained libraries for JDK 17 compatibility.
- Over 2-3 years, they migrated about 3,000 applications from their old framework to **Spring Boot**, building tooling to assist in this large effort.
- Now, most services run on **JDK 17 or newer**, with high-RPS services often on JDK 20, 21, or 23.
- JDK upgrades provided significant benefits:
    - **JDK 17** showed **G1 garbage collector improvements**, resulting in about 20% less CPU time spent on GC for high-RPS services – a "free" performance win.
    - **JDK 21** introduced **generational ZGC**. The non-generational ZGC didn't work well for their services with large heaps and old data, because it has to go through all the heap space every time it does it's thing, that becomes slow.
    - Generational ZGC proved to be a good general-purpose GC, dramatically **reducing GC pause times** from over a second (with G1 on JDK 21) to virtually zero. This reduction in pause times also **reduced error rates** caused by timeouts during GC events, allowing services to run more consistently and at higher CPU loads.
    - **JDK 21 and beyond** brought **Virtual Threads** (Project Loom). They added virtual thread support to their frameworks (Spring Boot Netflix, DGS) so developers benefit automatically without changing code. Virtual threads allow operations that might block (like database calls or other service calls) to run concurrently by default, significantly reducing overall response times in scenarios like GraphQL data fetching. The low overhead of virtual threads allows them to be the default even for fast operations, improving developer experience compared to manually managing thread pools or CompletableFutures. Paul's "hot take" is that **virtual threads combined with structured concurrency will completely replace reactive programming**, a technology Netflix was heavily invested in (e.g., RX Java) but largely moved away from due to code and debugging complexity.
    - A significant issue was encountered with virtual threads in JDK 23: **deadlocks** could occur when virtual threads using `synchronized` or re-entrant locks were pinned to platform threads, and these pinned threads were waiting for locks held by other virtual threads that had no available platform threads to run.
    - This pinning issue was **fixed in JDK 24 by JEP 491**, which reimplemented how `synchronized` works with virtual threads. With this fix, Netflix is resuming its rollout of virtual threads.

- Applications are deployed either directly on AWS instances or their internal container platform (Titus, similar to Kubernetes) using exploded jar files with embedded Tomcat. They have experimented with **GraalVM Native Image** but found it currently too hard to get right and detrimental to developer experience; they are instead **betting on AOT/Project Leiden** for future startup time improvements.
- They are generally **avoiding reactive programming** (like WebFlux) because it requires an end-to-end reactive stack which is hard to achieve with existing libraries, and virtual threads/structured concurrency are seen as the better solution. They have standardized on WebMvc.
- The upgrade to **Spring Boot 3** presented a challenge due to the transition from `javax` to **`jakarta` namespaces**. While trivial for applications, it required libraries built against Spring Boot 2 (using `javax`) to work with Spring Boot 3 (expecting `jakarta`). Their solution is to use **Gradle transforms** to perform a bytecode rewrite from `javax` to `jakarta` at artifact resolution time. This solution is open sourced.
- The **DGS framework** for GraphQL was open sourced in 2020 and built on GraphQL Java. Netflix collaborated closely with the Spring team on **Spring for GraphQL**, and DGS now integrates and uses components from Spring for GraphQL under the hood.

In summary, Netflix heavily relies on Java for its backend services, utilizing a modern stack centered around **Spring Boot**, the **DGS framework**, and the latest **JDK features** like **generational ZGC** and **Virtual Threads**, while standardizing on **GraphQL** for UI-backend and **gRPC** for server-server communication and moving away from reactive programming and REST.


[# Preparing for the Java 21 Certification (or Learning New Features) - Jeanne Boyarsky (Java Champion & author)](https://www.youtube.com/watch?v=J1YH_GsS-e0&list=PLX8CzqL3ArzVV1xRJkRbcM2tOgVwytJAi&index=7&ab_channel=Java)

![[JavaOne25/Deeper into Java/screenshots/4.png]]

The presentation covers **four main blocks** of content:

1. **Text Blocks, Sequence Collections, and String APIs**:
    
    - **Text Blocks** were introduced to make multi-line strings with quotes and special characters easier to read and write, solving problems like missing quotes and excessive backslashes seen in traditional strings.
    - They begin and end with triple quotes `"""`. The opening triple quotes must be on their own line.
    - Text blocks are simply **strings**; any method or usage applicable to a string works with a text block.
    - ![[JavaOne25/Deeper into Java/screenshots/5.png]]
    - White space is handled specifically: **incidental white space** (indentation for code readability) is ignored, while **essential white space** (to the right of the indentation line) is preserved.
    - New escape sequences like `\s` for a space and `\` to continue a line without a newline are available.
    - Triple quotes within a text block must be escaped, either by escaping all three `\"""` or just the first one `\"""`.
    - The talk also briefly mentions `indexOf` overloads and advises considering alternatives if using `indexOf` frequently.
    - ![[JavaOne25/Deeper into Java/screenshots/6.png]]
    - **Sequence Collections** were added in Java 21 to provide deterministic order for collections that previously didn't guarantee it, such as `HashSet`.
    - ![[JavaOne25/Deeper into Java/screenshots/7.png]]
    - They introduce interfaces like `SequenceCollection`, `SequenceSet`, and `SequenceMap`.
    - Sequence collections offer new methods like `getFirst()`, `getLast()`, and `reversed()`.
    - ![[JavaOne25/Deeper into Java/screenshots/8.png]]
    - Classes like `LinkedHashSet` and `TreeSet` implement sequence collection interfaces, while `HashSet` does not because its order is not guaranteed. Traditional methods like `iterator().next()` on non-sequenced collections still do not guarantee order.
    - ![[JavaOne25/Deeper into Java/screenshots/9.png]]
2. **Records and Sealed Classes**:
    
    - **Records** dramatically simplify the creation of immutable data carrier classes (like POJOs).
    - They automatically provide: a `final` class, `private final` instance variables, accessors (methods with the same name as the fields, e.g., `title()`), a constructor, and correct `equals()` and `hashCode()` implementations based on all component values.
    - ![[10.png]]
    - While records handle much of the boilerplate, care is still needed for mutable components (like `List`) to ensure true immutability, often requiring defensive copies in a constructor.
    - Records can use **compact constructors**, which have no parameter list and are used to validate or modify parameter values before they are assigned to the record's components.
    - ![[11.png]]
    - Records can implement interfaces.
    - **Sealed Classes** allow a class or interface to explicitly define a restricted set of permitted subclasses or implementing classes.
    - ![[12.png]]
    - Permitted types can be declared as `final` (no further subclassing), `non-sealed` (subclassing is allowed), or `sealed` (the rule recursively applies). Records and enums are implicitly final.
    - Sealed classes work with `instanceof` checks.
    - A minor related feature mentioned is that `static` members (including implicit ones in interfaces/enums) are now legal inside methods and classes.
3. **Instanceof, switch expressions, and pattern matching**:
    
    - This section highlights how **preview features** in Java allow testing new language features before they become standard.
    - **Pattern Matching for `instanceof`** simplifies type checking and casting. Instead of `if (obj instanceof Type) { Type variable = (Type) obj; ... }`, you can write `if (obj instanceof Type variable) { ... }`, and the pattern variable `variable` is automatically available and correctly typed within the `if` block.
    - Pattern variables are scoped to the logic of the check. They can be used after a guard clause (e.g., `if (!(obj instanceof Type variable)) return; ... use variable ...`).
    - Pattern variables are not final and can be reassigned, though this is confusing and not recommended.
    - **Pattern Matching for `switch`** extends this concept to switch statements and expressions.
    - Modern `switch` uses the `->` syntax for cases, avoiding the need for `break` and preventing fall-through errors. Multiple values can be combined in a single case with a comma.
    - ![[13.png]]
    - **Switch expressions** return a value and must be exhaustive, covering all possible input values (e.g., requiring a `default` or handling all enum constants).
    - ![[14.png]]
    - Pattern matching in `switch` allows using type patterns (`case Integer i -> ...`) and guards (`case Integer i when i > 21 -> ...`). The order of cases with patterns matters.
    - Pattern variables declared in switch cases are scoped only to that specific case.
    - ![[15.png]]
    - **Record Patterns** allow decomposing the components of a record directly within `instanceof` checks or `switch` cases, simplifying access to nested data. Example: `if (card instanceof Card(Suit suit, Rank rank))` makes `suit` and `rank` available. Nesting of record patterns is supported.
    - ![[16.png]]
    - Notes restrictions like the inability to use `Boolean`, `float`, `double`, or `long` directly in switch patterns (as of Java 21). Demonstrates using `instanceof` pattern matching to write a concise and correct `equals` method.
4. **Virtual Threads and Scope Values**:
    
    - **Platform Threads** (traditional `java.lang.Thread`) are heavyweight because they are tied to OS threads, limiting the number of concurrent threads and leading to wasted CPU time when threads block on I/O (network, disk).
    - **Virtual Threads** are lightweight implementations of `java.lang.Thread` that are not tied 1:1 to OS threads. They allow running a vast number of concurrent tasks, significantly improving throughput for **I/O-bound** applications. They do **not** improve performance for CPU-bound tasks.
    - A small number of platform threads can manage many virtual threads by "mounting" them when they are running and "unmounting" them when they block, allowing the platform thread to run other virtual threads.
    - Virtual threads can be created using `Executors.newVirtualThreadPerTaskExecutor()` or `Thread.ofVirtual()`. Using the `new Thread()` constructor _always_ creates a platform thread.
    - Virtual threads are often short-lived and disposable due to their low overhead, reducing the need for thread pooling.
    - ![[17.png]]
    - Executors gained `AutoCloseable` support, simplifying shutdown.