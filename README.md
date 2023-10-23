This document is a compilation of insightful articles about coding practices and
design patterns. In order to persist the key takeaways after the reading, I'll
write some conclusions for each article and add some linked resources.

# Table of contents
- [Table of contents](#table-of-contents)
- [Articles](#articles)
  - [Understanding Go's Stack and Heap](#understanding-gos-stack-and-heap)
    - [Leaking memory](#leaking-memory)
  - [Retry strategies](#retry-strategies)
    - [Discussion](#discussion)
- [Next in the list](#next-in-the-list)

# Articles

## [Understanding Go's Stack and Heap](https://golang.howtos.io/understanding-go-s-stack-and-heap/)

> The **stack** is a region of memory that is organized in
> a last-in-first-out (LIFO) manner. It is used to store local variables,
> function call information, and other data related to function
> execution. The stack has a fixed size, and memory
> allocation/deallocation is fast, as it involves simple pointer
> manipulation.
>
> The **heap**, on the other hand, is a region of memory
> used for dynamic memory allocation. It is responsible for storing
> objects and structures that are created at runtime. The heap has a
> larger size compared to the stack and requires manual memory management
> (allocation and deallocation). Memory allocation in the heap is slower,
> as it involves finding a suitable block of memory and updating memory
> allocation tables.

> When a goroutine is created, it is allocated a small initial stack size (…).
> Each goroutine’s stack starts with a small fixed size, but it can grow
> dynamically as needed. The stack grows in a contiguous manner (…)

> Whenever you create objects using the `new` keyword or allocate memory using
> functions like `make()`, the memory is allocated on the heap.

> Go employs a garbage collector called the **concurrent garbage collector**
> (concurrent GC) for automatic memory management. The concurrent GC
> works in the background, scanning and collecting unused memory blocks.

### Leaking memory

- Shrinking a slice doesn't modify the underlying array: the values aren't
  accessible anymore but the pointers in the array still reference a value
    - Solution: `nil`-out elements before shrinking
- `map`'s underlying implementation uses buckets (arrays) of key-value pairs. A
  `map` can only grow its buckets, so even if we `delete` all the elements, the
  values are zeroed but the capacity is untouched
  ([source](https://teivah.medium.com/maps-and-memory-leaks-in-go-a85ebe6e7e69)).

## [Retry strategies](https://encore.dev/blog/retries)

Usually clients will want to retry a failed request. The delay between requests
is key to keep a good balance between providing a snappy user experience and
protecting the server from load spikes.

This article analyzes different strategies through interactive visualizations:

- An **immediate retry policy** (a fixed, small delay) will cause bursts of
  requests that will overload the server. Even when the requests stop failing,
  the server will struggle to recover.
  - Even adding server instances and a load balancer, the problem will get worse
    as the number of clients grow.
  - In order to recover, it will be needed to scale the server horizontally
    after a spike in failures.
- **Waiting long times** after failed requests will protect the server but will
  cause a bad user experience. Failures will happen, usually with a low
  probability, and in these cases retrying immediately is safe. We only want
  long delays when the server is consistently failing.
- An **exponential backoff** strategy will increase the delay in each new
  retried request. This protects the server from a potential overload as the
  failure observations increase.
  - In order to prevent requests from syncing (which might happen during the
    first failures with small delays), **jitter** might be added. This will
    randomize the delay, avoiding situations in which clients retry requests at
    the same time, even if the delay between retrials is high.

### Discussion

- How would the code snippet change if we wanted to retry only on certain HTTP
  codes?
- If a request is a blocking process in a client, we'll seldom want an infinite
  retry strategy. We might want to add a maximum retry count or a maximum delay
  in order to free that process.

# Next in the list

- [ ] [Hashing](https://samwho.dev/hashing/)
- [ ] [Memory allocation](https://samwho.dev/memory-allocation/)
- [ ] [Load balancing](https://samwho.dev/load-balancing/)
