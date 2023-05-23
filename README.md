# FastSymbolicDifferentiation

[![Build Status](https://github.com/brianguenter/FastSymbolicDifferentiation.jl/actions/workflows/CI.yml/badge.svg?branch=main)](https://github.com/brianguenter/FastSymbolicDifferentiation.jl/actions/workflows/CI.yml?query=branch%3Amain)


FastSymbolicDifferentiation (**FSD**) is a package for generating efficient executables to evaluate derivatives. It can also generate efficient true symbolic derivatives for symbolic analysis.

There are two primary use cases where you should consider FastSymbolicDifferentiation:
* When you need the fastest possible executable for evaluating the derivative of a function and the overhead of the symbolic processing time is swamped by evaluation time.
* When you need to do other symbolic processing in addition to differentiation. **FSD** can generate a true symbolic derivative to be processed further in Symbolics.jl or another computer algebra system (CAS).

**FSD** symbolic processing time is generally much shorter than a typical CAS. For expressions with a few thousand operations it can be several orders of magnitude faster. The larger and more complex the expression the better the relative performance. The executables generated by **FSD** can also be much faster than a typical CAS, and can approach the efficiency of hand written derivatives[^2].

This is **beta** software being modified on a daily basis. Expect bugs and frequent, possibly breaking changes, over the next month or so. 

This functionality is coded and tested:
* dense and sparse symbolic Jacobian, dense symbolic Hessian
* compiled executable, in place and out of place, for dense Jacobian
* higher order derivatives

This is not yet coded:
* sparse symbolic Hessian
* compiled sparse Jacobian, sparse Hessian, dense Hessian

# How it works
The **FSD** symbolic differentiation algorithm is related to the [D* ](https://www.microsoft.com/en-us/research/publication/the-d-symbolic-differentiation-algorithm/) algorithm but is asymptotically faster so it works on much larger expression graphs. 

**FSD** transforms the input expression graph into a derivative graph and then factors this derivative graph to generate an efficient expression for the derivative. This is fundamentally different from forward and reverse automatic differentiation. See the [D* ](https://www.microsoft.com/en-us/research/publication/the-d-symbolic-differentiation-algorithm/) paper if you want to understand derivative graph factorization concepts. The algorithms used in **FSD** will be described in a soon to be written paper.

If your function is small or tree like (where each node in the expression graph has one parent on average) then Symbolics.jl may outperform **FSD**. For more complex functions with many common subexpressions it is likely that **FSD** will outperform Symbolics.jl, perhaps substantially (see benchmarks, below).

**FSD** can be used standalone if all you need is a derivative or in combination with Symbolics.jl if you need to do further analysis on the symbolic derivative. Converting between Symbolics.jl and **FSD** symbolic forms is straightforward. However, because of the tree based representation used by Symbolics.jl expression size can grow significantly when converting from **FSD** to Symbolics.jl form.

Unlike forward and reverse automatic differentiation you don't have to choose which differentiation algorithm to use based on the graph structure. **FSD** automatically generates efficient derivatives for arbitrary function types: ℝ¹->ℝ¹, ℝ¹->ℝᵐ, ℝⁿ->ℝ¹, and ℝⁿ->ℝᵐ, m≠1,n≠1. 

The efficiency of **FSD** comes from analysis of the graph structure of the function rather than sophisticated algebraic simplification rules. By default **FSD** applies only these algebraic simplications[^1] to expressions:
* x×0=>0
* x×1=>x
* x/1=>x
* x+0=>x
* c₁×c₂=>c₃ for c₁,c₂,c₃ constants
* c₁+c₂=>c₃ for c₁,c₂,c₃ constants
* c₁×(c₂×x) => (c₁×c₂)×x  for c₁,c₂ constants

These rules are generally safe in the sense of obeying IEEE floating point arithmetic rules. However if the runtime value of x happens to be NaN or Inf the **FSD** expression x*0 will identically return 0, because it will have been rewritten to 0 by the simplification rules. The expected IEEE result is NaN.

If you use FSD in your work please share the functions you differentiate with me. I'll add them to the benchmarks. The more functions available to test the easier it will be for others to determine if FSD is likely to help them with their problem.
<details> 
 <summary> Examples and basic usage </summary>
 
There are several ways to use FastSymbolicDifferentiation. You can do all your symbolic work, except differentiation, in Symbolics and then convert to **FSD** graph form just to do the differentiation, then convert back to Symbolics.jl form. Or you can do everything in **FSD**: create **FSD** variables, make an expression using those variables and then differentiate it. Creating the expressions in Symbolics.jl and then converting to **FSD** form can be slow.

Converting the other way can be slow because of the tree expression representation used by Symbolics.jl. FastSymbolicDifferentiation uses a graph representation it is possible that converting from FastSymbolicDifferentiation->Symbolic could result in an exponential increase in the size of the expression.

If all you need is an executable derivative function then the fastest workflow will be to do everything in **FSD**. 
 
**FSD** uses a global cache for common subexpression elimination so **FSD** is not thread safe (yet). Under ordinary conditions the memory used by the cache won't be an issue. But, if you have a long session where you are creating many complex functions it is possible the cache will use too much memory. If this happens call the function `clear_cache`after you have completely processed your expression.

Set up variables:
```
using FastSymbolicDifferentiation
using Symbolics

@variables x y z

julia> nx,ny,nz = Node.((x,y,z)) #create FastSymbolicDifferentiation variables.
(x, y, z)
```
FSD requires objectid consistency of vector variable elements but the vectors created by the Symbolics @variables macro do not satisfy this property:
 ```
 julia> @variables k[1:3]
1-element Vector{Symbolics.Arr{Num, 1}}:
 k[1:3]

julia> k[1] === k[1]
false
```
 As a temporary workaround you can use the `make_variables` function to create a vector of variables:
 ```
julia> X = make_variables(:x,3)
3-element Vector{Node}:
 x1
 x2
 x3

julia> X[1] === X[1]
true
```
I'm working with the SciML folks to figure out how to make FSD work seamlessly with Symbolics, but the two systems use different representations of expressions (FSD uses directed acyclic graphs and Symbolics uses trees). This difference has far reaching architectural effects so it might take some time to figure out the best path.
 
Compute Hessian:
```
julia> hessian(nx^2+ny^2+nz^2,[nx,ny,nz])
3×3 Matrix{Node}:
 2    0.0  0.0
 0.0  2    0.0
 0.0  0.0  2

 julia> hessian(nx*ny*nz,[nx,ny,nz])
 3×3 Matrix{Node}:
  0.0  z    y
  z    0.0  x
  y    x    0.0
```
Compute Jacobian:
```
julia> nx, ny = Node.((x, y))
(x, y)

julia> f1 = cos(nx) * ny
(cos(x) * y)

julia> f2 = sin(ny) * nx
(sin(y) * x)

julia> symb = symbolic_jacobian([f1, f2], [nx, ny]) #non-destructive
2×2 Matrix{Node}:
 (y * -(sin(x)))  cos(x)
 sin(y)           (x * cos(y))
```
Generate executable function that evaluates derivative function:
```
julia> func = jacobian_function([f1, f2], [nx, ny]);

julia> func(1.0, 2.0)
2×2 Matrix{Float64}:
 -1.68294    0.540302
  0.909297  -0.416147
```
Convert between FastSymbolicDifferentiation and Symbolics representations:
```
julia> f = x^2+y^2 #Symbolics expression
x^2 + y^2

julia> Node(f) #convert to FastSymbolicDifferentiation form
x^2 + y^2

julia> typeof(ans)
Node{SymbolicUtils.BasicSymbolic{Real}, 0}

julia> node_exp = nx^3/ny^4 #FastSymbolicDifferentiation expression
((x ^ 3) / (y ^ 4))

julia> dag_to_Symbolics_expression(node_exp)
(x^3) / (y^4)

julia> typeof(ans)
Symbolics.Num
```
</details>

<details>
    <summary> Benchmarks </summary>
 
## Benchmarks

The **FSDBenchmark** subdirectory has several benchmark functions you can use to compare the performance of [Symbolics.jl](https://symbolics.juliasymbolics.org/dev/) to FastSymbolicDifferentiation.jl on your computer. See the README.md file in this subdirectory for a brief overview of how to run the benchmarks yourself. See the source in `FSDBenchmarks.jl` for more details. Look for the function `benchmark_package`.
 
There are three types of benchmarks: **Symbolic**, **MakeFunction**, and **Exe**.

* The **Symbolic** benchmark is the time required to compute just the symbolic form of the derivative. The Symbolic benchmark can be run with simplification turned on or off for Symbolics.jl. If simplification is on then computation time can be extremely long but the resulting expression might be simpler and faster to execute.

* The **MakeFunction** benchmark is the time to generate a Julia Expr from an already computed symbolic derivative and to then compile it.

* The **Exe** benchmark measures just the time required to execute the compiled function using an in-place matrix.

All benchmarks show the ratio of time taken by Symbolics.jl to FastSymbolicDifferentiation.jl. Numbers greater than 1 mean FastSymbolicDifferentiation is faster.

All benchmarks were run on an AMD Ryzen 9 7950X 16-Core Processor with 32GB RAM running Windows 11 OS, Julia version 1.9.0.
### Chebyshev polynomial
The first example is a recursive function for 
the Chebyshev polynomial of order n:

```
@memoize function Chebyshev(n, x)
    if n == 0
        return 1
    elseif n == 1
        return x
    else
        return 2 * (x) * Chebyshev(n - 1, x) - Chebyshev(n - 2, x)
    end
end
```
The function is memoized for efficiency. 

The Chebyshev expression graph does not have many nodes even at the largest size tested (graph size increases linearly with Chebyshev order). For example, here is the graph of the 10th order expression: 
<img src="Documentation/Paper/illustrations/chebyshev10.svg" alt="drawing" height="400">
The complexity arises from the number of different paths from the root to the leaf of the graph.

The first set of three benchmarks show results with simplification turned off in Symbolics.jl, followed by a set of three with simplification turned on. Performance is somewhat better in the latter case but still slower than the FSD executable. Note that the y axis is logarithmic.

#### Chebyshev benchmarks with simplification off
<img src="FSDBenchmark\Data\figure_chebyshev_Symbolic_simplify_false.svg" alt="drawing" width="50%"> 
<img src="FSDBenchmark\Data\figure_chebyshev_MakeFunction_simplify_false.svg" alt="drawing" width="50%"> 
<img src="FSDBenchmark\Data\figure_chebyshev_Exe_simplify_false.svg" alt="drawing" width="50%">



#### Chebyshev benchmarks with simplification on
<img src="FSDBenchmark\Data\figure_chebyshev_Exe_simplify_true.svg" alt="drawing" width="50%">

With simplification on performance of the executable derivative function for Symbolics.jl is slightly better than with simplification off. But simplification processing time is longer.
 
### Spherical Harmonics

The second example is the spherical harmonics function. This is the expression graph for the spherical harmonic function of order 8:
<img src="Documentation/Paper/illustrations/sphericalharmonics_8.svg" alt="drawing" width="100%">

<details>
    <summary> Source for spherical harmonics benchmark </summary>

```
@memoize function P(l, m, z)
    if l == 0 && m == 0
        return 1.0
    elseif l == m
        return (1 - 2m) * P(m - 1, m - 1, z)
    elseif l == m + 1
        return (2m + 1) * z * P(m, m, z)
    else
        return ((2l - 1) / (l - m) * z * P(l - 1, m, z) - (l + m - 1) / (l - m) * P(l - 2, m, z))
    end
end
export P

@memoize function S(m, x, y)
    if m == 0
        return 0
    else
        return x * C(m - 1, x, y) - y * S(m - 1, x, y)
    end
end
export S

@memoize function C(m, x, y)
    if m == 0
        return 1
    else
        return x * S(m - 1, x, y) + y * C(m - 1, x, y)
    end
end
export C

function factorial_approximation(x)
    local n1 = x
    sqrt(2 * π * n1) * (n1 / ℯ * sqrt(n1 * sinh(1 / n1) + 1 / (810 * n1^6)))^n1
end
export factorial_approximation

function compare_factorial_approximation()
    for n in 1:30
        println("n $n relative error $((factorial(big(n))-factorial_approximation(n))/factorial(big(n)))")
    end
end
export compare_factorial_approximation

@memoize function N(l, m)
    @assert m >= 0
    if m == 0
        return sqrt((2l + 1 / (4π)))
    else
        # return sqrt((2l+1)/2π * factorial(big(l-m))/factorial(big(l+m)))
        #use factorial_approximation instead of factorial because the latter does not use Stirlings approximation for large n. Get error for n > 2 unless using BigInt but if use BigInt get lots of rational numbers in symbolic result.
        return sqrt((2l + 1) / 2π * factorial_approximation(l - m) / factorial_approximation(l + m))
    end
end
export N

"""l is the order of the spherical harmonic. I think"""
@memoize function Y(l, m, x, y, z)
    @assert l >= 0
    @assert abs(m) <= l
    if m < 0
        return N(l, abs(m)) * P(l, abs(m), z) * S(abs(m), x, y)
    else
        return N(l, m) * P(l, m, z) * C(m, x, y)
    end
end
export Y

SHFunctions(max_l, x::Node, y::Node, z::Node) = SHFunctions(Vector{Node}(undef, 0), max_l, x, y, z)
SHFunctions(max_l, x::Symbolics.Num, y::Symbolics.Num, z::Symbolics.Num) = SHFunctions(Vector{Symbolics.Num}(undef, 0), max_l, x, y, z)

function SHFunctions(shfunc, max_l, x, y, z)
    for l in 0:max_l-1
        for m in -l:l
            push!(shfunc, Y(l, m, x, y, z))
        end
    end

    return shfunc
end
export SHFunctions

function spherical_harmonics(::JuliaSymbolics, model_size)
    Symbolics.@variables x y z
    return SHFunctions(model_size, x, y, z), [x, y, z]
end

function spherical_harmonics(::FastSymbolic, model_size, x, y, z)
    nx = Node(x)
    ny = Node(y)
    nz = Node(z)

    graph = DerivativeGraph(SHFunctions(model_size, nx, ny, nz))
    return graph
end

function spherical_harmonics(package::FastSymbolic, model_size)
    Symbolics.@variables x, y, z
    return spherical_harmonics(package, model_size, x, y, z)
end
export spherical_harmonics
```
</details>

As was the case for Chebyshev polynomials the number of paths from the roots to the variables is much greater than the number of nodes in the graph. Once again the y axis is logarithmic.

<img src="FSDBenchmark\Data\figure_spherical_harmonics_Symbolic_simplify_false.svg" alt="drawing" width="50%">
<img src="FSDBenchmark\Data\figure_spherical_harmonics_MakeFunction_simplify_false.svg" alt="drawing" width="50%">
<img src="FSDBenchmark\Data\figure_spherical_harmonics_Exe_simplify_false.svg" alt="drawing" width="50%">
 
 The **Exe** benchmark took many hours to run and was stopped at model size 24 instead of 25 as for the **Symbolic** and **MakeFunction** benchmarks.

</details>

## Future work
The **FSD** algorithm is fast enough to differentiate large expression graphs (≈10⁵ operations) but LLVM compile time can be significant for graphs larger than this. For these very large graphs [DynamicExpressions.jl](https://github.com/SymbolicML/DynamicExpressions.jl) might be a better tradeoff between compile and execution time. I will be experimenting with this over the coming months. This would be useful when your function is changing frequently so compilation overhead cannot be amortized across many derivative evaluations.

The code currently uses BitVector for tracking reachability of function roots and variable nodes. This seemed like a good idea when I began and thought **FSD** would only be practical for modest size graphs (<10⁴ nodes). Unfortunately, for larger graphs the memory overhead of the BitVector representation becomes significant. It should be possible to automatically detect when it would make sense to switch from BitVector to Set. Then it will be possible to differentiate significantly larger graphs.

In its current form **FSD** can only differentiate symbolic expressions without branches. It is possible extend the algorithm to allow branching but this causes symbolic processing time to scale exponentially with the nesting depth of conditionals. For small nesting depths this might be acceptable so a future version of FSD might support limited nested conditionals. 

However, a better approach might be to use FSD as a processing step in a tracing JIT compiler, applying **FSD** to the basic blocks detected and compiled by the JIT. These basic blocks do not have branches. Many programs could be differentiated competely automatically by this method. I'm not a compiler expert so it is unlikely I will do this by myself. But contact me if *you* are a compiler expert and want a cool project to work on.

[^1]: More rules may be added in future versions of FSD to improve efficiency.
[^2]: See the D* paper for an example of an automatically generated derivative for Lagrangian dynamics that is comparable in efficiency to a hand written derivative.
