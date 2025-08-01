# Perfect backwards compatibility policy

Narwhals is primarily aimed at library maintainers rather than end users. As such,
we need to take stability and backwards compatibility extra-seriously. Our policy is:

- If you write code using `narwhals.stable.v1` or `import narwhals.stable.v2`, then we promise to
  never change or remove any public function you're using.
- If we need to make a backwards-incompatible change, it will be pushed into
  the main `narwhals` namespace (and eventually `narwhals.stable.v3`),
  leaving `narwhals.stable.v1` and `narwhals.stable.v2` unaffected.
- We will maintain `narwhals.stable.v1` and `narwhals.stable.v2` indefinitely,
  even as `narwhals.stable.v3` and other stable APIs come out. For example,
  Narwhals version 1.0.0 offers `narwhals.stable.v1`, whereas Narwhals 2.0.0 offers
  both `narwhals.stable.v1` and `narwhals.stable.v2`.

Like this, we enable different packages to be on different Narwhals stable APIs, and for
end-users to use all of them in the same project without conflicts nor
incompatibilities.

## Background

Ever upgraded a package, only to find that it breaks all your tests because of an intentional
API change? Did you end up having to litter your code with statements such as the following?

```python
if parse_version(pdx.__version__) < parse_version("1.3.0"):
    df = df.brewbeer()
elif parse_version("1.3.0") <= parse_version(pdx.__version__) < parse_version("1.5.0"):
    df = df.brew_beer()
else:
    df = df.brew_drink("beer")
```

Now imagine multiplying that complexity over all the dataframe libraries you want to support...

Narwhals offers a simple solution, inspired by Rust editions.

## Narwhals' Stable API

Narwhals implements a subset of the Polars API. What will Narwhals do if/when Polars makes
a backwards-incompatible change? Would you need to update your Narwhals code?

To understand the solution, let's go through an example. Suppose that, hypothetically, in Polars 2.0,
`polars.Expr.cum_sum` was renamed to `polars.Expr.cumulative_sum`. In Narwhals, we
have `narwhals.Expr.cum_sum`. Does this mean that Narwhals will also rename its method,
and deprecate the old one? The answer is...no!

Narwhals offers a `stable` namespace, which allows you to write your code once and forget about
it. That is to say, if you write your code like this:

=== "from/to_native"
    ```python
    import narwhals.stable.v2 as nw
    from narwhals.stable.v2.typing import IntoFrameT


    def func(df: IntoFrameT) -> IntoFrameT:
        return nw.from_native(df).with_columns(nw.col("a").cum_sum()).to_native()
    ```

=== "@narwhalify"
    ```python
    import narwhals.stable.v2 as nw
    from narwhals.stable.v2.typing import FrameT


    @nw.narwhalify
    def func(df: FrameT) -> FrameT:
        return df.with_columns(nw.col("a").cum_sum())
    ```

then we, in Narwhals, promise that your code will keep working, even in newer versions of Polars
after they have renamed their method.

Concretely, we would do the following:

- `narwhals.stable.v2`: you can keep using `Expr.cum_sum`
- `narwhals.stable.v3`: you can only use `Expr.cumulative_sum`, `Expr.cum_sum` will have been removed
- `narwhals`:  you can only use `Expr.cumulative_sum`, `Expr.cum_sum` will have been removed

So, although Narwhals' main API (and `narwhals.stable.v3`) will have introduced a breaking change,
users of `narwhals.stable.v2` will have their code unaffected.

### Exceptions

Are we really promising perfect backwards compatibility in all cases, without exceptions? Not quite.
There are some exceptions, which we'll now list. But we'll never intentionally break your code.
Anything currently in `narwhals.stable.v1` or `narwhals.stable.v2` will not be changed or removed
in future Narwhals versions.

Here are exceptions to our backwards compatibility policy:

- Unambiguous bugs. If a function contains what is unambiguously a bug, then we'll fix it, without
  considering that to be a breaking change.
- Radical changes in backends. Suppose that Polars was to remove
  expressions, or pandas were to remove support for categorical data. At that point, we might
  need to rethink Narwhals. However, we expect such radical changes to be exceedingly unlikely.
- We may consider making some type hints more precise.
- Anything labelled "unstable".
- We may sometimes need to bump the minimum versions of supported backends.

In general, decision are driven by use-cases, and we conduct a search of public GitHub repositories
before making any change.

### `import narwhals as nw`, `import narwhals.stable.v2 as nw`, or `import narwhals.stable.v1 as nw`?

Which should you use? In general we recommend:

- When prototyping, use `import narwhals as nw`, so you can iterate quickly.
- Once you're happy with what you've got and want to release something production-ready and stable,
  then switch out your `import narwhals as nw` usage for `import narwhals.stable.v2 as nw`.
- If you're starting a new project, use either the main Narwhals namespace or `narwhals.stable.v2`.
- If your project is already using `narwhals.stable.v1`, and you don't need any of the newer Narwhals
  features, there's probably no need to switch to `narwhals.stable.v2`, as that would require you to
  raise the minimum version of Narwhals you support. If you'd like to use `narwhals.stable.v2`, make
  sure to require at least `narwhals>=2.0`.

## `main` vs `stable.v2` differences

So far, nothing, everything non-unstable from the main namespace should be available in `narwhals.stable.v2`.

## `main` vs `stable.v1` differences

- Since Narwhals 1.49:

    - `nw.Expr.arg_max` and `nw.Expr.arg_min` are deprecated from the main Narwhals namespace.
      Note that the `Series` methods (`Series.arg_max`, `Series.arg_min`) will remain available.

- Since Narwhals 1.45:

    - `nw.any_horizontal` and `nw.all_horizontal` have a `ignore_nulls` keyword. In `narwhals.stable.v1`,
      it defaults to `False`, but in Narwhals 2.0 it will become a required argument in the main namespace.
    - `LazyFrame.with_row_index` requires `order_by` to be specified as it is an order-dependent operation, in the main Narwhals namespace.

- Since Narwhals 1.43:

    - `nw.get_level` is deprecated in the main Narwhals namespace.

- Since Narwhals 1.35:

    - pandas' ordered categoricals get mapped to `nw.Enum` instead of `nw.Categorical`.
    - `nw.Enum` must be provided `categories` at instantiation.

- Since Narwhals 1.29.0, `LazyFrame.gather_every` has been deprecated from the main namespace.

- Since Narwhals 1.25.0, `native_namespace` is generally deprecated across the API. Please
  use `backend` instead.

- Since Narwhals 1.24.1, an empty or all-null object-dtype pandas Series is inferred to
  be of dtype `String`. Previously, it would have been inferred as `Object`.

- Since Narwhals 1.23:

    - Passing an `ibis.Table` to `from_native` returns a `LazyFrame`. In
      `narwhals.stable.v1`, it returns a `DataFrame` with `level='interchange'`.
    - `eager_or_interchange_only` has been removed from `from_native` and `narwhalify`.
    - Order-dependent expressions can no longer be used with `narwhals.LazyFrame`.
    - The following expressions have been deprecated from the main namespace: `Expr.head`,
      `Expr.tail`, `Expr.gather_every`, `Expr.sample`, `Expr.arg_true`, `Expr.sort`.

- Since Narwhals 1.21, passing a `DuckDBPyRelation` to `from_native` returns a `LazyFrame`. In
  `narwhals.stable.v1`, it returns a `DataFrame` with `level='interchange'`.

- Since Narwhals 1.15, `Series` is generic in the native Series, meaning that you can
  write:
  ```python
  import narwhals as nw
  import polars as pl

  s_pl = pl.Series([1, 2, 3])
  s = nw.from_native(s, series_only=True)
  # mypy infers `s.to_native()` to be `polars.Series`
  reveal_type(s.to_native())
  ```
  Previously, `Series` was not generic, so in the above example
  `s.to_native()` would have been inferred as `Any`.

- Since Narwhals 1.13.0, the `strict` parameter in `from_native`, `to_native`, and `narwhalify`
    has been deprecated in favour of `pass_through`. This is because several users expressed
    confusion/surprise over what `strict=False` did.
    ```python
    # v1 syntax:
    nw.from_native(df, strict=False)

    # main namespace (and, when we get there, v2) syntax:
    nw.from_native(df, pass_through=True)
    ```
    If you are using Narwhals>=1.13.0, then we recommend using `pass_through`, as that
    works consistently across namespaces.

    In the future:

    - in the main Narwhals namespace, `strict` will be removed in favour of `pass_through`
    - in `stable.v1`, we will keep both `strict` and `pass_through`

- Since Narwhals 1.9.0, `Datetime` and `Duration` dtypes hash using both `time_unit` and
    `time_zone`.
    The effect of this can be seen when placing these dtypes in sets:

    ```python exec="1" source="above" session="backcompat"
    import narwhals.stable.v1 as nw_v1
    import narwhals as nw

    # v1 behaviour:
    assert nw_v1.Datetime("us") in {nw_v1.Datetime}

    # main namespace (and, when we get there, v2) behaviour:
    assert nw.Datetime("us") not in {nw.Datetime}
    assert nw.Datetime("us") in {nw.Datetime("us")}
    ```

    To check if a dtype is a datetime (regardless of `time_unit` or `time_zone`)
    we recommend using `==` instead, as that works consistently
    across namespaces:

    ```python exec="1" source="above" session="backcompat"
    # Recommended
    assert nw.Datetime("us") == nw.Datetime
    assert nw_v1.Datetime("us") == nw_v1.Datetime
    ```
