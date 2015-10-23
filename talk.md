<script type="text/javascript"
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>
# Introduction to profiling

## Olav Vahtras

Computational Python

---

layout: false

## Profiling

As python programs can be potentially slow it is important to be able to identify the bottlenecks of the code

* Example: For large symmetric matrices we may save space with packed triangular storage
* However for matrix multiplications it is more convenient with full square matrices
* Write a function that takes a triangular packed matrix and convert it to a square

---

* Get ``N`` from packed dimension ``N(N+1)/2``

```
    def get_square_dim(sp):
        from math import sqrt
        nn, = sp.shape
        # nn = n(n+1)/2 -> n = -1/2 + sqrt(1/4 + 2*nn)
        n = int(round(-0.5 + sqrt(0.25 + 2*nn)))
        # allocate a square matrix and fill elements
        return n
```
---

* From triangular to square


```
    def unpack(n, sp):
        sq = numpy.zeros((n, n))
        ij = 0
        for i in range(n):
            for j in range(i):
                sq[i, j] = sq[j, i] = sp[ij]
                ij += 1
            sq[i, i] = sp[ij]
            ij += 1
        return sq
```    

---
* The main program


```
    def main(sp):
        n = get_square_dim(sp)
        sq = unpack(n, sp)
        return sq
               
```
--
```
    >>> sp = numpy.random.rand(3)
    >>> print sp
    [ 0.39333476  0.99692927  0.41525095]
```
--
```
    >>> sq = main(sp)
    >>> print sq
    [[ 0.39333476  0.99692927]
     [ 0.99692927  0.41525095]]
```
---

### The ``profile`` module

Python provides a ``profile`` module for timing

```
    >>> from profile import run
    >>> n=10000; nn=n*(n+1)//2
```
--
```
    >>> run('main(numpy.random.rand(nn))')
             2011 function calls in 3.196 seconds

       Ordered by: standard name
       ncalls  tottime  percall  cumtime  percall filename:lineno(function)
            1    0.112    0.112    0.112    0.112 :0(rand)
         2001    0.088    0.000    0.088    0.000 :0(range)
            1    0.000    0.000    0.000    0.000 :0(round)
            1    0.004    0.004    0.004    0.004 :0(setprofile)
            1    0.000    0.000    0.000    0.000 :0(sqrt)
            1    0.040    0.040    0.040    0.040 :0(zeros)
            1    0.008    0.008    3.192    3.192 <string>:1(<module>)
            1    0.000    0.000    3.196    3.196 profile:0(main(numpy.random.rand(nn)))
            0    0.000             0.000          profile:0(profiler)
            1    2.944    2.944    3.072    3.072 profiling.py:11(unpack)
            1    0.000    0.000    3.072    3.072 profiling.py:22(main)
            1    0.000    0.000    0.000    0.000 profiling.py:3(get_square_dim)
```
---

### analyis

* The most time is spent in the unpack routine
* This is a candidate for rewriting
* Consider a Fortran version of ``unpack``

---
```

           SUBROUTINE UNPACK(N, SP, SQ)
           DOUBLE PRECISION SP(*), SQ(N, N)
    Cf2py intent(in) n, sp
    Cf2py intent(out) sq

           IJ = 1
           DO I=1, N
               DO J=1, I-1
                   SQ(I, J) = SP(IJ)
                   SQ(J, I) = SP(IJ)
                   IJ = IJ + 1
               END DO
               SQ(I, I) = SP(IJ)
               IJ = IJ + 1
           END DO
           RETURN
           END

```
---

### Compile to a python module


```
    $ f2py -m _unpackf unpack.F 
    Reading fortran codes...
	    Reading file 'unpack.F' (format:fix,strict)
    Post-processing...
	    Block: _unpackf
    {}
    In: :_unpackf:unpack.F:UNPACK
    vars2fortran: No typespec for argument "N".
			    Block: UNPACK
			    Block: MAIN
    Post-processing (stage 2)...
    Building modules...
	    Building module "_unpackf"...
		    Constructing wrapper function "UNPACK"...
    getarrdims:warning: assumed shape array, using 0 instead of '*'
		      UNPACK(SP,SQ,[N])
	    Wrote C/API module "_unpackf" to file "./_unpackfmodule.c"
```
--
```
    $ f2py -c -m _unpackf unpack.F  > /dev/null
```

---

### New main program


```
    def mainf(sp):
        from _unpackf import unpack
        n = get_square_dim(sp)
        sq = unpack(n, sp)
        return sq
```
---

### Run the Fortran version


```
    from profile import run
    n=2000; nn=n*(n+1)//2
```
--
```
    run('mainf(numpy.random.rand(nn))')
	     8 function calls in 0.232 seconds

       Ordered by: standard name

       ncalls  tottime  percall  cumtime  percall filename:lineno(function)
	    1    0.108    0.108    0.108    0.108 :0(rand)
	    1    0.000    0.000    0.000    0.000 :0(round)
	    1    0.004    0.004    0.004    0.004 :0(setprofile)
	    1    0.000    0.000    0.000    0.000 :0(sqrt)
	    1    0.008    0.008    0.228    0.228 <string>:1(<module>)
	    1    0.000    0.000    0.232    0.232 profile:0(mainf(numpy.random.rand(nn)))
	    0    0.000             0.000          profile:0(profiler)
	    1    0.112    0.112    0.112    0.112 profiling.py:27(mainf)
	    1    0.000    0.000    0.000    0.000 profiling.py:3(get_square_dim)
```
---

### Conclusion


* The profiling does not give interal information from the compiled version
* The time in the main/mainf programs has been reduced from 3 s to 0.1 s

---

### Profiling code snippets


* The timeit module executes a single statement $10^6$ times
* An optional setup parameter
* Report the time

```
    import timeit
    print timeit.timeit('math.sqrt(2.0)', setup='import math')
    0.288702964783
    print timeit.timeit('sqrt(2.0)', setup='from math import sqrt')
    0.2072930336
```

---

### Line profiling

Consider the script
```
#hello.py
from time import sleep

def hello():
    print "Hello"
    sleep(3)
    print "Goodbye"

hello()
```

Rather than knowing how much time is spent in the function we may want to know line-by-line what happens

---

### The `line_profiler` module

* Use the third-party package `line_profiler` to get timing statistics line-by-line
* Install with

```
$ pip install line_profiler
```


* The `line-profiler` package contains a script `kernprof` which is used to execute your
file (instead of `python`)
* `kernprof` defines a decorator which you can use to analyze the function in question

---
The steps are:

* Decorate the function you want to time with the `@profile` decorator

```python
@profile
def hello():
    print "Hello"
    sleep(3)
    print "Goodbye"
```
--
* Execute the script with the `kernprof` script

```
$ kernprof -l -v hello.py 
Hello
Goodbye
Wrote profile results to hello.py.lprof
Timer unit: 1e-06 s

Total time: 3.00322 s
File: hello.py
Function: hello at line 3

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
     3                                           @profile
     4                                           def hello():
     5         1           41     41.0      0.0      print "Hello"
     6         1      3003110 3003110.0    100.0      sleep(3)
     7         1           68     68.0      0.0      print "Goodbye"

```

---

### Summary

* `profile`  module  for function-level profiling your code
* `line_profiler` module for line-level profiling your code
* `timeit` module for timing short code snippets

Do not ever optimize your code without profiling
