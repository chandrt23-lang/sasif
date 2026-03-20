# SAS IF-style data step logic for data.table

Provides SAS-style IF/ELSE chains, independent IF rules, and DELETE
logic for fast, vectorized transformations on data.table objects. This
enables clinical programmers to express SDTM and ADaM-style derivations
in familiar SAS-like syntax while leveraging data.table performance.

## Usage

``` r
data_step(dt, ..., copy = TRUE)
```

## Arguments

- dt:

  A data.table.

- ...:

  One or more rule objects created by if_do(), else_if_do(), else_do(),
  if_independent(), or delete_if().

- copy:

  Logical. If TRUE (default), a copy of dt is modified and returned.

## Value

A data.table with applied transformations.

## Examples

``` r
library(data.table)
#> Warning: package 'data.table' was built under R version 4.5.2

dt <- data.table(
  AGE = c(40, 60, 80),
  SEX = c("M", "F", "M")
)

out <- data_step(
  dt,
  if_do(AGE <= 45, GROUP = 1),
  else_if_do(AGE <= 70, GROUP = 2),
  else_do(GROUP = 3),
  if_independent(SEX == "M", MALE = 1)
)

out
#>      AGE    SEX GROUP  MALE
#>    <num> <char> <num> <num>
#> 1:    40      M     1     1
#> 2:    60      F     2    NA
#> 3:    80      M     3     1
```
