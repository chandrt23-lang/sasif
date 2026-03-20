# Create an independent SAS-style IF rule

Creates an independent IF rule that is evaluated regardless of IF/ELSE
chains.

## Usage

``` r
if_independent(condition, ...)
```

## Arguments

- condition:

  Logical condition evaluated on the data.table.

- ...:

  Named assignments to apply when condition is TRUE.

## Value

A rule object for data_step().
