# Variable Parameters for Lambdas and Method Groups

## Summary

Allow `params` arrays in lambdas and inferred delegates:

```cs
var lambda = (params int[] xs) => xs.Length;
lambda(); // 0
lambda(1, 2, 3); // 3

int method(params int[] xs)
{
    return xs.Length;
}
var inferred = method; // <anonymous delegate>
inferred(); // 0
inferred(1, 2); // 2
```

## Relevant Background

- [`params` arrays](https://docs.microsoft.com/dotnet/csharp/language-reference/keywords/params)
- [Default Parameters for Lambdas and Method Groups](./lambda-method-group-defaults.md)

## Motivation

While we don't have a direct use case, we think that if we want to make this change, we should make it now as it is a breaking change for inferred delegate type scenarios when inferring from a method that has a params parameter. We don't think that waiting for a use case would change the semantic design strategy here, as it needs to be consistent with the work already approved for default lambda parameters.

## Current Behavior

Currently, when a user implements a lambda with `params` array, the compiler raises an error stating that `params` is not allowed. 

```cs
var lambda = (params int[] xs) => xs.Length; // error CS1670: params is not valid in this context
```

When a user attempts to use a method group where the underlying method has a `params` array, the
`params` annotation isn't propagated, so the call to the method doesn't typecheck due to a mismatch in the number of expected parameters.

```cs
int method(params int[] xs) { }
var inferred = method; // System.Action<int[]>
inferred(1, 2, 3); // error CS1593: Delegate 'Action<int[]>' does not take 3 arguments
```

## Breaking Change

Currently, the inferred type of a method group is `Action` or `Func` so the following code compiles:

```cs
int method(params int[] xs)
{
  return xs.Length;
}
var inferred = method; // System.Func<int[], int>
Apply(inferred, 3); // Ok

int Apply(System.Func<int[], int> f, int p)
{
  return f(new[] { p });
}
```

Following this change, code of this nature would cease to compile.

```csharp
int method(params int[] xs)
{
  return xs.Length;
}
var inferred = method; // <anonymous delegate>
Apply(inferred, 3); // error CS1503: Argument 1: cannot convert from '<anonymous delegate>' to 'System.Func<int[], int>'

int Apply(System.Func<int[], int> f, int p)
{
  return f(new[] { p });
}
```

The impact of this breaking change needs to be considered. Fortunately, the use of `var` to infer the type of a method group has only been supported since C# 10, so only code which has been written since then explicitly relying on this behavior would break.

## Detailed design

### Syntax

This enhancement requires the following addition to the grammar for lambda expressions.

```ANTLR
lambda_parameter_list
    : lambda_parameters
    | lambda_parameters ',' parameter_array
    | parameter_array
    ;
```

No changes to the grammar are necessary for method groups since this proposal would only change their semantics.

### Binder Changes

#### Synthesizing New Delegate Types

As with the behavior for delegates with `ref`, `out`, or default parameters, a new natural type is generated for each lambda or method group defined with `params` array.
Note that in the below examples, the notation `a'`, `b'`, etc. is used to represent these anonymous delegate types.

```cs
var lambda1 = (params int[] xs) => xs.Length; // internal delegate int a'(params int[] arg1)
var lambda2 = (params string[] values) => string.Join(',', values); // internal delegate void b'(params string[] arg1);
var inferred = lambda2; // internal delegate void b'(params string[] arg1);
```

#### Conversion and Unification Behavior 

The anonymous delegates described previously will be unified and converted consistently with named delegates which already support `params` arrays.

### IL/Runtime Behavior

The IL for this feature will be very similar in nature to the IL emitted for lambdas with `ref`, `out`, and default parameters. A class which inherits from `System.Delegate` or similar will be generated, and the `Invoke` method will include `.param` directives to set `System.ParamArrayAttribute`&mdash;just as would be the case for a standard named delegate with `params` array.

## Design meetings

- [LDM 2022-10-10](https://github.com/dotnet/csharplang/blob/main/meetings/2022/LDM-2022-10-10.md#params-support-for-lambda-default-parameters): decision to add support for `params` in the same way as default parameters.
