# Create a SAS-style IF rule

Creates a mutually exclusive IF rule for use inside data_step().

## Usage

``` r
if_do(condition, ...)
```

## Arguments

- condition:

  Logical condition evaluated on the data.table.

- ...:

  Named assignments to apply when condition is TRUE.

## Value

A rule object for data_step().
