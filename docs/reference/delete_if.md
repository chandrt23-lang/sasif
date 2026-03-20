# Create a SAS-style DELETE rule

Creates a DELETE rule to remove rows from the data.table when condition
is TRUE.

## Usage

``` r
delete_if(condition)
```

## Arguments

- condition:

  Logical condition evaluated on the data.table.

## Value

A rule object for data_step().
