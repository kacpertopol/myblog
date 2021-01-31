# Introduction

The motivation for this work stems from a practical problem of writing a
*Wolfram Language* (WL) application aimed to abstract away the notion of
scalar valued functions for numerical computations. The goal was for the
user to specify, in the WL, the number of arguments for a function,
which arguments are discrete indices and which arguments are floating
point numbers. For floating point arguments the user additionally
defines points and weights used for interpolations and integrations over
these arguments and finally specifies which arguments can be used for
purposes of code parallelization. Given this input, the job of the
application is to create a *FORTRAN 90* module and a directory with
supplementary data. The user can import this module and work with an
abstraction of the scalar valued functions: store function values on
different cores and handle them using different *MPI* and *OPENMP*
threads, interpolate function values and integrate over function
arguments.

The application requires further testing but it is feature complete.
This was possible, to a large extent, due to a choice to base the
implementation around monadic types. The monadic style of programming
can also be appropriate for many other problems and in this paper a
simple approach to include monads and the *Haskell* style "do\" notation
in the WL is proposed. This combined with the powerful functions built
into the *Mathematica* [@mathematica] [@mathematica] system can lead to
the production of very powerful code.

This work was influenced by an excellent book by Bartosz Milewski
[@bmilew] containing an introduction to Category Theory. First in
section [2](#categorical_interpretation){reference-type="ref"
reference="categorical_interpretation"} a categorical interpretation of
the WL is introduced. Next, in section
[3](#monadic_types){reference-type="ref" reference="monadic_types"} the
implementation of monadic types and the "do\" notation is described. In
section [4](#examples){reference-type="ref" reference="examples"}
examples and tips for practical implementations are shown. Finally
section [5](#summary){reference-type="ref" reference="summary"} contains
a summary. The text contains snippets of WL code and pseudo-code. Some
code fragments will use comments `(*...*)` to hold additional
information intended for interpretation by the reader.

# Categorical interpretation of the *Wolfram Language* {#categorical_interpretation}

In order to think about the WL as a category a subset of the WL will be
considered consisting of functions defined in the following manner:

    f[(*pattern a*)] := (*expression*)

where *f* stands for the function name (other symbols can also be used),
*pattern a* is a WL language pattern expression that specifies the
arguments accepted by the function *f* and *expression* is a WL
expression that can be constructed from the named parts of *pattern a*.
This is not directly enforced by the WL, but ideally all possible
results of *f* match a different pattern, *pattern b*, so that:

    MatchQ[f[(*expression matching pattern a*)],(*pattern b*)]

is always true. Curryied functions can be used and this can be
interpreted as replacing `f` with `f[x]`, `f[x][y]`, ... in the pseudo
code above and treating the replacement as a whole set of functions
built from the different values of `x`, `y`, ...

It is seems natural to treat this subset of the WL as a category with
morphisms being functions (e.g. `f`, `f[x]`, `f[x][y]`, ...) and objects
being types defined by WL patterns (e.g. the set of all WL expressions
that match *pattern a* defines type *a* and the set of expressions that
match *pattern b* defines type *b*). In this context a monad is a type
function: it takes a pattern that defines one type and returns a new
pattern that defines a different type. Some additional definitions are
necessary and they discussed in the next section.

# Introduction of monadic types and the "do\" notation {#monadic_types}

A user can implement a monad *m* in the WL by supplying three
definitions. The first definition:

    pattern[m] = (*pattern m a*)

provides a pattern that matches a WL expression if it is an *m* monad
for any type argument *a*. The second definition implements the monadic
*return* function:

    return[m][x_] := (*expression matching pattern[m]*)

if the type of *x* is *a* then this expression should be a *m* monad for
type argument *a*. In particular the resulting WL expression should
match `pattern[m]`. The final definition is an implementation of the
monadic *bind* operator for *m*:

    bind[m][ma_ , aTOmb_] := (*implementation of bind*)

where *ma* is an *m* monad for type argument *a* and *aTOmb* is a
function taking an element of type *a* and returning an *m* monad for
type argument *b*. The user does not have to supply additional pattern
constraints in `bind`, however it is up to the user to make sure that
these definitions result in an object that satisfies the monad laws.

Having these three definitions for monad *m*, the implementation of the
"do\" notation in the WL is straightforward and consists of two parts.
The first is a recursive definition that reduces the `do[m]` expression
by applying the user defined `bind[m]` operator:

    do[m_][x___ , y_ , r:Except[_LeftArrow]]:= 
       If[MatchQ[y , LeftArrow[_ , _]] , 
          do[m][x , 
             chk[m][
             bnd[m][snd[y] , 
                Function[Evaluate[fst[y]] , r]]]
             ],
          do[m][x , 
             chk[m][bnd[m][y , 
                Function[Unique["nonExistantVariable"] , r]]]
          ]
       ];

Here `bnd[m]` will be replaced by the appropriate user supplied
`bind[m]` function when the first argument matches `pattern[m]` and the
second argument is a function, `fst`, `snd` are helper functions that
take the left and right expression from a `LeftArrow` statement, and
finally `chk[m]` is a function that checks the result of `bind[m]`
against the user defined `pattern[m]`. Once `do[m]` is reduced to a
contain a single expression the final expression is checked and
returned:

    do[m_][mx_]:=chk[m][mx];

These definitions together with the supplementary functions are provided
for the readers convenience at [@repos]. This repository contains a
short WL package *wlmonad.wl* and a number of examples.

# Hanoi Tower example {#examples}

This section takes a look at the classic Hanoi Tower puzzle. The puzzle
consists of three poles onto which stacks of disks with different sizes
$1 , 2 , 3 , \ldots$ can be placed. At the beginning of the puzzle all
disks are stacked on the first pole with smaller disks being placed on
top of larger ones. The aim of the puzzle is to transport all disks from
the first pole to the third pole in discrete steps. In each step only
one disk can be taken from the top of a stack and placed on a larger
disk or on an empty pole.

The puzzle is solved in the *hanoi_tower_example.nb* notebook from
[@repos]. The notebook starts a definition of a pattern for the Hanoi
Tower puzzle:

    towers = {_List , _List , _List};

The puzzle will be represented by a list containing three elements. Each
element is a list and corresponds to a single pole of the puzzle.
Elements of this list will be numbers corresponding to disc sizes.
Another pattern is provided for expressions that will be used to record
the moves that lead to the solution:

    move = {_ -> _ , {_List , _List , _List}};

Moves will be two element lists, the first element (e.g. `1->2`) will
record the number of the pole from which the disk was taken and the
number of the pole onto which the disk was dropped (e.g. `1` , `2`). The
second element will contain the current configuration of the puzzle.

Next three definitions are provided for the *hT* monad that will be used
in a recursive solution to the puzzle. The first definition is a pattern
that will match any WL expression that is an *hT* monad for any type
argument:

    pattern[hT] = hT[towers , {move ...}];

Next the return function is defined as (please note that an empty list
also matches `{move ...}`):

    return[hT][x_] := hT[x , {}];

Finaly the bind operator for *hT* is given (please note that no
additional pattern constraints are necessary for `ma` and `aTomb`):

    bind[hT][ma_ , aTomb_] := 
       With[
          {val = aTomb[grab[ma]]}, 
          hT[grab[val] , join[rest[ma], rest[val]]]
       ];

where `grab` and `rest` are helper functions:

    grab[ma : pattern[hT]] := ma[[1]];
    rest[ma : pattern[hT]] := ma[[2]];

and `join` provides a lazy version of the `Join` function:

    join[a : {move ...} , b : {move ...}] := Join[a , b];

Providing lazy functions that evaluate only when they are called with
appropriate arguments is important for recursion.

Only one action is provided for this monad with two definitions. The
first one moves a single disk of puzzle `tower` from pole number `from`
to pole number `to`:

    moveDiscs[from_ , to_ , 1][tower : towers] :=
       Module[{newtower},
          newtower = ReplacePart[
             ReplacePart[
                tower , 
                to -> Join[tower[[from]][[1 ;; 1]], tower[[to]]]
             ] , 
             from -> tower[[from , 2 ;;]]
          ];
          hT[newtower , {{from -> to , newtower}}]
       ];

The second one moves `n` disks from pole number `from` to pole number
`to`. This time the definition uses the "do\" notation and recursion:

    moveDiscs[from_ , to_ , n_][tower : towers] :=
       do[hT][
          toOther <- moveDiscs[
                      from , 
                      other[from , to] , 
                      n - 1][tower],
          toGoal <- moveDiscs[
                      from , 
                      to , 
                      1][toOther],
          finalMove <- moveDiscs[
                         other[from , to] , 
                         to , 
                         n - 1][toGoal],
          return[hT][finalMove]
       ];

where `<-` is the `LeftArrow` operator (`<ESC><-<ESC>` in the WL) and
`other[i , j]` returns a pole number that is different then `i` and `j`.
The procedure of solving the puzzle can be read off directly from the
code above. First $n-1$ disks are moved from pole number `from` to a
pole that is different from `from` and `to`. Next, one disk is moved
from pole number `from` to pole number `to`. Finaly the $n-1$ disks are
moved from the other pole to the `to` pole.

A solution to the puzzle with three disks on the first pole in the
initial configuration can be obtained by evaluating:

    moveDiscs[1 , 3 , 3][makeTower[3]]

where `makeTower[3]` prepares the initial configuration of the puzzle.
The result is:

    hT[
       {{},{},{1,2,3}},
       {
          {1->3,{{2,3},{},{1}}},
          {1->2,{{3},{2},{1}}},
          {3->2,{{3},{1,2},{}}},
          {1->3,{{},{1,2},{3}}},
          {2->1,{{1},{2},{3}}},
          {2->3,{{1},{},{2,3}}},
          {1->3,{{},{},{1,2,3}}}
       }
    ]

The first element of this expression is the final configuration of the
solved puzzle. The second element is a list containing the moves used to
obtain the solution.

The repository [@repos] more examples including a *list* monad and a
*maybe* monad. The reader is encouraged to download and explore these
notebooks.

# Summary

The *Mathematica* [@mathematica] system has a very powerful set of built
in functions. The flexibility of the *Wolfram Language* makes it
possible to program in many different styles. Such freedom can, however,
be overwhelming and it is a good idea to conform to styles of
programming that are tailored to particular problems. Combining the
monadic style of programming with the powerful standard library of the
*Wolfram Language* can be very useful for problems that require a
functional solution. There are many ways of implementing this style of
programming in the *Wolfram Language*, the author belives that the
method presented in this paper is relatively simple and robust.

# Acknowledgmnets {#acknowledgmnets .unnumbered}

The project was financed from the resources of the National Science
Center, Poland, under Grants No.2016/22/M/ST2/00173 and No.
2016/21/D/ST2/01120.

9

Wolfram Research, Inc., Mathematica, Version 12.1, Champaign, IL (2020).
"Category Theory For Programmers\", Bartosz Milewski, 2019
(<https://www.blurb.com/b/9621951-category-theory-for-programmers-new-edition-hardco>)
<https://gitlab.com/kacpertopolnicki/wlmonad>
