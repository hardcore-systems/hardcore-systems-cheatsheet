# Rust Advanced Design Patterns & Typestate Cheatsheet
## J-Ladder Hierarchical Model

### L0 One-Line Essence
Rust design patterns harness static typestates, lifetimes, and ownership boundaries to transform runtime pointer, concurrency, and state machine failures into compile-time guarantee checks.

### L1 Four-Sentence Logic
1. **Typestate Security**: Encodes transition steps into generic types, where state-jump functions consume old types to prohibit invalid method calls at compile-time.
2. **Move Invariance Protection**: Fixes heap/stack memory offsets using `Pin<P>` to ensure self-referential pointer chains remain valid during struct memory shifts.
3. **RAII Resource Boundaries**: Couples physical resources to variables' scope via the Drop trait to automate allocation releases and prevent resource lock leaks.
4. **Zero-Cost Dynamic Abstracts**: Converts strategy dispatches and iterator adapters into flat loops during compilation via generic monomorphizations.

### L2 Core Data Flow
`Typestate start` ➜ `Connection<Closed>` ➜ `connect()` ➜ `consume Closed & emit Connection<Handshaking>` ➜ `call send() ➜ Compile-Time Error` ➜ `complete_handshake()` ➜ `consume Handshaking & emit Connection<Active>` ➜ `send() executes successfully` ➜ `Move occurs ➜ Pinning holds self-refs` ➜ `Out of scope ➜ Drop frees socket`

---

## 📂 Core Knowledge Cards (Cards 1-28)

### Card 1: RAII & Guard Pattern
*   **Theory**: Binds physical resource lifetimes directly to local stack variable scopes, freeing allocations on variable destruction.
*   **Details**: Implements the `Drop` trait. Standard `MutexGuard` acts as an RAII sentinel, locking mutability on entry and unlocking the mutex automatically on exit.
*   **Trade-off**: Calling `std::mem::forget` skips destructor runs, leaking resources permanently.

### Card 2: Typestate Pattern
*   **Theory**: Represents states via generic types to validate valid interfaces at compile-time.
*   **Details**: Declares `Connection<State>` with states `Closed`, `Connecting`, and `Connected`. Restricts `.send()` to `Connection<Connected>`. Transition methods consume `self` by value.
*   **Trade-off**: Discarding and reconstructing generic objects overheads quick runtime state jumps, making classic state patterns cleaner for runtime-dominated systems.

### Card 3: Type-safe Builder Pattern
*   **Theory**: Enforces mandatory field initialization at compile-time, eliminating runtime null pointer exceptions.
*   **Details**: Uses phantom types (e.g., `Builder<HasName, HasEmail>`) to track properties. The compiler blocks the terminal `.build()` invocation if any trait constraint is missing.
*   **Trade-off**: Generates complex compiler errors and nested signatures, which are best managed via derive macros.

### Card 4: Newtype Pattern
*   **Theory**: Wraps base types in tuple structs to enforce semantic separation at compile-time.
*   **Details**: Encapsulates floats like `struct Meter(f64)` and `struct Foot(f64)` to prevent mixed calculations. Bypasses orphan rules to implement foreign traits.
*   **Trade-off**: Native operations (e.g., mathematical symbols) are blocked, requiring manual trait implementations or `Deref` setups.

### Card 5: Self-Referential Structs & Pinning
*   **Theory**: Prevents undefined behaviors by ensuring self-referential pointers remain valid when structs move in memory.
*   **Details**: Wraps pointers in `Pin<P>` to lock memory offsets. Essential for async generator state machines where references point to local fields.
*   **Trade-off**: Bypassing Unpin markers demands unsafe pointer manipulations, which can trigger memory corruptions if mishandled.

### Card 6: Interior Mutability Pattern
*   **Theory**: Allows mutating inner struct values when holding an immutable reference (`&T`).
*   **Details**: Uses `Cell` (value copies for Copy types) or `RefCell` (tracks borrows dynamically, panicking on simultaneous read-write borrows).
*   **Trade-off**: Mismatched borrows trigger `borrow_mut_already_borrowed_panic` crashes; concurrent code must use `Mutex` or `RwLock`.

### Card 7: Orphan Rules & Extension Traits
*   **Theory**: Extends third-party library types with custom methods without altering upstream crates.
*   **Details**: Declares extension traits (e.g. `pub trait MyExt`) locally, implementing them on target types like `std::string::String`.
*   **Trade-off**: Overlapping method names across extensions cause compiler errors, requiring explicit trait imports.

### Card 8: Visitor AST Deserializer Pattern
*   **Theory**: Decouples data format parsing from final memory structures, enabling zero-copy deserialization.
*   **Details**: Standard in Serde. Deserializers parse formats (e.g., JSON) and invoke Visitor callbacks (`visit_str`), matching target fields.
*   **Trade-off**: Deeply nested payloads can overflow the runtime stack; parsers require strict recursion depth limits.

### Card 9: Strategy Dispatch
*   **Theory**: Compares generic Strategy dispatches (monomorphized at compile-time for zero indirect pointer overhead) with dynamic strategy structures (`Box<dyn Strategy>` loaded via vtable pointers).
*   **Details**: Declares dynamic strategy structures using dyn Traits or static generic constraints.
*   **Trade-off**: Static strategy dispatches allow compiler inlinings but increase binary size; dynamic dispatches involve indirect pointer lookups.

### Card 10: RAII Rollback Guard
*   **Theory**: Utilizes stack unwind mechanisms during a thread panic. Destructors check for incomplete transactions and execute automatic database ROLLBACK commands to guard data integrity.
*   **Details**: Implements TransactionGuard structs executing conditional rollbacks in drop callbacks.
*   **Trade-off**: Aborting execution paths immediately bypassing drop skips destructors entirely.

### Card 11: Typestate Handshakes
*   **Theory**: Tracks network handshake processes using generic type conversions, preventing application frames from being dispatched before the connection matures.
*   **Details**: Connects transition methods over type parameters.
*   **Trade-off**: Requires extensive compile-time types mapping.

### Card 12: Deref Coercion
*   **Theory**: Overloads Deref and DerefMut traits, directing unresolved method lookups on wrapper types to the inner wrapped object automatically.
*   **Details**: Implements Deref to Target type.
*   **Trade-off**: Overusing Deref creates anti-patterns.

### Card 13: Lazy Iterators
*   **Theory**: Chain operations like `map()` or `filter()` retain lazy structures. The nested sequence compiles into a flat, highly optimized loop once terminal collections are triggered.
*   **Details**: Employs lazy iterators mapping items inline.
*   **Trade-off**: Nesting filters can complicate compiler lifetime resolutions.

### Card 14: PhantomData Markers
*   **Theory**: Advises borrow checkers on generic parameter lifetimes without incurring physical memory footprints.
*   **Details**: Declares `PhantomData<T>` as a Zero-Sized Type (ZST) to assert ownership bounds and lifetime variances on unused generics.
*   **Trade-off**: PhantomData is a compiler-only construct; excessive use complicates signatures and generic error tracing.

### Card 15: Command Invoker
*   **Theory**: Decouples command dispatchers from system targets, wrapping actions in objects to support undo/redo workflows.
*   **Details**: Declares execute and undo callbacks on commands.
*   **Trade-off**: Small structures multiply rapidly.

### Card 16: Observer Event Bus
*   **Theory**: Thread-safe event distribution structures breaking reference cycles via Weak pointers.
*   **Details**: Manages listeners collections as `Vec<Weak<dyn Listener>>`.
*   **Trade-off**: High-frequency acquisitions risk thread locking stalls.

### Card 17: Abstract Factory
*   **Theory**: Decouples component allocations, resolving creations dynamically.
*   **Details**: Generates registry lookup tables.
*   **Trade-off**: Disables static optimizer loops.

### Card 18: OnceCell Singleton
*   **Theory**: Resolves compiler-static restrictions, initializing global variables safely at runtime.
*   **Details**: Uses `OnceCell` or `once_cell::sync::Lazy` to initialize variables on the first read. Enforces thread-safe write locks and locks values as read-only.
*   **Trade-off**: Singleton values are immutable post-initialization; hot-reloads require custom wrappers.

### Card 19: Flyweight Cow Pattern
*   **Theory**: Saves heap space by keeping references in `Cow<'a, B>`. Automatically clones data to generate owned variables only when mutable writes are triggered.
*   **Details**: Wraps variables in Cow types.
*   **Trade-off**: High write densities make Cow lookups redundant.

### Card 20: Memento Snapshot
*   **Theory**: Captures object properties without exposing private states.
*   **Details**: Packs states into immutable snapshot objects.
*   **Trade-off**: Capturing bulky datasets frequently strains memory.

### Card 21: Adapter Wrapper
*   **Theory**: Interfaces foreign libraries safely, mapping naked pointers to RAII handles.
*   **Details**: Encapsulates unsafe C pointers inside Safe C++ Adapters.
*   **Trade-off**: Generates stack nesting levels.

### Card 22: Bridge Component
*   **Theory**: Separates abstract specifications from physical execution structures.
*   **Details**: Holds pluggable dyn traits inside context models.
*   **Trade-off**: Adds indirect pointers.

### Card 23: Decorator Wrapper
*   **Theory**: Wraps readers in functional layers to enrich runtime features dynamically.
*   **Details**: Implements target traits recursively.
*   **Trade-off**: Deep structures are hard to debug.

### Card 24: Chain Middleware
*   **Theory**: Models sequential request processors that execute checkpoints or chain invocations.
*   **Details**: Manages handlers array passing context.
*   **Trade-off**: Long chains increase latency.

### Card 25: Dynamic State Trait
*   **Theory**: Transitions behavior dynamically by switching states in-place.
*   **Details**: Swaps Boxed state instances inside contexts.
*   **Trade-off**: Trait objects trigger heap allocations.

### Card 26: Template Method Hook
*   **Theory**: Provides skeleton algorithms while leaving target steps open to overrides.
*   **Details**: Declares default trait methods.
*   **Trade-off**: Refactoring base traits breaks implementations.

### Card 27: FFI Safe Wrapper
*   **Theory**: Intercepts thread panics crossing runtime boundaries to prevent aborts.
*   **Details**: Wraps unsafe boundaries in `catch_unwind`.
*   **Trade-off**: Doesn't catch panic=abort configurations.

### Card 28: Actor Channel Isolated
*   **Theory**: Protects task state values by processing mailbox channels sequentially.
*   **Details**: Maps MPSC queues as actor mailbox channels.
*   **Trade-off**: Payload cloning adds memory overheads.
