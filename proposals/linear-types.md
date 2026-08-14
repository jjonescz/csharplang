# Linear Types

This proposal is based on [the ownership proposal](https://github.com/dotnet/csharplang/pull/10296),
only instead of adding a C++-like RAII (implicit `Drop`), we propose a more powerful variant.

## Design

A type can be made _linear_ by marking it with the `[Linear]` attribute.

Each instance of a _linear_ type starts as _owning_ reference which cannot be copied, only [moved](#moving),
and which must be _consumed_ by one of its methods exactly once at some point
(on non-exception code paths; exceptions can be [handled](#exceptions) by implementing a disposable interface).

_Consuming_ is done by instance methods marked with `[OwningRef]` which
- makes the receiver an _owning_ reference,
- ensures the method consumes or moves all its _owned_ [fields](#nested-linear-types).

Consider the following _linear_ type:

```cs
[Linear] struct Transaction
{
    public void Execute(string sql) { ... }
    [OwningRef] public void Commit() { ... }
    [OwningRef] public void Rollback() { ... }
}
```

Thanks to the rules specified above, the compiler ensures either `Commit` or `Rollback` is called before the _owning_ reference goes out of scope.

```cs
void M1()
{
    var tx = new Transaction();
    tx.Execute("select 1;");
    // error: linear type tx is not consumed
}

void M2()
{
    var tx = new Transaction();
    tx.Execute("select 1;");
    tx.Commit(); // ok
    tx.Rollback(); // error: linear type tx is already consumed above
}
```

Notice that the base ownership proposal has no way of expressing this.
One way would be to throw inside `Drop` if neither `Rollback` nor `Commit` has been called,
but that moves the check from compile time to run time, doesn't prevent multiple calls of `Rollback`/`Commit`,
and is actually currently impossible as `Drop` must not throw under the base proposal.
Another way would be to implicitly `Rollback` inside `Drop`,
but that's difficult if `Rollback` must be `async`,
and you also discover that you forgot to commit your transaction only at run time instead of compile time.

### Moving

To move an _owning_ reference, the `move` keyword must be used except for receivers. This is similar to `ref` parameters.

```cs
var f = new File();
Consume(move f);
f.ToString(); // error: f moved above

void Consume(move File f)
{
    f.Close(); // compiler-enforced
}

[Linear] class File
{
    [OwningRef] public void Close() { }
}
```

### Borrowing references

It is possible to borrow a _readonly_ or _mutable_ reference to a _linear_ type.
Such reference cannot outlive the _owning_ reference, ensuring there is no "use after free".

There is new syntax for borrowing: `borrow` and `mut borrow` (similar to `ref readonly` and `ref` for `struct`s).

```cs
var reader = new FileReader();
var result = Parse(mut borrow reader);
// note: reader implicitly dropped here

int[] Parse(mut borrow FileReader reader)
{
    while (reader.TryReadByte(out var b)) { ... }
}

[Linear] class FileReader
{
    // note: `readonly` is allowed on members of linear classes
    public readonly int Position => ...;        // readonly borrowed receiver
    public bool TryReadByte(out byte b) { ... } // mutable borrow receiver
    [OwningRef] public void Drop() { ... }      // moved owning receiver
}
```

More rules are TBD, but they will likely be similar to the base proposal.

### Nested linear types

If a _linear_ type owns other _linear_ types as fields, they must be _consumed_ alongside their parent (and so on recursively).

```cs
[Linear] class TempFile
{
    private readonly File _underlying;

    [OwningRef] public void Delete()
    {
        _underlying.Delete();
        _underlying.Close(); // compiler-enforced
    }
}

[Linear] class File
{
    public void Delete() { ... }
    [OwningRef] public void Close() { ... }
}
```

A `[OwningRef(consumes: false)]` instance method can `move this`;
a `[OwningRef]` instance method has to _consume_ `this`.

```cs
[Linear] class Builder(move Builder parent, string item)
{
    [OwningRef(consumes: false)] public Builder Add(string item) // moves `this`
    {
        return new Builder(move this, item);
    }

    [OwningRef] public string[] Build() // consumes `this`
    {
        return
        [
            item,
            .. parent.Build(), // compiler-enforced
        ];
    }
}
```

This could be enforced automatically (each `[OwningRef]` method could either _consume_ or `move` the receiver),
but for _linear_ types without fields, that might be error-prone:
removing `move this` from a method that was supposed to be non-consuming would silently work.

### Arrays

Arrays `T[]` are _linear_ if `T` is a _linear_ type or [constrained](#generics) to `allows move`.
_Linear_ arrays have `[OwningRef] void Consume(Consumer<T> consumer)` method
which takes a `void Consumer<T>(move T item) where T : allows move` delegate that needs to consume the elements.

### Implicit drop

If a `void Drop()` method is implemented by the _linear_ type, the type opts into _implicit dropping_
which means that the `Drop` method is called automatically when the _owning_ reference goes out of scope.
This corresponds to the behavior of the base ownership proposal.

It is possible to combine an implicit `Drop` with an explicit consuming method:

```cs
{
    var files = new FileList();
    // files.Drop called implicitly here
}

{
    var files = new FileList();
    files.ForEach(f => f.Delete());
}

[Linear] class FileList
{
    private readonly File[] _files;

    // FileList can be consumed in one of two ways:
    [OwningRef] public void ForEach(Consumer consume) { ... } // consumes receiver, so it cannot be used afterwards
    [OwningRef] public void Drop() // implicitly called when the instance goes out of scope
        => _files.Consume(f => { /* drop */ });
}

delegate void Consumer(move File file);

[Linear] class File
{
    // File can be consumed in one of two ways:
    [OwningRef] public void Delete() { ... } // deletes the file, so its handle cannot be used afterwards
    [OwningRef] public void Drop() { } // implicitly called when the instance goes out of scope
}
```

There is also `ValueTask DropAsync()` method which can be implemented instead (the sync `Drop` cannot be declared in that case).
The implicitly-called `DropAsync` method is `await`ed at the call site and it is an error if the containing method is not `async`.

If any field of a _linear_ type cannot be _implicitly dropped_, the containing type either:
- cannot have an implicit `Drop`, or
- its implicit `Drop` must consume the field explicitly.

### Exceptions

If a _linear_ type implements `IDisposable` or `IAsyncDisposable`, the dispose method:

- must be marked as `[OwningRef]`, and
- will be automatically called on exception code paths
  (`DisposeAsync` method is `await`ed at the call site and it is an error if the containing method is not `async`).

For example, the `Transaction` type could implement `IDisposable` to call `Rollback` inside `Dispose`.
Then if `Execute` call throws, the transaction is rolled back at the end of the scope.

This is effectively adding `try`/`catch` to every usage of a _linear_ disposable type.
However, that is not required for memory-safety, hence type authors can choose to not make their _linear_ type disposable,
and save on `try`/`catch`es in exchange for possibly increased resource pressure in exception code paths
(note that it should not be a resource _leak_ if the native resource is managed by a class with a finalizer that disposes it).
The design space is also open to allow making this choice on the consuming side, e.g.,
by using some wrapper type to let the compiler know that we don't want the automatic disposal.

`using` statement is disallowed for _linear_ types.

The automatic disposal is not synthesized in the other methods that own the instance.
Automatic disposal protects owning locals until ownership is transferred.
An owning method is responsible for its own exceptional cleanup.

```cs
var t = new Transaction();
t.Execute("sql");
t.Commit(ComputeOptions());

[Linear] class Transaction : IDisposable
{
    public void Execute(string sql) { ... }
    [OwningRef] public void Commit(...) { ... }
    [OwningRef] public void Dispose() { ... }
}

// equivalent to:

Transaction? t = null;
try
{
    t = new Transaction();
    t.Execute("sql");

    var arg = ComputeOptions();

    var t2 = t;
    t = null;
    t2.Commit(arg);
}
catch
{
    t?.Dispose();
    throw;
}

[Linear] class Transaction : IDisposable
{
    public void Execute(string sql) { ... }

    [OwningRef] public void Commit(...) { ... }
    [OwningRef] public void Dispose() { ... }
}
```

### Conversions

A _linear_ type can be only converted to another _linear_ type.

```cs
object o = new Transaction(); // error: linear type Transaction cannot be converted to non-linear type object
BaseLinearType b = new Transaction(); // ok: b is still linear and needs to be consumed properly
b.Consume();
```

Inheritance rules (TBD) will ensure the base and derived types agree on implicit/explicit `Drop` and on `Drop`/`DropAsync`.

### Generics

We add a new anti-constraint `allows move` to allow substituting a _linear_ type into a generic type parameter.

```cs
[Linear] class OwningList<T> where T : allows move
{
    private readonly T[] _elements;

    public void Add(move T element) { ... } // takes ownership of element

    public mut borrow T this[int index] => _elements[index];

    [OwningRef] public void Consume(Consumer<T> consumer) => _elements.Consume(consumer);
}
```

It is best to consume the list in the same method that created it, otherwise one would need to pass the `Consumer<T>` around virally:

```cs
var list = new OwningList<Transaction>();
var count = Count(borrow list);
list.Consume(tx => tx.Commit());

int Count<T>(borrow OwningList<T> list) where T : allows move => ...;
```

The generic container cannot know if `T` has an implicit drop, but it can be used like this:

```cs
var list = new OwningList<File>();
var count = Count(borrow list);
list.Consume(f => { /* f implicitly dropped */ });
```

### Closures

Closures can be imitated using normal types, although there could be compiler infrastructure
to synthesize those from lambdas and local functions in the future:

```cs
var file = new File();
var action = new FileAction(move file);
new byte[] { 1, 2, 3 }.ForEach(mut borrow action);

[Linear] class FileAction(move File f) : ILinearImplicitAction<byte>
{
    public void Invoke(byte b)
    {
        f.Write(b);
    }

    [OwningRef] public void Drop() { }
}

// linear action with implicit drop
[Linear] interface ILinearImplicitAction<T>
{
    void Invoke(T arg);
    [OwningRef] void Drop();
}

[Linear] class File
{
    public readonly int Length => ...;
    public void Write(byte b) { }
    [OwningRef] public void Drop() { } // this means the File can be implicitly dropped
}

static class Extensions
{
    public static void ForEach<T>(this IEnumerable<T> collection, mut borrow ILinearImplicitAction<T> action)
    {
        foreach (var item in collection)
        {
            action.Invoke(item);
        }
    }
}
```

### Other compilers

Older compilers and other languages will be forbidden to use _linear_ types and `move`/`borrow`/`mut borrow` parameters
by emitting `CompilerFeatureRequiredAttribute, ObsoleteAttribute(error: true)` and `modreq`.

## Examples

### Immutable array builder

```cs
[Linear] public struct ImmutableArrayBuilder<T>
{
    private readonly T[] _backing;

    public ImmutableArrayBuilder(int len)
    {
        _backing = new T[len];
    }

    public T this[int index] { ... }

    [OwningRef]
    public ImmutableArray<T> ToImmutable()
    {
        unsafe
        {
            return ImmutableCollectionsMarshal.AsImmutableArray(this._backing);
        }
    }
}
```

Like in the base ownership proposal, using `ImmutableCollectionsMarshal.AsImmutableArray` here is safe
because we know the backing array won't be changed after the `ToImmutable` method is called.
However, in addition, this version ensures that the builder is not dropped unintentionally without using the built array.

### Array pool

This is very similar to the base proposal except it does not return the array on exception.
That could be changed by implementing `IDisposable`, of course.
But it may be very much desired in general for object pools to not return the rented objects on exception
since an exception might have left those objects in a corrupted state, so we do not want to return them to the pool.

```cs
public sealed class OwnedArrayPool<T>
{
    private readonly ArrayPool<T> _pool;

    public static OwnedArrayPool<T> Shared { get; } =
        new OwnedArrayPool<T>(ArrayPool<T>.Shared);

    private OwnedArrayPool(ArrayPool<T> pool)
    {
        _pool = pool;
    }

    public RentedArray Rent(int minimumLength)
    {
        return new RentedArray(_pool.Rent(minimumLength), this);
    }

    private void Return(T[] array)
    {
        _pool.Return(array);
    }

    [Linear] public readonly struct RentedArray
    {
        private readonly T[] _rented;
        private readonly OwnedArrayPool<T> _pool;

        private RentedArray(T[] rented, OwnedArrayPool<T> pool)
        {
            _rented = rented;
            _pool = pool;
        }

        public Span<T> Span => _rented; // Span gets the right implicit lifetime

        [OwningRef] public void Drop()
        {
            _pool?.Return(_rented); // need to handle `default(RentedArray)`, hence the `?`
        }
    }
}
```

### Buffer writer

A _linear_ extension of the `IBufferWriter<T>` .NET type:

```cs
IBufferWriter<byte> writer = ...;
var reservation = writer.Reserve(256);
int written = Encode(value, reservation.Span);
reservation.Advance(written); // compiler-enforced

[Linear] ref struct BufferReservation
{
    public Span<byte> Span => ...;

    [OwningRef] public void Advance(int written) { ... }
    [OwningRef] public void Cancel() { ... }
}
```

### RPC reply

```cs
void Handle(Request request, move Reply reply)
{
    if (!Authorize(request))
    {
        reply.Reject(403);
        return;
    }

    reply.Send(CreateResponse(request));
}

[Linear] sealed class Reply
{
    [OwningRef] public void Send(Response response) { ... }
    [OwningRef] public void Reject(int statusCode) { ... }
}
```

### Immutable APIs

In general, immutable APIs often require the result to be used or they result in hard-to-see bugs.

Consider Roslyn Workspace APIs. The following snippet has a bug that is not detected in current C# without custom analyzers.

```cs
var workspace = new AdhocWorkspace();
workspace.Solution.AddProject("A");
workspace.Solution.AddProject("B");
```

Merely requiring result values to be used does not help:

```cs
var workspace = new AdhocWorkspace();
var project = workspace.Solution.AddProject("A"); // result used
project = workspace.Solution.AddProject("B"); // result used
workspace.TryApplyChanges(project.Solution); // bug, project "A" not added
```

If types are marked as _linear_ properly, the correct version below would be enforced.
Specifically, project would not have an implicit `Drop` method, hence it would need to be consumed explicitly, for example via `project.Solution`.

```cs
var workspace = new AdhocWorkspace();
var project = workspace.Solution.AddProject("A");
project = project.Solution.AddProject("B");
workspace.TryApplyChanges(project.Solution);
```

It could be still possible to explicitly "fork" the builder if desired in rare scenarios:

```cs
var workspace = new AdhocWorkspace();
var projectA = workspace.Solution.AddProject("A");
var projectB = projectA.Fork().AddProject("B");
// both projectA and projectB usable
projectA.Forget(); projectB.Forget(); // projects don't have an implicit Drop
```

### Builder APIs

Or consider a hypothetical query builder API:

```cs
var b = new QueryBuilder();
var b2 = b.Select("id");
b2 = b.Where("count = 1"); // previously an uncaught bug, now error: b already consumed
```

## Related work

- [Higher RAII via linear types](https://verdagon.dev/blog/higher-raii-uses-linear-types)
- [Must move types in Rust](https://smallcultfollowing.com/babysteps/blog/2023/03/16/must-move-types/)
- [C# proposal: destructible types](https://github.com/dotnet/roslyn/issues/161)
