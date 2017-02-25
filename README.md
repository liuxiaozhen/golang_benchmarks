# Golang Benchmarks

Various benchmarks for different patterns in Go. Many of these are implementations of or were
inspired by
[this excellent article on performance in Go](http://bravenewgeek.com/so-you-wanna-go-fast/).

### Allocate on Stack vs Heap

`allocate_stack_vs_heap_test.go`

Benchmark Name|Iterations|Per-Iteration|Bytes Allocated per Operation|Allocations per Operation
----|----|----|----|----
BenchmarkAllocateFooStack | 1000000000 | 2.27 ns/op  |  0 B/op   | 0 allocs/op
BenchmarkAllocateBarStack | 1000000000 | 2.27 ns/op  |  0 B/op   | 0 allocs/op
BenchmarkAllocateFooHeap  | 50000000   | 29.0 ns/op  | 32 B/op   | 1 allocs/op
BenchmarkAllocateBarHeap  | 50000000   | 30.2 ns/op  | 32 B/op   | 1 allocs/op

Generated using go version go1.7.5 darwin/amd64

This benchmark just looks at the difference in performance between allocating a struct on the stack
versus on the heap. As expected, allocating a struct on the stack is much faster than allocating it
on the heap. The two structs I used in the benchmark are below:

```
type Foo struct {
	foo int64
	bar int64
	baz int64
}

type Bar struct {
	foo int64
	bar int64
	baz int64
	bah int64
}
```

One interesting thing is that although `Foo` is only 24 bytes, when we allocate it on the heap, 32 bytes are
allocated and when `Bar` is allocated on the heap, 32 bytes are allocated for it as well. When I first
saw this, my initial suspicion was that Go's memory allocator allocates memory in certain bin sizes instead
of the exact size of the struct, and there is no bin size between 24 and 32 bytes, so `Foo` was allocated
the next highest bin size, which was 32 bytes. This
[blog post](https://medium.com/@robertgrosse/optimizing-rc-memory-usage-in-rust-6652de9e119e#.x2kfg63oh)
examines a similar phenomenen in Rust and its memory allocator jemalloc. As for Go, I found the following
in the file
[runtime/malloc.go](https://github.com/golang/go/blob/01c6a19e041f6b316c17a065f7a42b8dab57c9da/src/runtime/malloc.go#L27):

```
// Allocating a small object proceeds up a hierarchy of caches:
//
//	1. Round the size up to one of the small size classes
//	   and look in the corresponding mspan in this P's mcache.
//  ...
```

### Buffered vs Synchronous Channel

`buffered_vs_unbuffered_channel_test.go`

Benchmark Name|Iterations|Per-Iteration
----|----|----
BenchmarkSynchronousChannel | 5000000  | 240 ns/op
BenchmarkBufferedChannel    | 10000000 | 108 ns/op

Generated using go version go1.7.5 darwin/amd64

This benchmark examines the speed with which one can put objects onto a channel and comes from this
[golang-nuts forum post](https://groups.google.com/forum/#!topic/golang-nuts/ec9G0MGjn48). Using a buffered
channels is over twice as fast as using a synchronous channel which makes sense since the goroutine that is
putting objects into the channel need not wait until the object is taken out of the channel before placing
another object into it.

### Channel vs Ring Buffer

`channel_vs_ring_buffer_test.go`

Benchmark Name|Iterations|Per-Iteration|Bytes Allocated per Operation|Allocations per Operation
----|----|----|----|----
BenchmarkChannelSPSC    | 10000000  |        140 ns/op  |     8 B/op |    1 allocs/op
BenchmarkRingBufferSPSC | 20000000  |        106 ns/op  |     8 B/op |    1 allocs/op
BenchmarkChannelSPMC    |  5000000  |        369 ns/op  |     8 B/op |    1 allocs/op
BenchmarkRingBufferSPMC |  5000000  |        341 ns/op  |     8 B/op |    1 allocs/op
BenchmarkChannelMPSC    |  3000000  |        417 ns/op  |     8 B/op |    1 allocs/op
BenchmarkRingBufferMPSC |   500000  |       8387 ns/op  |     8 B/op |    1 allocs/op
BenchmarkChannelMPMC    |       20  |   70112786 ns/op  | 10532 B/op | 1031 allocs/op
BenchmarkRingBufferMPMC |        1  | 1228960979 ns/op  | 14256 B/op | 1015 allocs/op

The blog post [So You Wanna Go Fast?](http://bravenewgeek.com/so-you-wanna-go-fast/) also took a look at using
channels versus using a
[lock-free ring buffer](https://github.com/Workiva/go-datastructures/blob/master/queue/ring.go). I decided to
run similar benchmarks myself and the results are above. The benchmarks SPSC, SPMC, MPSC, and MPMC refer to
Single Producer Single Consumer, Single Producer Mutli Consumer, Mutli Producer Single Consumer, and
Mutli Producer Mutli Consumer respectively. The blog post found that for the SPSC case, a channel was faster
than a ring buffer when the tests were run on a single thread (`GOMAXPROCS=1`) but the ring buffer was faster
when the tests were on multiple threads (`GOMAXPROCS=8`). The blog post also examined the the SPMC and MPMC
cases and found similar results. That is, channels were faster when run on a single thread and the ring buffer
was faster when the tests were run with multiple threads. I ran all the test with `GOMAXPROCS=4` which is
the number of CPU cores on the machine I ran the tests on (a 2015 MacBook Pro with a 3.1 GHz Intel Core i7
Processor, it has 2 physical CPUs,`sysctl hw.physicalcpu`, and 4 logical CPUs, `sysctl hw.logicalcpu`).
Ultimately, the benchmarks I ran produced different results. They show that in the SPSC and SPMC cases the
performance of a channel and ring buffer are similar with the ring buffer holding a small advantage. However,
in the MPSC and MPMC a channel performed much better than a ring buffer did.

### defer

`defer_test.go`

Benchmark Name|Iterations|Per-Iteration
----|----|----
BenchmarkMutexUnlock     | 50000000 | 25.8 ns/op
BenchmarkMutexDeferUnlock| 20000000 | 92.1 ns/op

Generated using go version go1.7.5 darwin/amd64

`defer` carries a slight performance cost, so for simple use cases it may be preferable
to call any cleanup code manually. As this [blog post](http://bravenewgeek.com/so-you-wanna-go-fast/)
notes, `defer` can be called from within conditional blocks and must be called if a functions
panics as well. Therefore, the compiler can't simply add the deferred function
wherever the function returns and instead `defer` must be more nuanced, resuling in the performance
hit. There is, in fact, an [open issue](https://github.com/golang/go/issues/14939) to address the
performance cost of `defer`.
[Another discussion](http://grokbase.com/t/gg/golang-nuts/158zz5p42w/go-nuts-defer-performance)
suggests calling `defer mu.Unlock()` before one calls `mu.Lock()` so the defer call will
be moved out of the critical path:

```
defer mu.Unlock()
mu.Lock()
```

### Function Call

`function_call_test.go`

Benchmark Name|Iterations|Per-Iteration
----|----|----
BenchmarkPointerToStructMethodCall | 2000000000 | 0.32 ns/op
BenchmarkInterfaceMethodCall       | 2000000000 | 1.90 ns/op
BenchmarkFunctionPointerCall       | 2000000000 | 1.91 ns/op

Generated using go version go1.7.5 darwin/amd64

This benchmark looks at the overhead for three different kinds of function calls: calling a method
on a pointer to a struct, calling a method on an interface, and calling a function through a function
pointer field in a struct. As expected, the method call on the pointer to the struct is the fastest since
the compiler knows what function is being called at compile time, whereas the others do not. For example,
the interface method call relies on dynamic dispatch at runtime to determine which function call and
likewise the function pointer to call is determined at runtime as well and has almost identical performance
to the interface method call.

### Pass By Value vs Reference

`pass_by_value_vs_reference_test.go`

Benchmark Name|Iterations|Per-Iteration
----|----|----
BenchmarkPassByReferenceOneWord    |  1000000000  | 2.20 ns/op
BenchmarkPassByValueOneWord        |  1000000000  | 2.58 ns/op
BenchmarkPassByReferenceFourWords  |   500000000  | 2.71 ns/op
BenchmarkPassByValueFourWords      |  1000000000  | 2.78 ns/op
BenchmarkPassByReferenceEightWords |  1000000000  | 2.32 ns/op
BenchmarkPassByValueEightWords     |   300000000  | 4.35 ns/op

This benchmark looks at the performance cost of passing a variable by reference vs passing it by value. For
small structs there doesn't appear to be much of a difference, but as the structs gets larger we start to
see a bit of difference which is to be expected since the larger the struct is the more words that have to
be copied into the function's stack when passed by value.

### Pool

`pool_test.go`

Benchmark Name|Iterations|Per-Iteration|Bytes Allocated per Operation|Allocations per Operation
----|----|----|----|----
BenchmarkAllocateBufferNoPool | 20000000 |  118 ns/op | 368 B/op | 2 allocs/op
BenchmarkChannelBufferPool    | 10000000 |  213 ns/op |  43 B/op | 0 allocs/op
BenchmarkSyncBufferPool       | 50000000 | 27.7 ns/op |   0 B/op | 0 allocs/op

This benchmark compares three different memory allocation schemes. The first approach just
allocates its buffer on the heap normally. After it's done using the buffer it will eventually be
garbage collected. The second approach uses Go's sync.Pool type to pool buffers which caches objects
between runs of the garbage collector. The last approach uses a channel to permanently pool objects.
The difference between the last two approaches is the sync.Pool dynamically resizes itself and clears
items from the pool during a GC run. Two good resources to learn more about pools in Go
are these blog posts:
[Using Buffer Pools with Go](https://elithrar.github.io/article/using-buffer-pools-with-go/)
and
[How to Optimize Garbage Collection in Go](https://www.cockroachlabs.com/blog/how-to-optimize-garbage-collection-in-go/).

### Slice Initialization Append vs Index

`slice_intialization_append_vs_index_test.go`

Benchmark Name|Iterations|Per-Iteration|Bytes Allocated per Operation|Allocations per Operation
----|----|----|----|----
BenchmarkSliceInitializationAppend | 10000000 | 132 ns/op | 160 B/op | 1 allocs/op
BenchmarkSliceInitializationIndex  | 10000000 | 119 ns/op | 160 B/op | 1 allocs/op

This benchmark looks at slice initialization with `append` versus using an explicit index. I ran this benchmark
a few times and it seesawed back and forth. Ultimately, I think they compile down into the same code so there
probably isn't any actual performance difference. I'd like to take an actual look at the assembly
that they are compiled to and update this section in the future.

### String Concatenation

`string_concatenation_test.go`

Benchmark Name|Iterations|Per-Iteration|Bytes Allocated per Operation|Allocations per Operation
----|----|----|----|----
BenchmarkStringConcatenation      | 20000000 |  83.9 ns/op |  64 B/op | 1 allocs/op
BenchmarkStringBuffer             | 10000000 |   131 ns/op |  64 B/op | 1 allocs/op
BenchmarkStringJoin               | 10000000 |   144 ns/op | 128 B/op | 2 allocs/op
BenchmarkStringConcatenationShort | 50000000 |  25.4 ns/op |   0 B/op | 0 allocs/op

This benchmark looks at the three different ways to perform string concatenation, the first uses the builtin `+`
operator, the second uses a `bytes.Buffer` and the third uses `string.Join`. It seems using `+` is preferable
to either of the other approaches which are similar in performace.

The last benchmark highlights a neat optimization Go performs when concatenating strings with `+`. The
documentation for the string concatenation function in
[runtime/string.go](https://github.com/golang/go/blob/d7b34d5f29324d77fad572676f0ea139556235e0/src/runtime/string.go)
states:

```
// The constant is known to the compiler.
// There is no fundamental theory behind this number.
const tmpStringBufSize = 32

type tmpBuf [tmpStringBufSize]byte

// concatstrings implements a Go string concatenation x+y+z+...
// The operands are passed in the slice a.
// If buf != nil, the compiler has determined that the result does not
// escape the calling function, so the string data can be stored in buf
// if small enough.
func concatstrings(buf *tmpBuf, a []string) string {
  ...
}
```

That is, if the compiler determines that the resulting string does not escape the calling function it will
allocate a 32 byte buffer on the stack which can be used as the underlying buffer for the string if it 32
bytes or less. In the last benchmark, the resulting string is in fact less than 32 bytes so it can be stored
on the stack saving a heap allocation.
