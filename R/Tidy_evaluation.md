# Tidy Evaluation

## Simplest function
### Evaluation
#### `identity`
```R
identity(6)
#> [1] 6

identity(2 * 3)
#> [1] 6

a <- 2
b <- 3
identity(a * b)
#> [1] 6
```

Another:
```R
df <- data.frame(
  y = 1,
  var = 2
)

df$y
#> [1] 1
```

### Quoting
### `quote`
```R
quote(6)
#> [1] 6

quote(2 * 3)
#> 2 * 3

quote(a * b)
#> a * b
```

Same Same :
```R
"a * b"
#> [1] "a * b"

~a * b
#> ~a * b

quote(a * b)
#> a * b
```
Another:
```R
df <- data.frame(
  y = 1,
  var = 2
)

var <- "y"
df[[var]]
#> [1] 1
```

### Direct & Indirect Evaluation

Equivalent:
```R
df[[var]] # Indirect
#> [1] 1

df[["y"]] # Direct
#> [1] 1
```
Not Equivalent:
```R
df$var    # Direct
#> [1] 2

df$y      # Direct
#> [1] 1
```

||Quoted|Evaluated|
|---|---|---|
|Direct| 	`df$y`| 	`df[["y"]]`|
|Indirect| 	???| 	`df[[var]]`|

## Unquoting in the tidyverse

All quoting functions in the tidyverse support a single unquotation mechanism, the `!!` operator (pronounced **bang-bang**). You can use `!!` to cancel the automatic quotation and supply indirect references everywhere an argument is automatically quoted.

- In dplyr most verbs quote their arguments:
```R
library("dplyr")

by_cyl <- mtcars %>%
  group_by(!!x_var) %>%            # Refer to x_var
  summarise(mean = mean(!!y_var))  # Refer to y_var
```

- In ggplot2 `aes()` is the main quoting function:

```R
library("ggplot2")

ggplot(mtcars, aes(!!x_var, !!y_var)) +  # Refer to x_var and y_var
  geom_point()
```

## Quote and unquote

1. Use `enquo()` to make a function automatically quote its argument.
2. Use `!!` to unquote the argument.

Example:
```R
grouped_mean <- function(data, group_var, summary_var) {
  group_var <- enquo(group_var)
  summary_var <- enquo(summary_var)

  data %>%
    group_by(!!group_var) %>%
    summarise(mean = mean(!!summary_var))
}
#Use:
grouped_mean(mtcars, cyl, mpg)
```

## Strings instead of quotes

**This doesn't WORK:**
```R
var <- "height"
mutate(starwars, rescaled = !!var * 100)
#> Error in mutate_impl(.data, dots): Evaluation error: non-numeric argument to binary operator.
```

### Strings

```R
"height"
#> [1] "height"   -> R String

quote(height)
#> height         expression -> R symbol

sym("height")
#> height         "string"-> R symbol

var = "height"

enquo(var)
#> height         -> R symbol
```
A symbol is much more than a string, it is a reference to an R object.

Difference between `enquo()` and `quote()` is that:
- `enquo()` : recives a symbol representing an argument as parameter. The expression supplied to that argument will be captured instead of being evaluated.
- `quote()`: recives an expression but with the difference that does't evaluates it

Alternate example:
```R
grouped_mean2 <- function(data, group_var, summary_var) {
  group_var <- sym(group_var)
  summary_var <- sym(summary_var)

  data %>%
    group_by(!!group_var) %>%
    summarise(mean = mean(!!summary_var))
}

#Use:
grouped_mean2(starwars, "gender", "mass")
```

## Dealing with multiple arguments

Quoting and unquoting multiple variables is pretty much the same process as for single arguments:

- Unquoting multiple arguments requires a variant of `!!`, the big bang operator `!!!`.

- Quoting multiple arguments can be done in two ways: internal quoting with the plural variant `enquos()` and external quoting with `vars()`.

### The `...` argument

As a programmer you can do three things with `...` :

1. **Evaluate** the arguments contained in the dots and materialise them in a list by forwarding the dots to `list()`:
```R
materialise <- function(data, ...) {
    dots <- list(...)
    dots
}
materialise(mtcars, 1 + 2, important_name = letters)
#> [[1]]
#> [1] 3
#> 
#> $important_name
#>  [1] "a" "b" "c" "d" "e" "f" "g" "h" "i" "j" "k" "l" "m" "n" "o" "p" "q"
#> [18] "r" "s" "t" "u" "v" "w" "x" "y" "z"
```

2. Quote the arguments in the dots with `enquos()`:

```R
capture <- function(data, ...) {
    dots <- enquos(...)
    dots
}

# All arguments passed to ... are automatically quoted and returned as a list. The names of the arguments become the names of that list:

capture(mtcars, 1 + 2, important_name = letters)
#> [[1]]
#> <quosure>
#>   expr: ^1 + 2
#>   env:  global
#> 
#> $important_name
#> <quosure>
#>   expr: ^letters
#>   env:  global
```

3. Forward the dots to another function:
```R
forward <- function(data, ...) {
  forwardee(...)
}
# When dots are forwarded the names of arguments in ... are matched to the arguments of the forwardee:

forwardee <- function(foo, bar, ...) {
  list(foo = foo, bar = bar, ...)
}

forward(mtcars, bar = 100, 1, 2, 3)
#> $foo
#> [1] 1
#> 
#> $bar
#> [1] 100
#> 
#> [[3]]
#> [1] 2
#> 
#> [[4]]
#> [1] 3
```

The last two techniques are important. There are two distinct situations:

1. You don’t need to modify the arguments in any way, just passing them through. Then simply forward `...` to other quoting functions in the ordinary way.

2. You’d like to change the argument names (which become column names in `dplyr::mutate()` calls) or modify the arguments themselves (for instance negate a `dplyr::select()`ion). In that case you’ll need to use `enquos()` to quote the arguments in the dots. You’ll then pass the quoted arguments to other quoting functions by forwarding them with the help of `!!!`.


### Simple forwarding of `...`

Example:

```R
forward <- function(data, ...) {
  forwardee(...)
}
# When dots are forwarded the names of arguments in ... are matched to the arguments of the forwardee:

forwardee <- function(foo, bar, ...) {
  list(foo = foo, bar = bar, ...)
}

forward(mtcars, bar = 100, 1, 2, 3)
grouped_mean <- function(.data, .summary_var, ...) {
  .summary_var <- enquo(.summary_var)

  .data %>%
    group_by(...) %>%
    summarise(mean = mean(!!.summary_var))
}

# Use
grouped_mean(mtcars, disp, cyl, am, vs)
#> # A tibble: 7 x 4
#> # Groups:   cyl, am [?]
#>     cyl    am    vs  mean
#>   <dbl> <dbl> <dbl> <dbl>
#> 1     4     0     1 136. 
#> 2     4     1     0 120. 
#> 3     4     1     1  89.8
#> 4     6     0     1 205. 
#> # ... with 3 more rows
```

### Quote multiple arguments

- We’ll quote the dots with `enquos()`.
- We’ll unquote-splice the quoted dots with `!!!`.

Example:
```R
grouped_mean2 <- function(.data, .summary_var, ...) {
  summary_var <- enquo(.summary_var)
  group_vars <- enquos(...)

  .data %>%
    group_by(!!!group_vars) %>%
    summarise(mean = mean(!!summary_var))
}

grouped_mean2(mtcars, disp, cyl, am)
#> # A tibble: 6 x 3
#> # Groups:   cyl [?]
#>     cyl    am  mean
#>   <dbl> <dbl> <dbl>
#> 1     4     0 136. 
#> 2     4     1  93.6
#> 3     6     0 205. 
#> 4     6     1 155  
#> # ... with 2 more rows
```

## Modifying inputs

### Modifying names

#### Default argument names

```R
args_names <- function(...) {
  vars <- enquos(..., .named = TRUE)
  names(vars)
}

args_names(mean(height), weight)
#> [1] "mean(height)" "weight"

args_names(avg = mean(height), weight)
#> [1] "avg"    "weight"
```

#### Unquoting argument names

