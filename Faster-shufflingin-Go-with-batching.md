
![image-24](https://lemire.me/blog/wp-content/uploads/2024/08/image-24-825x510.jpg)

# Faster shuffling in Go with batching

Random integer generation is a fundamental operation in programming, often used in tasks like shuffling arrays. Go’s standard library provides convenient tools like `rand.Shuffle` for such purposes. You may be able to beat the standard library by a generous margin. Let us see why.

Go’s `rand.Shuffle` implements the [Fisher-Yates (or Knuth) shuffle algorithm](https://en.wikipedia.org/wiki/Fisher–Yates_shuffle), a standard method for generating a uniform random permutation. In Go, you may call it like so:

```go
import "math/rand/v2"

func shuffleStandard(data []uint64) {
  rand.Shuffle(len(data), func(i, j int) {
    data[i], data[j] = data[j], data[i]
  })
}
```

The `rand.Shuffle` function takes a length and a swap function, internally generating random indices in the range `[0, i)` for each position `i` and swapping elements accordingly.

Under the hood, `rand.Shuffle` needs random integers within a shrinking range. This can be a surprisingly expensive step that requires a modulo/division operation per function call. We can optimize such a process so that division are avoided with the following routine ([Lemire, 2019](https://arxiv.org/abs/1805.10941)):

```go
package main

import (
    "math/bits"
)

func randomBounded(rangeVal uint64) uint64 {
    random64bit := rand.Uint64()
    hi, lo := bits.Mul64(random64bit, rangeVal)
    leftover := lo
    if leftover < rangeVal {
        threshold := -rangeVal % rangeVal
        for leftover < threshold {
            random64bit = rand.Uint64()
            hi, lo = bits.Mul64(random64bit, rangeVal)
            leftover = lo
        }
    }
    return hi // [0, range)
}
```

This function returns an integer in the interval `[0, rangeVal)`. It is fair: as long as the 64-bit generator is uniform, it will generate all values in the interval with the same probability.

The original shuffle function in the Go standard library used a slower division-based approach while the `math/rand/v2` approach uses this fast algorithm. So for better performance, always use `math/rand/v2` and not `math/rand`.

While efficient, this improved function needs to generate many random numbers. For `rand.Shuffle`, this means one `rand.Uint64()` call per element.

We can optimize drastically the function with batching ([Brackett-Rozinsky and Daniel Lemire, 2024](https://onlinelibrary.wiley.com/doi/10.1002/spe.3369)). That is, we can use a single 64-bit random integer to produce multiple random numbers, reducing the number of random-number generations.

I have implemented shuffling by batching in Go and my results are promising.

The `partialShuffle64b` function is the heart of my approach:

```go
func partialShuffle64b(storage []uint64, indexes [7]uint64, n, k,
                           bound uint64) uint64 {
    r := rand.Uint64()

    for i := uint64(0); i < k; i++ {
        hi, lo := bits.Mul64(n-i, r)
        r = lo
        indexes[i] = hi
    }

    if r < bound {
        bound = n
        for i := uint64(1); i < k; i++ {
            bound *= (n - i)
        }
        t := (-bound) % bound

        for r < t {
            r = rand.Uint64()
            for i := uint64(0); i < k; i++ {
                hi, lo := bits.Mul64(n-i, r)
                r = lo
                indexes[i] = hi
            }
        }
    }
    for i := uint64(0); i < k; i++ {
        pos1 := n - i - 1
        pos2 := indexes[i]
        storage[pos1], storage[pos2] = storage[pos2],
                     storage[pos1]
    }
    return bound
}
```

The `partialShuffle64b` function performs a partial shuffle of a slice by generating `k` random indices from a range of size `n`, using a single 64-bit random number `r` obtained from `rand.Uint64()`, and then swapping elements at those indices. I reuse an array where the indices are stored. Though it looks much different, it is essentially a generalization of the `randomBounded` routine already used by the Go standard library.

Using the `partialShuffle64b` function, we can code a shuffle function where we generate indices in batches of two:

```go
func shuffleBatch2(storage []uint64) {
    i := uint64(len(storage))
    indexes := [7]uint64{}
    for ; i > 1<<30; i-- {
        partialShuffle64b(storage, indexes, i, 1, i)
    }
    bound := uint64(1) << 60
    for ; i > 1; i -= 2 {
        bound = partialShuffle64b(storage, indexes, i, 2, bound)
    }
}
```

We can also generalize and generate the indices in batches of six when possible:

```go
func shuffleBatch23456(storage []uint64) {
    i := uint64(len(storage))
    indexes := [7]uint64{}

    for ; i > 1<<30; i-- {
        partialShuffle64b(storage, indexes, i, 1, i)
    }
    bound := uint64(1) << 60
    for ; i > 1<<19; i -= 2 {
        bound = partialShuffle64b(storage, indexes, i, 2, bound)
    }
    bound = uint64(1) << 57
    for ; i > 1<<14; i -= 3 {
        bound = partialShuffle64b(storage, indexes, i, 3, bound)
    }
    bound = uint64(1) << 56
    for ; i > 1<<11; i -= 4 {
        bound = partialShuffle64b(storage, indexes, i, 4, bound)
    }
    bound = uint64(1) << 55
    for ; i > 1<<9; i -= 5 {
        bound = partialShuffle64b(storage, indexes, i, 5, bound)
    }
    bound = uint64(1) << 54
    for ; i > 6; i -= 6 {
        bound = partialShuffle64b(storage, indexes, i, 6, bound)
    }
    if i > 1 {
        partialShuffle64b(storage, indexes, i, i-1, 720)
    }
}
```

Using the fact that Go comes with full support for benchmarking, we can try shuffling 10,000 elements as a Go benchmark:

```go
func BenchmarkShuffleStandard10K(b *testing.B) {
    size := 10000
    data := make([]uint64, size)
    for i := 0; i < size; i++ {
        data[i] = uint64(i)
    }
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        shuffleStandard(data)
    }
}

func BenchmarkShuffleBatch2_10K(b *testing.B) { ... }
func BenchmarkShuffleBatch23456_10K(b *testing.B) { ... }
```

On an Intel Gold 6338 CPU @ 2.00GHz with Go 1.24, I get the following results :

*   **Standard (rand.Shuffle)**: 13 ns/element.
*   **Batch 2**: 9.3 ns/element (~1.4x faster).
*   **Batch 2-3-4-5-6**: 5.0 ns/element (~2.6x faster).

You will get different results on your systems. [I make my code available on GitHub](https://github.com/lemire/Code-used-on-Daniel-Lemire-s-blog/tree/master/2025/04/06).

The batched approaches reduce the number of random numbers generated, amortizing the cost of `rand.Uint64()` across multiple outputs. The 6-way batch maximizes this effect, yielding the best performance.

Go’s `rand.Shuffle` is a solid baseline, but batching random integer generation can make it much faster. By generating multiple random numbers from a single 64-bit value, we can boost efficiency—by over 2x in our benchmarks.

**References**

*   Nevin Brackett-Rozinsky, Daniel Lemire, [Batched Ranged Random Integer Generation](https://onlinelibrary.wiley.com/doi/10.1002/spe.3369), Software: Practice and Experience 55 (1), 2024.
*   Daniel Lemire, [Fast Random Integer Generation in an Interval](https://arxiv.org/abs/1805.10941), ACM Transactions on Modeling and Computer Simulation, 29 (1), 2019.

**Appendix: Code**

```go
package main

// Nevin Brackett-Rozinsky, Daniel Lemire, Batched Ranged Random
// Integer Generation, Software: Practice and Experience 55 (1), 2024.
// Daniel Lemire, Fast Random Integer Generation in an Interval,
// ACM Transactions on Modeling and Computer Simulation, 29 (1), 2019.
import (
    "fmt"
    "math/bits"
    "math/rand/v2"
)

func shuffleBatch2(storage []uint64) {
    i := uint64(len(storage))
    indexes := [7]uint64{}
    for ; i > 1<<30; i-- {
        partialShuffle64b(storage, indexes, i, 1, i)
    }
    bound := uint64(1) << 60
    for ; i > 1; i -= 2 {
        bound = partialShuffle64b(storage, indexes, i, 2, bound)
    }
}

func shuffleBatch23456(storage []uint64) {
    i := uint64(len(storage))
    indexes := [7]uint64{}

    for ; i > 1<<30; i-- {
        partialShuffle64b(storage, indexes, i, 1, i)
    }
    bound := uint64(1) << 60
    for ; i > 1<<19; i -= 2 {
        bound = partialShuffle64b(storage, indexes, i, 2, bound)
    }
    bound = uint64(1) << 57
    for ; i > 1<<14; i -= 3 {
        bound = partialShuffle64b(storage, indexes, i, 3, bound)
    }
    bound = uint64(1) << 56
    for ; i > 1<<11; i -= 4 {
        bound = partialShuffle64b(storage, indexes, i, 4, bound)
    }
    bound = uint64(1) << 55
    for ; i > 1<<9; i -= 5 {
        bound = partialShuffle64b(storage, indexes, i, 5, bound)
    }
    bound = uint64(1) << 54
    for ; i > 6; i -= 6 {
        bound = partialShuffle64b(storage, indexes, i, 6, bound)
    }
    if i > 1 {
        partialShuffle64b(storage, indexes, i, i-1, 720)
    }
}

func partialShuffle64b(storage []uint64, indexes [7]uint64, n, k,
  bound uint64) uint64 {
    r := rand.Uint64()

    for i := uint64(0); i < k; i++ {
        hi, lo := bits.Mul64(n-i, r)
        r = lo
        indexes[i] = hi
    }

    if r < bound {
        bound = n
        for i := uint64(1); i < k; i++ {
            bound *= (n - i)
        }
        t := (-bound) % bound

        for r < t {
            r = rand.Uint64()
            for i := uint64(0); i < k; i++ {
                hi, lo := bits.Mul64(n-i, r)
                r = lo
                indexes[i] = hi
            }
        }
    }
    for i := uint64(0); i < k; i++ {
        pos1 := n - i - 1
        pos2 := indexes[i]
        storage[pos1], storage[pos2] = storage[pos2],
                    storage[pos1]
    }
    return bound
}
```

Daniel Lemire, "Faster shuffling in Go with batching," in *Daniel Lemire's blog*, April 6, 2025, [https://lemire.me/blog/2025/04/06/faster-shuffling-in-go-with-batching/](https://lemire.me/blog/2025/04/06/faster-shuffling-in-go-with-batching/).

