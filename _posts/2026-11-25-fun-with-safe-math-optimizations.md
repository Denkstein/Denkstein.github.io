---
layout: post
title: "Fun with safe math optimizations"
date: 2026-11-25
---

GCC calls it `-funsafe-math-optimizations`. Depending on your reading, that is either a warning about certain floating point compiler options or an invitation to have fun with some splendidly safe math optimizations.

In this article I would like to dive into some of the details: What exactly do the optimizations do? Which techniques depend on strict floating-point math? What kind of code breaks when using fast-math? How much faster does your code actually become when using fast-math optimizations? And if these optimizations are unsafe, does leaving them off make your code safe?

If your code does a lot of floating-point math, the details below are worth knowing, and if not, feel free to read on anyway for a foray into the fascinating world of floating-point math.

## The switches

As a slightly incomplete overview, GCC and Clang have `-ffast-math`, while MSVC has `/fp:fast`. The two are broadly similar: neither is a single optimization, both are a bundle of permissions for the compiler.

GCC and Clang let you take the bundle apart more. Here are some of the options that `-ffast-math` encompasses:

| flag | permission |
| --- | --- |
| `-fno-math-errno` | math functions don't need to set `errno` |
| `-ffinite-math-only` | no value is ever NaN or infinite |
| `-fno-trapping-math` | nobody observes floating-point exceptions |
| `-fno-signed-zeros` | +0 and -0 are interchangeable |
| `-freciprocal-math` | `x / y` may become `x * (1/y)` |
| `-fassociative-math` | operations may be reordered and regrouped |

`-funsafe-math-optimizations` stands for the last four of these. `-ffast-math` is everything in the table, plus a few smaller items.

MSVC gives you less control. The two relevant options are `/fp:precise`, which is the default, and `/fp:fast`, which gives you most of what `-ffast-math` does.

## What strict floating-point arithmetic actually guarantees

Without these flags, the compiler follows IEEE 754 as closely as it can. Each `+`, `-`, `*`, `/` and `sqrt` returns the correctly rounded result, and the compiler keeps the operations in the order you wrote them. That is a strong guarantee, with two caveats.

### Caveat 1: correctly rounded steps don't add up to a correct result

At the risk of stating the obvious: just because every intermediate result was rounded correctly, that doesn't mean that the final result will be rounded correctly. Anyone who has seen a pie chart whose percentages add up to 99% knows the problem. The usual floating-point example is adding 0.1 ten times ([Compiler Explorer](https://godbolt.org/z/ErMqsWbY3)): the result is 0.99999999999999988898, and `sum == 1.0` is false.

### Caveat 2: compilers get very close to IEEE 754, but not all the way

The good news first: for basic arithmetic, with default architecture options on x86-64, strict mode really is reproducible. Nevertheless, portable bit-exact results for a whole program are difficult to achieve, because of cases like the following:

**Example 1: FMA.** Modern CPUs have a fused multiply-add instruction that computes `a * b + c` with a single rounding. That usually makes results more accurate, but it also makes them depend on the target: GCC and Clang use it by default, without any fast-math flag, as soon as the target architecture has it. So consider a function like this:

```cpp
double diff_of_sq(double x, double y) {
    return x * x - y * y;
}
```

When using FMA instructions, the code will first calculate `tmp = y * y`, and then use a single FMA instruction to calculate `x * x - tmp`. This means that the intermediate result of `y * y` is rounded, but the intermediate result of `x * x` is not. Consequently, when the above function is called with inputs of x = y = 0.1, the result is 0.0 with `-O3`, but -8.3·10⁻¹⁹ with `-O3 -march=native` ([Compiler Explorer](https://godbolt.org/z/e75c6njvh)).

**Example 2: x87.** Problems occur not only with modern hardware, but also with older hardware. Before SSE2, x86 did floating-point math in the x87 floating-point unit, whose registers are 80 bits wide. 32-bit builds often still use it. Intermediate results then carry more precision and a wider exponent range than a `double`. That can make results more accurate, but it also makes them depend on how the code is written. Here is the sum of 0.1 from above, once as a loop and once as a single expression ([Compiler Explorer](https://godbolt.org/z/nr3bvPqYv)):

|  | loop, `sum += v` ten times | `v + v + … + v` in one expression |
| --- | --- | --- |
| 64-bit (SSE) | 0.99999999999999988898 | 0.99999999999999988898 |
| `-m32` (x87) | 0.99999999999999988898 | 1 |

In the single expression, all ten additions happen in an 80-bit register, and the result is rounded to a double only once, at the end. In the loop, `sum += v` is an assignment, and with the settings used here, GCC rounds to a 64-bit double at every assignment. So the result of a mathematical operation depends on how you write it, as well as on whether you compile for 32 or 64 bit.

**Example 3: math functions.** IEEE 754 requires correct rounding for basic arithmetic and `sqrt`. For other functions it only recommends it, and C and C++ leave the accuracy to the implementation. Functions like `asin` or `cbrt` in particular can be off by one unit in the last place (ulp) pretty often. Just to give one example: Here is the result of `asin(1.0 / 45.0)`, calculated by different compilers with their standard libraries: [Compiler Explorer](https://godbolt.org/z/onsbqa5nG). The correctly rounded result is 0.022224051618267154. In this case, MSVC's library gives the correct result, while glibc, which GCC and Clang both use here, deviates by one ulp. But there are plenty of examples where this is reversed, or where all of them are off.

**Example 4: constant folding.** The same function with the exact same argument can even return two different results within the same program ([Compiler Explorer](https://godbolt.org/z/sP4oq3dvq)). When the argument is a constant, GCC evaluates `std::cbrt` and `std::asin` at compile time, using the MPFR library, which rounds correctly. When the argument is evaluated at runtime, the call goes to glibc, which is less accurate.

These four are only a sample. The behavior can be further altered by various subtle compiler settings. The bottom line is that bit-exact results across compilers and platforms are very hard to achieve. Projects that actually need them have to do a lot more work than merely not turning on fast-math optimizations.

## What fast-math actually does

In a nutshell, fast-math lets the compiler pretend that floating-point numbers behave like real numbers, and optimize on that basis. This sometimes makes the result less accurate, sometimes more, and sometimes it becomes something else entirely.

The clearest way to see what that means is to look at what stops working.

### No errno

According to ISO C and C++, math functions report errors through the `errno` variable. `sqrt(-1.0)` sets it to `EDOM`, `exp(1000.0)` sets it to `ERANGE`.

Almost nobody reads `errno` after a `sqrt`. But the compiler has to assume that somebody might, and that may be expensive: it can't replace a `sqrt` call with the CPU's vector square root instruction, or with a vectorized function call, because those do not set `errno`.

GCC and Clang can turn this off individually with `-fno-math-errno`, while in MSVC it is part of `/fp:fast`.

Unless you were in the habit of regularly checking `errno`, losing this costs you nothing.

### No floating-point traps

This is another rarely used error mechanism. The CPU can be set up to raise a floating-point exception when an operation is invalid, overflows or divides by zero, and a program can also query status flags afterwards. With `-fno-trapping-math` you promise the compiler that you do neither, so it can move or remove operations without worrying about their side effects.

GCC assumes trapping math by default. Clang and MSVC don't.

So this permission costs even less, and most C++ code built with Clang or MSVC has been compiled under it all along.

### No infinity, no NaN

This is the last error mechanism built into floating point. An operation that overflows returns ±infinity. An operation with invalid arguments, such as `0.0 / 0.0`, returns NaN ("not a number").

Under fast-math, the hardware still produces these values. But the compiler may assume that no argument and no result is ever NaN or infinite, and optimize accordingly.

This allows the compiler to apply simplifications that look obvious, but are wrong in strict floating point because NaN and infinity don't follow the usual rules:

| expression | fast-math simplifies it to | strict result |
| --- | --- | --- |
| `x - x` | 0 | NaN for x = inf |
| `x / x` | 1 | NaN for x = 0 |
| `x == x` | true | false for x = NaN |

Controversially, it also means that the compiler might not properly process functions designed for checking for these special values:

| check | GCC/Clang `-ffinite-math-only` | MSVC `/fp:fast` |
| --- | --- | --- |
| `isnan(x)` | false for all inputs | true for x = NaN |
| `isinf(x)` | false for all inputs | true for x = inf |

Under GCC and Clang, `isnan` and `isinf` always return false, so you have no well-defined way to check whether a given floating-point number is one of those values.

If I had to defend this design, I'd argue that returning a special value was never great error handling. A NaN at the end of a computation tells you that something failed but not which operation, or why. For that you have to validate the inputs of an operation before you perform it, and that still works under fast-math, since it's a comparison between ordinary numbers.

One thing that still works in practice is to check the underlying bit representation of the floating-point number ([Compiler Explorer](https://godbolt.org/z/5q3Y8vffq)). This does bypass the promise you made to the compiler that these values never occur, so I probably wouldn't ship these checks in production code, but for debugging they're serviceable.

In GCC and Clang this optimization has its own flag, `-ffinite-math-only`, so you can also take it back out: `-ffast-math -fno-finite-math-only`. In MSVC it's part of `/fp:fast`, although `isnan` and `isinf` keep working there.

In practice, most code only encounters these values by accident. A double overflows only above 1.8·10³⁰⁸, so code working with physical quantities, money or pixel coordinates reaches infinity only through a bug, typically a division by something that should never have been zero. NaN, likewise, comes from invalid operations such as 0/0 or the square root of a negative number. Code that knows its domain can rule these out at the inputs.

The exception is code that uses these values deliberately, for example infinity as the starting value when searching for a minimum, or NaN as a marker for missing data. Such code usually can be adjusted easily to still work under fast-math, but when not (e.g. with third-party code), taking out this optimization via `-fno-finite-math-only` can be a reasonable option.

### Signed zeros

Floating point has two zeros, +0 and -0. They compare equal, but they don't behave the same. `-0.0 + 0.0` is +0.0. So `x + 0.0` is not a no-op: for x = -0.0 it returns +0.0. (Subtracting works: `x - 0.0` always returns x.) Likewise, `0.0 - x` is not the same as `-x`, and `x * 0.0` is not always +0.

With `-fno-signed-zeros`, the compiler may ignore the distinction and simplify all of these ([Compiler Explorer](https://godbolt.org/z/xcM4M6TET)). It is also a prerequisite for further optimizations enabled by `-fassociative-math`, since distribution may change the sign of a zero: for a = -1, b = 1 and c = -1, `a * (b + c)` is -0, while `a * b + a * c` is +0. That is why GCC and Clang only accept `-fassociative-math` when `-fno-signed-zeros` is also set.

### Division by a constant

Dividing by a number is not the same as multiplying by its inverse, unless the inverse is exactly representable, as it is for powers of two. `x / 3.0` and `x * (1.0 / 3.0)` do differ for some inputs. So by default the compiler may not turn a division into a multiplication, even though the multiplication is several times faster. In [this example](https://godbolt.org/z/sP6M6dcTh), GCC compiles `x / 2.0` to a multiplication by 0.5 in both builds, but `x / 3.0` only under `-ffast-math`.

One interesting thing that correctly rounded division makes possible is emulating integer division with doubles. For non-negative integers below 2⁵³, IEEE 754 arithmetic guarantees that `floor(a / b)` is exactly the integer quotient. With `-freciprocal-math`, or MSVC's `/fp:fast`, the compiler may replace the division by a constant with a multiplication by its inverse, and the guarantee is gone:

```cpp
floor(a / 49.0)   // a = 49:   strict: 1    with reciprocal math: 0
```

1/49 rounds down slightly, 49 times that is 0.9999999999999999, and `floor` does the rest ([Compiler Explorer](https://godbolt.org/z/a7aMYdenj)).

Outside such tricks, the change is harmless. The deviations you get from this transformation are tiny, and if you need exact integer division, C/C++ already have built-in integer types for you.

### Subnormal numbers

As floating-point numbers get very small, they don't suddenly drop to zero. Below 2⁻¹⁰²² (for doubles) the normal format runs out of exponents, and the numbers continue with a fixed spacing of 2⁻¹⁰⁷⁴ down to zero, losing one bit of precision with each halving. These are the subnormal numbers. They guarantee a useful property: if two numbers are not equal, their difference is not zero.

The problem is that on many CPUs, subnormals are slow. On my machine an operation that produces one takes about 40 ns instead of 0.5 ns. CPUs therefore offer a mode that flushes subnormal results to zero and treats subnormal inputs as zero.

You can switch that mode on in code. GCC and Clang also do it for you: if you pass `-ffast-math` or `-funsafe-math-optimizations` when linking, they add a small piece of startup code that sets it. The generated code for your functions doesn't change at all.

Because this is a CPU setting, it applies to the whole program, including library code that was compiled without fast-math. Code that runs into subnormal numbers can then behave in unexpected ways.

The simplest example breaks the property from above ([Compiler Explorer](https://godbolt.org/z/49z4o98qT)). x = 1.5·2⁻¹⁰²² and y = 2⁻¹⁰²² are both normal numbers and not equal, but their difference is subnormal and gets flushed to zero. So `(x != y) ? 1 / (x - y) : 0` returns infinity, even though the check was meant to rule out exactly that.

Until recently, there was a surprising trap with shared libraries. Up to GCC 12, a shared library built with `-ffast-math` activated the same startup code, so merely loading it switched the whole program to flush-to-zero, even if the program itself was built without fast-math. GCC 13 and Clang 19 no longer add that code to shared libraries, so only the flags used to link the final program itself decide.

MSVC does not offer this as a compiler option, so here you can only set it manually in code.

Whether any of this matters depends on whether your values ever get that small. The normal range of a double reaches down to 2.2·10⁻³⁰⁸. For comparison, even the Planck length, measured in meters, is 1.6·10⁻³⁵, so physical quantities in sensible units stay nowhere near the subnormal range.

Values usually get there by decaying toward zero, as in the tail of a reverb filter or a damped simulation. Then flushing them to zero is exactly what you want: audio software commonly turns on flush-to-zero deliberately, because subnormals in decaying signals cause CPU spikes.

### Reassociation

Floating-point arithmetic doesn't follow all the algebraic rules of real numbers. `(a + b) + c` can differ from `a + (b + c)`, `a * (b + c)` can differ from `a * b + a * c`, and `(a + b) - b` can differ from `a`. `x * 1e300 / 1e300` can even be infinity rather than x. So by default the compiler may not regroup, distribute or cancel terms. With the `-fassociative-math` option, all of these become permitted (despite sounding like it only applies to associativity). A number of numerical techniques rely on the compiler not applying these rules. They compute something that is zero for real numbers, but which in floating point gives you a different, useful result.

#### Kahan summation

A classic example for this is Kahan summation, a technique for increasing the accuracy of additions. It keeps track of the rounding error of each addition and feeds it back in:

```cpp
double sum = 0.0;
double compensation = 0.0;
for (double el : values) {
    double compensated_el = el - compensation;
    double temp_sum = sum + compensated_el;
    compensation = (temp_sum - sum) - compensated_el;
    sum = temp_sum;
}
```

The trick is to do the addition you wanted, and then subtract the inputs from the result again. With real numbers, that would just give you zero, but with strict floating-point math, it gives you the rounding error of the addition, which is then subtracted from the next element before it is added. This technique, and related ones, can give you significant additional precision when summing up a large number of floating-point values.

As one example, here's summing 0.1 ten million times, compiled with `-O3` ([Compiler Explorer](https://godbolt.org/z/scKcKa5qW)):

|  | plain loop | Kahan |
| --- | --- | --- |
| strict | 999999.99983897537 | 1000000 |
| GCC `-ffast-math -march=native` | 1000000.0000402561 | 1000000.0000402561 |
| Clang `-ffast-math -march=native` | 999999.99998993601 | 999999.99998993601 |

(The exact fast-math values depend on the CPU that Compiler Explorer runs on.)

As you can see from the resulting numbers, only the Kahan sum without floating-point optimizations delivers the expected result exactly. Under fast-math, the compiler sees that the compensation term is mathematically zero, so it reduces the whole thing to the plain loop.

Another interesting thing worth pointing out though is that with fast-math, the result is actually closer to the exact result than the plain loop without fast-math. This is not uncommon to see. In this case, the vectorized loop keeps several partial sums side by side and only adds them up at the end, and smaller partial sums lose less to rounding. More generally, with flexibility in math optimizations and access to modern hardware, the compiler can choose transformations and instructions that are not only more performant, but also more accurate.

It's also worth noting that Kahan summation does come with a performance cost. In my testing, it was about 4× slower than the plain loop under strict floating-point math. This is slower than for example 128-bit integer addition, in case you were considering a fixed-point alternative. It is also not a silver bullet for accuracy: Summing e.g. `{ 1.0e18, 1.0, -1.0e18 }` does not return the correct answer, even when using Kahan summation.

#### Loops that run until nothing changes

A stopping rule for numerical calculations that is sometimes used is "stop when adding the next term no longer changes the sum". Here it is in a Taylor series for the exponential function:

```cpp
// exp(x) = 1 + x + x^2/2! + x^3/3! + ...
double exp_series(double x) {
    double sum = 1.0;
    double term = x;
    for (int k = 2; sum + term != sum; ++k) {
        sum += term;
        term *= x / k;
    }
    return sum;
}
```

The loop relies on rounding. Once `term` is small enough that adding it to `sum` no longer changes it, the loop ends.

For real numbers, `sum + term != sum` simply means `term != 0`, and under fast-math the compiler may rewrite it that way. Whether it does depends on the compiler and on how the test is spelled. GCC, Clang and MSVC all leave the above spelling alone. All three rewrite the equivalent `(sum + term) - sum != 0`. The loop then runs until `term` underflows to zero.

I'd argue that this kind of loop is questionable in general, and not only under fast-math, because, depending on what you're trying to calculate, it can produce surprises, even when using strict math. Here is the same stopping rule applied when using a Newton iteration to calculate cube roots:

```cpp
double cube_root(double a) {
    double x = first_guess(a);   // within a factor of 2 of the root
    for (;;) {
        double step = (a / (x * x) - x) / 3.0;
        if (x + step == x) {
            return x;
        }
        x += step;
    }
}
```

`cube_root(4.0)` never returns. That is not a fast-math problem: it happens under strict IEEE arithmetic, with all three compilers. Close to the root, the rounding error in `a / (x * x)` is large enough to push `x` to the neighboring double, and the next step pushes it back.

A better way is an explicit tolerance:

```cpp
if (std::abs(step) <= epsilon * std::abs(x)) {
    return x;
}
```

This compares two ordinary numbers instead of relying on how an addition rounds, so there is nothing for fast-math to simplify, with negligible difference in performance. It also gives you control over the loop exit: you decide how accurate the result needs to be, instead of always iterating down to the last bit. Setting `epsilon` to the machine epsilon (`std::numeric_limits<double>::epsilon()`), the loop stops within one to two ulp of the exact result.

For best performance, work out how many iterations a satisfactory result needs, and run exactly that many. In this example, 7 Newton steps are enough for every positive double. A loop with a fixed count has no data-dependent exit, so the compiler can easily unroll it and vectorize it across many inputs.

[This example](https://godbolt.org/z/x5xnMTTqe) has all three versions of the cube root. The tolerance version and the fixed version give the same results with and without `-ffast-math`.

#### Rounding with a magic number

Say you want to round a double to the nearest integer, as if by calling `nearbyint`. There is a neat trick that achieves the same as the function and is sometimes faster, depending on the platform. It exploits the fact that all double values between 2⁵² and 2⁵³ are integers. Add 1.5·2⁵² to any double with |x| < 2⁵¹ (the trick may not work for larger magnitudes), and the result lands in that range, rounded to an integer. Subtract 1.5·2⁵² again, and you are left with x rounded to the nearest integer:

```cpp
static constexpr double magic = 6755399441055744.0;   // 1.5 * 2^52
double r = (x + magic) - magic;
```

When using fast-math, the compiler sees right through our little trick, notices that we add and subtract the same number, and completely cancels out those operations. In [this example](https://godbolt.org/z/4sbY4ndY6), the first pane (`-O3 -ffast-math -march=native`) compiles the trick to a bare `ret`.

Why would anyone write this? Because the original x86-64 instruction set (SSE2) has no instruction that rounds a double to a whole number. `std::nearbyint` is a library call there, which is about 10× slower than two additions in my test. The second pane of the example shows that baseline: the trick as an addition and a subtraction, and `std::nearbyint` as a call.

On current hardware the trick is obsolete. SSE4.1 added a rounding instruction to x86, and 64-bit ARM has always had one. When the compiler is allowed to use it, `std::nearbyint` becomes that single instruction, which is as fast as the trick and also works under fast-math.

#### Cody-Waite range reduction

Cody-Waite range reduction is a trick used to calculate accurate results for functions like `exp`, `sin`, `cos` and friends.

Functions like `exp` are computed with a polynomial, and such a polynomial is only accurate on a small interval. So the first step is to reduce the argument to that interval. For `exp` that uses exp(x) = exp(ln(2) · n + r) = 2ⁿ · exp(r), with n = round(x / ln 2) and r = x - n·ln 2. Calculating and multiplying with 2ⁿ is exact in floating point. Additionally, r is guaranteed to lie between -0.35 and 0.35, where a polynomial of modest degree is accurate.

That moves all the difficulty into computing r. We need r to be precise, because any absolute error in r becomes a relative error of the same size in the result (as the derivative of exp(r) is exp(r)). The trouble is the term n·ln 2, which goes wrong in two ways. First, ln 2 can't be stored exactly in a double, and multiplying by n multiplies that small error too. For x = 100, n is 144, and this alone puts r off by about 3.3·10⁻¹⁵, or 120 ulp of r. Second, the product n·ln 2 is itself rounded, which adds an error of similar size.

The solution is to split ln 2 into two constants:

```cpp
static constexpr double hi = 0x1.62e42fee00000p-1;   // the first 32 bits of ln 2, then zeros
static constexpr double lo = 0x1.a39ef35793c76p-33;  // the next 53 bits
double r = (x - n * hi) - n * lo;
```

Each step is chosen in a clever way that avoids rounding where possible:

- `hi` has only 32 significant bits, and n needs at most 11, so `n * hi` fits into a double exactly.
- `n * hi` is close to x, and the difference of two doubles that close is always exact.
- Only `n * lo` rounds. But `lo` is about 10⁹ times smaller than ln 2, so its rounding error is also about 10⁹ times smaller.

In effect, r is computed with 85 bits of ln 2 instead of 53, using one extra multiply and subtract.

For real numbers, `(x - n*hi) - n*lo` is the same as `x - n*(hi + lo)`, and `hi + lo` rounds to the double closest to ln 2. Clang's fast-math makes exactly that simplification, which turns the trick back into the naive version. (GCC does not seem to optimize this away.)

[This example](https://godbolt.org/z/ajahK7YEe) computes `exp` both ways and compares the results with the correctly rounded value. Without fast-math, the Cody-Waite version stays within 1 ulp, while the naive one is up to 415 ulp off at x = 700, because the error grows with n. With `-ffast-math` and Clang, the two versions give identical results for every input.

## What you gain

Following IEEE 754 strictly enables clever tricks, but it also forbids some optimizations. A few of them are small. The compiler may simplify `sqrt(x) * sqrt(y)` to `sqrt(x * y)` or `sin(x) / cos(x)` to `tan(x)`, saving a function call each ([Compiler Explorer](https://godbolt.org/z/rqYc8h9zE)), and it may rearrange a polynomial into a form with a shorter chain of dependent operations. Each of these also changes the result for some inputs: `sqrt(x) * sqrt(y)` with x = y = 1e200 is 1e200 before and infinity after.

The biggest gains come from vectorization: processing multiple values with a single instruction. Simple element-wise loops get vectorized even under strict math. But loops that combine values, like a sum, a dot product or a maximum, require reassociation, and calls to math functions require that nobody checks `errno` or floating-point exceptions. Those only get vectorized with fast-math. Performance gains from this are large, and they grow with the width of the vector registers. Even with the default x86-64 architecture selected by the compiler (SSE2), you get significant speedups, but that instruction set is from the year 2000. If you set a newer target architecture with wider registers, this speedup becomes even more significant. This is an ongoing process too: Over time, registers will only get wider, and the hardware that has them becomes more widespread, meaning either your speedup becomes greater, or your loss of portability by using these instructions becomse smaller.

### Vectorized loops

I wanted to get a feel for how big the speedup can be, so I ran some quick tests on my machine (an x86-64 i7-11370H, on which GCC uses 256-bit vectors) using GCC 13.3, compiled with `-O3 -march=native`, with and without `-ffast-math`. The results are nanoseconds per element of an array of 4096 doubles, for a few different types of operations.

| operation | loop | strict (ns/element) | fast-math (ns/element) | speedup |
| --- | --- | --- | --- | --- |
| Accumulation | `s += x[i]` | 0.88 | 0.21 | 4.2× |
| Dot product | `s += x[i] * y[i]` | 0.89 | 0.22 | 4.1× |
| Find max | `m = x[i] > m ? x[i] : m` | 0.90 | 0.21 | 4.2× |

### Vectorized math functions

Vectorization can speed up not just basic arithmetic, but also math functions from the standard library. For `sqrt`, the standard library call is replaced by a vector instruction. For other functions, GCC may replace them with vectorized versions of the normal functions ([Compiler Explorer](https://godbolt.org/z/3e8dWG7hW)).

| operation | strict (ns/element) | fast-math (ns/element) | speedup |
| --- | --- | --- | --- |
| `sqrt` | 1.34 | 0.67 | 2.0× |
| `exp` | 4.25 | 2.00 | 2.1× |
| `sin` | 7.06 | 0.78 | 9.0× |

Do note that the vectorized versions come with a slightly increased tolerance as well: Where the base versions are off by at most one ulp, the vectorized functions may be off by up to 4 ulp.

## When to use fast-math, and when not

Advice surrounding fast-math is often presented as a dichotomy: On the one side you have the default strict mode, which is intuitive, predictable and safe, on the other side are the fast-math optimizations, which can speed up your code, but also make it much more unpredictable and unsafe, so it needs to be tested thoroughly.

My biggest objection to this is to the first half: Actually, floating-point math is really difficult to get right, period, even when using strict semantics. In this article, I showed a few examples where it gives unexpected results, but just in general, I'd argue that most floating-point code is not generically correct, but requires knowledge of the domain the code is supposed to operate in.

Take something as simple as comparing two numbers for equality: Due to the imprecision inherent to floating-point math (even with strict math), you can almost never just compare two floating-point numbers with `==`. Instead, you check if numbers are within a tolerance of each other. This requires a decision about what tolerance is appropriate, which depends entirely on what the numbers represent.

This extends even to safety-critical applications, where there are still inaccuracies that simply don't matter because they are far below anything relevant in the real world.

Many strict floating-point semantics focus on behavior at the edges: NaNs, infinities, signed zeros, subnormals and exceptions. But numerical code usually has to stay clear of these cases anyway, regardless of the compiler flags. Strict arithmetic doesn't make an algorithm numerically stable, and it doesn't guarantee that the result is useful.

Strict math also comes at a performance cost that isn't always obvious. Many developers would be surprised by some of the optimizations it prevents, such as a simple loop over `sqrt` not getting vectorized because the compiler has to preserve `errno`, or a division by a constant not getting replaced by a multiplication with its inverse, even when the difference is irrelevant to the application. And that is not to mention the biggest gain that fast-math offers: vectorized sums, reductions and math functions.

There are, of course, techniques that genuinely require precise floating-point semantics. But I'd claim that most developers writing math code don't rely on these techniques. The people most likely to recommend against fast-math are often IEEE 754 experts who know these tricks inside out, and rely on them to write code that works generically for every possible input. Their advice is perfectly reasonable for the code they write, but that doesn't necessarily make it applicable to everyone else. For your own code? If you can't name the technique your code relies on, it probably doesn't rely on one.

Of course, I am not arguing to simply flip the switch in a codebase you don't understand and cannot adequately test or benchmark. Fast-math changes assumptions in ways that can be difficult to track down, and none of the failures above necessarily comes with an error message. But floating-point code should be tested thoroughly in any case, including when compiled with strict settings.

When your algorithms don't rely on special floating-point techniques, fast-math can deliver significant speedups at relatively little cost. In one robotics-adjacent project I worked on, enabling fast-math improved end-to-end performance by around 10%, just from changing a compiler flag. You don't have to hand-optimize every loop or rewrite your calculations to make better use of the hardware. You can simply let the compiler choose a more efficient way to perform them.

Regardless of what you choose to do in your code base, please stay fun and safe, and I hope that this article was interesting and will help you make informed decisions in the future.
