---
title : my blog
author : <center><a href="mailto:kacpertopol@gmail.com">kacper topolnicki</a></br><a href="mailto:kacpertopol@gmail.com">kacpertopol@gmail.com</a><center>
inv : " bloglight.html"
---


# [infinite matrices in *Mathematica*, part 3](./2021-01-19_gen_dark.html)
<center>
[*2021-01-19*](2021-01-19_gen_light.html)
</center>

[mapping, addition, multiplication](2021-01-19_gen_dark.html) <a id = "NCE" href = "2021-01-19_gen_dark.html">...</a>



The <a id = "NCE" href = https://kacpertopol.github.io/myblog/2021-01-16_gen_light.html> first post</a>
in this series introduced the idea of implementing infinite matrices in the *Wolfram Language*. 
<a id = "NCE" href = https://kacpertopol.github.io/myblog/2021-01-17_gen_light.html>Next</a>
we defined a type pattern to represent infinite matrices and implemented the transposition
operation. This time we will write three more operations. The mapping function will 
allow us to map arbitrary functions over the elements of the matrix. Finally we will
define the matrix - matrix
addition and matrix - matrix multiplication functions.

First mapping. This is straightforward:
```Mathematica
mapiM[f_ (*1*)][im : iM (*2*)]  := 
 Module[{nofRows , nofCols , rowFunction , colFunction},
  {nofRows , nofCols , rowFunction , colFunction} = im (*3*);
  {
		nofCols , 
		nofRows , 
  		Function[row, 
			{#[[1]] , #[[2]] , f[#[[3]]]} & /@ rowFunction[row]] (*4*),
  		Function[col, 
			{#[[1]] , #[[2]] , f[#[[3]]]} & /@ colFunction[col]] (*5*) 
  	}
  ]
```
There are two arguments. The first argument, marked 1 above, is the function that
will be applied to each element of the matrix. The second argument, marked 2,
is the matrix itself. In 3 the matrix components are unpacked to local variables.
The mapping operation is performed inside each non zero component of the matrix 
in 4 and 5 - `#[[3]]` is the value of the matrix element and in `f[#[[3]]]` 
the user supplied function is applied to this element.

To see how this works in action we can evaluate:
```Mathematica
matrixFormiM[{1 , 5} , {1 , 5}][mapiM[Function[el , -el]][tMat]]
```
where `tMat` is the $P$ matrix from 
<a id = "NCE" href = https://kacpertopol.github.io/myblog/2021-01-16_gen_light.html>part 1</a>
and was defined in *Mathematica* in 
<a id = "NCE" href = https://kacpertopol.github.io/myblog/2021-01-17_gen_light.html>part 2</a>.
The result:
$$
\left(
\begin{array}{cccccccc}
 p-1 & -p & 0 & 0 & 0 & \cdot  & \cdot  & \cdot  \\
 0 & p-1 & -p & 0 & 0 & \cdot  & \cdot  & \cdot  \\
 0 & 0 & p-1 & -p & 0 & \cdot  & \cdot  & \cdot  \\
 0 & 0 & 0 & p-1 & -p & \cdot  & \cdot  & \cdot  \\
 0 & 0 & 0 & 0 & p-1 & \cdot  & \cdot  & \cdot  \\
 \cdot  & \cdot  & \cdot  & \cdot  & \cdot  & \cdot  & \cdot  & \cdot  \\
 \cdot  & \cdot  & \cdot  & \cdot  & \cdot  & \cdot  & \cdot  & \cdot  \\
 \cdot  & \cdot  & \cdot  & \cdot  & \cdot  & \cdot  & \cdot  & \cdot  \\
\end{array}
\right)
$$
 
We have two more operations to go. First matrix - matrix addition. We will begin
by defining a helper function `addsame`. It will take as argument
a list containing
all non zero elements in both of the matrices we want to add - for a given row (or column): 
```Mathematica
addsame[{a___ , {r_ , c_ , v1_} , b___ , {r_ , c_ , v2_} , d___}] := 
 addsame[{a , b , d , {r , c , v1 + v2}}]; (*1*)
addsame[l_] := l; (*2*)
```
The definition uses recursion and is divided into two parts.
The function pattern marked 1 above is more specific and acts only when two matrix
elements have the same row and column number. If this is the case then the 
values of the matrix elements are added `v1 + v2`. The second definition,
marked 2, is more
general and will act only when no two elements have the same row and column numbers.

This helper function is part of the implementation of `addiM` - a function adding two
infinite matrices togather:
```Mathematica
addiM[im1 : iM , im2 : iM] (*1*):=
  Module[
   {nofRows1 , nofCols1 , rowFunction1 , colFunction1,
    nofRows2 , nofCols2 , rowFunction2 , colFunction2 , newFunRow , 
    newFunCol},
   {nofRows1 , nofCols1 , rowFunction1 , colFunction1} = im1;
   {nofRows2 , nofCols2 , rowFunction2 , colFunction2} = im2;
   If[nofRows1 != nofRows2 , 
    Throw["The number of rows does not match in addiM"]]; (*2*)
   If[nofCols1 != nofCols2 , 
    Throw["The number of columns does not match in addiM"]]; (*3*)
   newFunRow = Function[row , 
     addsame[Join[rowFunction1[row] , rowFunction2[row]]]
     ]; (*4*)
   newFunCol = Function[column , 
     addsame[Join[colFunction1[column] , colFunction2[column]]]
     ]; (*5*)
   newiM[nofRows1 , nofCols2, newFunRow, newFunCol]
   ];
```
The arguments, marked 1 above, are two infinite matrices.
Next the elements of the matrices are unpacked into local variables
and in 2 and 3 the sizes of the matrices are checked. Failure at this point
will result in an exception being thrown.
In 4 and 5 above - new row and column functions are created. `addsame` is being
used in conjunction with `Join` 
to concatenate the non zero elements in a given row (marked 4) or column (marked 5) of
infinite matrices 
`im1` and `im2`. To see all of this in action try evaluating:
```Mathematica
matrixFormiM[{1 , 5} , {1 , 5}][
 addiM[mapiM[Function[el , -el]][tMat] , tMat]]
```
We can expect a null matrix ... and we get:
$$
\left(
\begin{array}{cccccccc}
 0 & 0 & 0 & 0 & 0 & \cdot  & \cdot  & \cdot  \\
 0 & 0 & 0 & 0 & 0 & \cdot  & \cdot  & \cdot  \\
 0 & 0 & 0 & 0 & 0 & \cdot  & \cdot  & \cdot  \\
 0 & 0 & 0 & 0 & 0 & \cdot  & \cdot  & \cdot  \\
 0 & 0 & 0 & 0 & 0 & \cdot  & \cdot  & \cdot  \\
 \cdot  & \cdot  & \cdot  & \cdot  & \cdot  & \cdot  & \cdot  & \cdot  \\
 \cdot  & \cdot  & \cdot  & \cdot  & \cdot  & \cdot  & \cdot  & \cdot  \\
 \cdot  & \cdot  & \cdot  & \cdot  & \cdot  & \cdot  & \cdot  & \cdot  \\
\end{array}
\right)
$$
... a null matrix.

Now for the final function in our infinite matrix implementation - matrix multiplication. 
Here is the code:
```Mathematica
muliM[im1:iM , im2:iM]:=
	Module[
		{nofRows1 , nofCols1 , rowFunction1 , colFunction1,
		nofRows2 , nofCols2 , rowFunction2 , colFunction2 , 
		newFunRow , newFunCol},
		{nofRows1 , nofCols1 , rowFunction1 , colFunction1} = im1;
		{nofRows2 , nofCols2 , rowFunction2 , colFunction2} = im2;
		newFunRow (*1*) = Function[row , 
			Module[
				{allElements , columns},
				allElements = Flatten[timesel[#,rowFunction2[#[[2]]]]&/@rowFunction1[row] , 1]; (*3*)
				columns = Union[allElements[[;; , 2]]];
				Fold[{#1[[1]] , #1[[2]] , #1[[3]] + #2[[3]]}& , #]&/@(Cases[allElements , {_ , # , _}]&/@columns) (*4*)
			]
		];
		newFunCol (*2*) = Function[column , 
			Module[
				{allElements , rows},
				allElements = Flatten[lesemit[#,colFunction1[#[[1]]]]&/@colFunction2[column] , 1];
				rows = Union[allElements[[;; , 1]]];
				Fold[{#1[[1]] , #1[[2]] , #1[[3]] + #2[[3]]}& , #]&/@(Cases[allElements , {# , _ , _}]&/@rows)
			]
		];
		newiM[nofRows1 , nofCols2,newFunRow,newFunCol] (*3*)
	];
```
The approach is similar to addition. In 1 and 2 above we create two new functions `newFunRow` and `newFunCol`. These
two functions will be plugged into the constructor for the new matrix in 3. 

Let's first have a look at `newFunRow` and start with the definition of matrix multiplication. If we multiply two
matrices $A$ and $B$ to get a matrix $C$ then it's elements are:
$$
C_{r c} = \sum_{k} A_{r k} B_{k c}
$$
So, for each r-th row of $A$ 
we will need all k-th rows of B.
Using our notation for non zero matrix elements (nested lists, inner lists contain the row number, the element number
and the value of the matrix element) then the result can be written as:
$$
\text{row from A} = \{\{r , c_{1} , a_{r c_{1}}\} , \{r , c_{2} , a_{r c_{2}}\} , \ldots\}
$$
If $r =$ `row` then this is calculated in `rowFunction1[row]` (marked 3 above).
If $k =$ `#[[2]]` then all k-th rows are calculated in `rowFunction2[#[[2]]]` in 3 above. For `#[[2]]`
$ = c_1$ and using our notation for non zero matrix elements, the $c_1$-th row can be written as:
$$
\text{row from B} = \{\{c_{1} , c'_{1} , b_{c_{1} c'_{1}}\} , \{c_{1} , c'_{2} , b_{c_1 c'_{2}}\} , \ldots\}
$$ 
Next comes `timesel`:
```Mathematica
timesel[
	{r1_Integer?Positive , c1_Integer?Positive , v1_} , 
   	row : {{_Integer?Positive , _Integer?Positive , _} ...}] := 
		{r1 , #[[2]] , v1 #[[3]]} & /@ row;
```
All this function does is multiply appropriate elements in each row making sure that the row and column
of the multiplied element is correct. Finally, the resulting list `allElements` is folded and the 
matrix elements summed up (see 4 in `muliM` above).

This is a lot to take in but it is strightforward once you disect the expressions in these functions.
The situation with `newFunCol` is analogous. The rows and columns are carefully swapped and instead 
of `timesel` there is:
```Mathematica
lesemit[
	{r1_Integer?Positive , c1_Integer?Positive , v1_} , 
   	column : {{_Integer?Positive , _Integer?Positive , _} ...}] := 
		{#[[1]] , c1 , v1 #[[3]]} & /@ column;
```

That's it. <a id = "NCE" href = 2021-01-17/infiniteMatrix.nb>Here is the notebook</a>. I might turn this into a *Wolfram Language* package. When
that time comes I'll provide a link below in an EDIT, maybe with some more examples.



# [infinite matrices in *Mathematica*, part 2](./2021-01-17_gen_dark.html)
<center>
[*2021-01-17*](2021-01-17_gen_light.html)
</center>

representation, transposition <a id = "NCE" href = "2021-01-17_gen_dark.html">...</a>



# [infinite matrices in *Mathematica*, part 1](./2021-01-16_gen_dark.html)
<center>
[*2021-01-16*](2021-01-16_gen_light.html)
</center>

introduction, motivation <a id = "NCE" href = "2021-01-16_gen_dark.html">...</a>



# [hi](./2021-01-15_gen_dark.html)
<center>
[*2021-01-15*](2021-01-15_gen_light.html)
</center>

first post, what this blog is about <a id = "NCE" href = "2021-01-15_gen_dark.html">...</a>


