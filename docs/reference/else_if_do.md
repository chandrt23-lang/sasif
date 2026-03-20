# Create a SAS-style ELSE IF rule

Creates an ELSE IF rule for use inside data_step().

## Usage

``` r
else_if_do(condition, ...)
```

## Arguments

- condition:

  Logical condition evaluated on the data.table.

- ...:

  Named assignments to apply when condition is TRUE.

## Value

A rule object for data_step().
