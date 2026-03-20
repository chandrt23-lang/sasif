# Clinical ADaM Derivations with sasif

## Introduction

Clinical programmers working in R often face a common challenge when
migrating from SAS: in SAS, a single `IF ... THEN DO` block can assign
**multiple variables** at once under one condition. In R, traditional
approaches like `case_when()` or
[`fifelse()`](https://rdrr.io/pkg/data.table/man/fifelse.html) force you
to **repeat the same condition for every variable** — increasing QC risk
and reducing readability.

`sasif` solves this by bringing SAS-style `IF / ELSE IF / ELSE` control
flow into R’s `data.table` ecosystem. One condition governs all
assignments in a block — just like SAS.

This vignette walks through three real-world ADaM derivation scenarios:

1.  **ADSL** — Population flags and treatment variables
2.  **ADLB** — Laboratory value categorisation
3.  **ADAE** — Treatment-emergent adverse event flags

------------------------------------------------------------------------

## Setup

``` r
library(sasif)
library(data.table)
#> Warning: package 'data.table' was built under R version 4.5.2
```

------------------------------------------------------------------------

## Scenario 1 — ADSL: Population Flags

### The Problem

In a typical ADSL derivation, when a subject is in the treatment arm,
multiple variables need to be assigned simultaneously — population
flags, treatment labels, numeric codes, and treatment dates.

In traditional R, every variable requires its own repeated condition:

``` r
# ❌ Traditional R — condition repeated for every variable
adsl <- adsl %>% mutate(
  SAFFL   = case_when(ACTARMCD == "TRTA" ~ "Y"),
  SAFFLN  = case_when(ACTARMCD == "TRTA" ~ 1),
  TRT01A  = case_when(ACTARMCD == "TRTA" ~ ACTARMCD),
  TRT01AN = case_when(ACTARMCD == "TRTA" ~ 1),
  ITTFL   = case_when(ACTARMCD == "TRTA" ~ "Y"),
  FASFL   = case_when(ACTARMCD == "TRTA" ~ "Y"),
  RANDFL  = case_when(ACTARMCD == "TRTA" ~ "Y"),
  PPFL    = case_when(ACTARMCD == "TRTA" ~ "Y")
  # Same condition written 8 times — high QC risk
)
```

If the condition ever changes, you must update it in 8 places. Miss one
and your derivation silently diverges — a real risk in regulated
environments.

### The sasif Solution

``` r
# Create sample ADSL data
adsl <- data.table(
  USUBJID  = c("S01", "S02", "S03", "S04"),
  ACTARMCD = c("TRTA", "TRTA", "SCRNFAIL", "TRTA"),
  RFSTDTC  = c("2024-01-10", "2024-01-15", NA, "2024-01-20"),
  RFENDTC  = c("2024-06-10", "2024-06-15", NA, "2024-06-20")
)

# ✅ sasif — condition written ONCE, governs all assignments
ADSL <- data_step(adsl,
  if_do(ACTARMCD == "TRTA",
    SAFFL   = "Y",
    SAFFLN  = 1,
    TRT01A  = "Treatment A",
    TRT01AN = 1,
    TRTSDT  = as.Date(RFSTDTC, "%Y-%m-%d"),
    TRTEDT  = as.Date(RFENDTC, "%Y-%m-%d"),
    ITTFL   = "Y",
    FASFL   = "Y",
    RANDFL  = "Y",
    PPFL    = "Y"
  )
)

print(ADSL[, .(USUBJID, ACTARMCD, SAFFL, TRT01A, TRT01AN, ITTFL, FASFL)])
#>    USUBJID ACTARMCD  SAFFL      TRT01A TRT01AN  ITTFL  FASFL
#>     <char>   <char> <char>      <char>   <num> <char> <char>
#> 1:     S01     TRTA      Y Treatment A       1      Y      Y
#> 2:     S02     TRTA      Y Treatment A       1      Y      Y
#> 3:     S03 SCRNFAIL   <NA>        <NA>      NA   <NA>   <NA>
#> 4:     S04     TRTA      Y Treatment A       1      Y      Y
```

All 10 variables are derived from a single condition block. Clean,
readable, and audit-friendly — exactly like SAS `IF ... THEN DO`.

------------------------------------------------------------------------

## Scenario 2 — ADSL: Multi-Arm Treatment Assignment (IF / ELSE IF / ELSE)

When a study has multiple treatment arms, use the full IF / ELSE IF /
ELSE chain. The first matching condition wins — all others are skipped:

``` r
adsl2 <- data.table(
  USUBJID  = c("S01", "S02", "S03", "S04", "S05"),
  ACTARMCD = c("TRTA", "TRTB", "TRTC", "TRTA", "TRTB"),
  AGE      = c(35, 52, 67, 44, 58)
)

ADSL2 <- data_step(adsl2,
  if_do(ACTARMCD == "TRTA",
    TRT01A  = "Treatment A",
    TRT01AN = 1
  ),
  else_if_do(ACTARMCD == "TRTB",
    TRT01A  = "Treatment B",
    TRT01AN = 2
  ),
  else_do(
    TRT01A  = "Placebo",
    TRT01AN = 99
  )
)

print(ADSL2[, .(USUBJID, ACTARMCD, TRT01A, TRT01AN)])
#>    USUBJID ACTARMCD      TRT01A TRT01AN
#>     <char>   <char>      <char>   <num>
#> 1:     S01     TRTA Treatment A       1
#> 2:     S02     TRTB Treatment B       2
#> 3:     S03     TRTC     Placebo      99
#> 4:     S04     TRTA Treatment A       1
#> 5:     S05     TRTB Treatment B       2
```

Notice that both `TRT01A` (character label) and `TRT01AN` (numeric code)
are derived together under each condition — no repetition needed.

------------------------------------------------------------------------

## Scenario 3 — ADSL: Age Categorisation

Derive both the age category label and its numeric code in one chain:

``` r
adsl3 <- data.table(
  USUBJID = c("S01", "S02", "S03", "S04", "S05"),
  AGE     = c(32, 45, 58, 71, 80)
)

ADSL3 <- data_step(adsl3,
  if_do(AGE <= 45,
    AGECAT  = "YOUNG",
    AGECATN = 1
  ),
  else_if_do(AGE <= 70,
    AGECAT  = "MIDDLE",
    AGECATN = 2
  ),
  else_do(
    AGECAT  = "OLD",
    AGECATN = 3
  )
)

print(ADSL3[, .(USUBJID, AGE, AGECAT, AGECATN)])
#>    USUBJID   AGE AGECAT AGECATN
#>     <char> <num> <char>   <num>
#> 1:     S01    32  YOUNG       1
#> 2:     S02    45  YOUNG       1
#> 3:     S03    58 MIDDLE       2
#> 4:     S04    71    OLD       3
#> 5:     S05    80    OLD       3
```

------------------------------------------------------------------------

## Scenario 4 — ADLB: Laboratory Value Categorisation

A common ADaM derivation — categorise lab values as LOW, NORMAL, or HIGH
based on reference ranges, and derive both the character and numeric
category together:

``` r
adlb <- data.table(
  USUBJID  = c("S01", "S01", "S02", "S02", "S03"),
  LBTESTCD = c("ALB", "ALB", "ALB", "ALB", "ALB"),
  AVAL     = c(2.8, 4.2, 5.6, 3.5, 1.9),
  ANRLO    = c(3.5, 3.5, 3.5, 3.5, 3.5),
  ANRHI    = c(5.0, 5.0, 5.0, 5.0, 5.0)
)

ADLB <- data_step(adlb,
  if_do(LBTESTCD == "ALB" & AVAL < ANRLO,
    ALBCAT  = "LOW",
    ALBCATN = 1
  ),
  else_if_do(LBTESTCD == "ALB" & AVAL > ANRHI,
    ALBCAT  = "HIGH",
    ALBCATN = 2
  ),
  else_do(
    ALBCAT  = "NORMAL",
    ALBCATN = 3
  )
)

print(ADLB[, .(USUBJID, LBTESTCD, AVAL, ANRLO, ANRHI, ALBCAT, ALBCATN)])
#>    USUBJID LBTESTCD  AVAL ANRLO ANRHI ALBCAT ALBCATN
#>     <char>   <char> <num> <num> <num> <char>   <num>
#> 1:     S01      ALB   2.8   3.5     5    LOW       1
#> 2:     S01      ALB   4.2   3.5     5 NORMAL       3
#> 3:     S02      ALB   5.6   3.5     5   HIGH       2
#> 4:     S02      ALB   3.5   3.5     5 NORMAL       3
#> 5:     S03      ALB   1.9   3.5     5    LOW       1
```

Both `ALBCAT` and `ALBCATN` are always consistent — they are derived
from the same condition, so they can never diverge.

------------------------------------------------------------------------

## Scenario 5 — ADAE: Treatment-Emergent Flag (TRTEMFL)

Flag adverse events that started on or after the treatment start date:

``` r
adae <- data.table(
  USUBJID = c("S01", "S01", "S02", "S02", "S03"),
  AEDECOD = c("Headache", "Nausea", "Fatigue", "Dizziness", "Rash"),
  ASTDT   = as.Date(c("2024-01-15", "2023-12-01",
                       "2024-01-20", "2024-02-10", "2024-01-25")),
  TRTSDT  = as.Date(c("2024-01-10", "2024-01-10",
                       "2024-01-15", "2024-01-15", "2024-01-20")),
  TRTEDT  = as.Date(c("2024-06-10", "2024-06-10",
                       "2024-06-15", "2024-06-15", "2024-06-20"))
)

ADAE <- data_step(adae,
  if_do(ASTDT >= TRTSDT & ASTDT <= TRTEDT,
    TRTEMFL = "Y",
    TRTEMA  = AEDECOD
  )
)

print(ADAE[, .(USUBJID, AEDECOD, ASTDT, TRTSDT, TRTEMFL)])
#>    USUBJID   AEDECOD      ASTDT     TRTSDT TRTEMFL
#>     <char>    <char>     <Date>     <Date>  <char>
#> 1:     S01  Headache 2024-01-15 2024-01-10       Y
#> 2:     S01    Nausea 2023-12-01 2024-01-10    <NA>
#> 3:     S02   Fatigue 2024-01-20 2024-01-15       Y
#> 4:     S02 Dizziness 2024-02-10 2024-01-15       Y
#> 5:     S03      Rash 2024-01-25 2024-01-20       Y
```

------------------------------------------------------------------------

## Scenario 6 — DELETE: Remove Unwanted Records

Use
[`delete_if()`](https://chandrt23-lang.github.io/sasif/reference/delete_if.md)
to remove rows explicitly — mirrors the SAS `DELETE` statement and makes
the intent clear in the code:

``` r
adlb2 <- data.table(
  USUBJID  = c("S01", "S02", "S03", "S04", "S05"),
  LBTESTCD = c("ALB", NA,    "ALB", "ALB", NA),
  VISIT    = c("WEEK 1", "WEEK 1", "UNSCHEDULED", "WEEK 2", "WEEK 4"),
  AVAL     = c(4.2, 3.8, 5.1, 4.0, 3.5)
)

ADLB2 <- data_step(adlb2,
  delete_if(is.na(LBTESTCD)),
  delete_if(VISIT == "UNSCHEDULED")
)

print(ADLB2)
#>    USUBJID LBTESTCD  VISIT  AVAL
#>     <char>   <char> <char> <num>
#> 1:     S01      ALB WEEK 1   4.2
#> 2:     S04      ALB WEEK 2   4.0
```

Only records with valid test codes and scheduled visits are retained.

------------------------------------------------------------------------

## Scenario 7 — Independent Flags (if_independent)

Use
[`if_independent()`](https://chandrt23-lang.github.io/sasif/reference/if_independent.md)
when conditions are **not** mutually exclusive — each condition is
evaluated on its own, so multiple flags can apply to the same row
simultaneously:

``` r
adsl4 <- data.table(
  USUBJID = c("S01", "S02", "S03", "S04"),
  AGE     = c(30, 68, 45, 72),
  WEIGHTKG = c(48, 72, 55, 43),
  DIABFL  = c("N", "Y", "N", "Y")
)

ADSL4 <- data_step(adsl4,
  if_independent(AGE > 65,       SENIORFL  = "Y"),
  if_independent(WEIGHTKG < 50,  LOWWTFL   = "Y"),
  if_independent(DIABFL == "Y",  COMORBFL  = "Y")
)

print(ADSL4)
#>    USUBJID   AGE WEIGHTKG DIABFL SENIORFL LOWWTFL COMORBFL
#>     <char> <num>    <num> <char>   <char>  <char>   <char>
#> 1:     S01    30       48      N     <NA>       Y     <NA>
#> 2:     S02    68       72      Y        Y    <NA>        Y
#> 3:     S03    45       55      N     <NA>    <NA>     <NA>
#> 4:     S04    72       43      Y        Y       Y        Y
```

Subject S04 (age 72, weight 43, diabetic) receives all three flags —
because all three conditions are TRUE for that row simultaneously.

------------------------------------------------------------------------

## Key Principle: When to Use Which Function

| Situation | Use |
|----|----|
| First matching condition should win | [`if_do()`](https://chandrt23-lang.github.io/sasif/reference/if_do.md) + [`else_if_do()`](https://chandrt23-lang.github.io/sasif/reference/else_if_do.md) + [`else_do()`](https://chandrt23-lang.github.io/sasif/reference/else_do.md) |
| Multiple conditions can apply to same row | [`if_independent()`](https://chandrt23-lang.github.io/sasif/reference/if_independent.md) |
| Remove rows from dataset | [`delete_if()`](https://chandrt23-lang.github.io/sasif/reference/delete_if.md) |

> **Important:** Do not mix
> [`if_do()`](https://chandrt23-lang.github.io/sasif/reference/if_do.md)
> chains with
> [`if_independent()`](https://chandrt23-lang.github.io/sasif/reference/if_independent.md)
> on the same variable.
> [`if_independent()`](https://chandrt23-lang.github.io/sasif/reference/if_independent.md)
> runs **after** the chain and will overwrite earlier assignments. Use
> one approach consistently per variable.

------------------------------------------------------------------------

## Summary

`sasif` brings three key benefits to clinical R programming:

- **One condition, multiple assignments** — no repeated logic, no QC
  risk of conditions diverging
- **Familiar SAS syntax** — `IF / ELSE IF / ELSE` control flow that
  clinical programmers already know
- **data.table performance** — fully vectorized, no row loops, scales to
  millions of rows

For more information, see the [package
documentation](https://chandrt23-lang.github.io/sasif).
