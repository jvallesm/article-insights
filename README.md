This document is a compilation of insightful articles about coding practices and
design patterns. In order to persist the key takeaways after the reading, I'll
write some conclusions for each article and add some linked resources.

# Articles

## Understanding Go's Stack and Heap

Source: https://golang.howtos.io/understanding-go-s-stack-and-heap

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

## Go slice internals

Sources:

- https://go.dev/blog/slices-intro
- https://www.dolthub.com/blog/2023-10-20-golang-pitfalls-3

In Go, slices are built on top of arrays. Therefore, some clarifications are
needed:

> Go’s arrays are values. An array variable denotes the entire array; it is not
> a pointer to the first array element (as would be the case in C). This means
> that when you assign or pass around an array value you will make a copy of its
> contents.

The zero value of an array is a ready-to-use array whose elements are themselves
zeroed:

```go
var a [4]int
fmt.Println(a[2] == 0) // true
```

A slice is a descriptor of an array segment. From the [`slice`
definition](https://github.com/golang/go/blob/abf84221/src/runtime/slice.go#L15):

```go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```

Where `len` is the slice length and `cap` is its capacity or maximum length.

How slices work with their underlying arrays:

- Slicing an array doesn't copy its values.
- The `append` function appends the elements x to the end of the slice s, and
  grows the slice if a greater capacity is needed.
  - If the capacity doesn't need to be touched, the value is set in the
    underlying array and `len` is updated.
  - If the capacity is increased, the underlying array is copied into a larger
    one and the reference to the pointer (along with `len` and `cap`) is
    updated.

### Gotchas

> The full array will be kept in memory until it is no longer referenced.
> Occasionally this can cause the program to hold all the data in memory when
> only a small piece of it is needed.

```go
func CopyDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    b = digitRegexp.Find(b)

    // If we don't make this copy, unnecessary data in memory will be
    // referenced.
    c := make([]byte, len(b))
    copy(c, b)
    return c
}
```

Also, reusing a slice passed to `append` as an argument might be dangerous when
we aren't aware of the slice internals:

```go
sliceA := append([]int{1, 2}, 3)
sliceB := append(sliceA, 4)
sliceC := append(sliceA, 5)
sliceC[0] = 0

fmt.Println(sliceA) // prints [0 2 3]
fmt.Println(sliceB) // prints [0 2 3 5]
fmt.Println(sliceC) // prints [0 2 3 5]
```

## Retry strategies

Source: https://encore.dev/blog/retries

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

## Load Balancing

Source: https://samwho.dev/load-balancing

- **Round Robin** sends a request to each server **in turn**. Good when servers
  are equally powerful and requests are equally expensive.
- These conditions seldom happen in the real world. To address variance in
  request cost or server power, we can add a **request queue** with the trade-off of
  increasing the response latency.
- **Weighted Round Robin** assigns a weight to each server, modifying the
  probability of receiving a request. This weight can be assigned manually (if
  the power of each server is known) or dynamically (e.g. by comparing the
  latencies of the last N requests).
- In the **Least Connections** algorithm the load balancer leverages the
  knowledge of the open connections in each server and sends the request to the
  server with the least work. It's simple to implement and performs well with
  variance. Broadly, it is optimized for avoiding dropped requests at the cost
  of a slightly higher latency than Weighted Round Robin.
- This algorithm can be optimized for latency by adding the latency of the last
  N requests in each server.

## Rethinking Classical Concurrency Patterns

Source: https://www.youtube.com/watch?v=5zXAHh5tJqQ
([slides](https://drive.google.com/file/d/1nPdvhB0PutEJzdCq5ms6UI58dp50fcAN/view))

- Async optimisations are subtle and involve an extra complexity.
- Converting async <-> sync calls is [trivial](https://go.dev/play/p/FxSsaTToIHe).
- Add concurrency on the caller side, and make it an internal detail.
- Worker pool: don’t use waitgroups but channels (semaphore):

```go
sem := make(chan token, limit)
for _, task := range hugeSlice {
    sem <- token{}
    go func(task Task) {
        perform(task)
        <-sem
    }(task)
}

for n := limit; n > 0; n-- {
    sem <- token{}
}
```

## Google's Go style guide

Source: https://google.github.io/styleguide/go

- No `Get` in getters
  ([source](https://google.github.io/styleguide/go/decisions#getters))
- No [line length](https://google.github.io/styleguide/go/guide#line-length)
- No [util
  package](https://google.github.io/styleguide/go/best-practices#util-packages)
- Use [option
  structs](https://google.github.io/styleguide/go/best-practices#option-structure)
  for long lists of arguments.

## The Best Go framework: no framework?

Source: https://threedots.tech/post/best-go-framework

- Go is built around the Unix Philosophy, which favors building small
  independent pieces of software that do one thing well (i.e. `cat example.txt |
  sort | uniq`).
- It’s easy to quickly lose all your time saved on the project bootstrap just to
  fight with one framework limitation.
- Investing in maintainability most usually pays off. Loosely coupled
  architecture (e.g. being able to migrate one piece at a time) is a great way
  to achieve it.

## Cohesion in Go

Source: https://threedots.tech/post/increasing-cohesion-in-go-with-generic-decorators

- Highly cohesive code (a module or a function) is focused on a single purpose.
- Decorators (aka middlewares) can be used not only in implementation details
  but in the application logic. They isolate the purpose and ease testability.

```go
type subscribeAuthorizationDecorator struct {
    base SubscribeHandler
}

func (d subscribeAuthorizationDecorator) Handle(ctx context.Context, cmd Subscribe) error {
    user, err := UserFromContext(ctx)
    if err != nil {
        return err
    }

    if !user.Active {
        return errors.New("the user's account is not active")
    }

    return d.base.Handle(ctx, cmd)
}
```

- From Go 1.18, we can reduce the boilerplate by using generics:

```go
type CommandHandler[C any] interface {
    Handle(ctx context.Context, cmd C) error
}

// ...

func NewUnauthorizedSubscribeHandler(logger Logger, metricsClient MetricsClient) CommandHandler[Subscribe] {
    return loggingDecorator[Subscribe]{
        base: metricsDecorator[Subscribe]{
            base:   SubscribeHandler{},
            client: metricsClient,
        },
        logger: logger,
    }
}
```

## Compiling Containers – Dockerfiles, LLVM and BuildKit

Source: https://www.docker.com/blog/compiling-containers-dockerfiles-llvm-and-buildkit

- A docker image is made up of layers. Those layers form an immutable
  filesystem.
- Analogy: if an image is like an executable, a container is like a process. You
  can run multiple containers from one image, and a running image isn't an image
  at all but a container.
- BuildKit would be then a compiler that goes from a Dockerfile and a build
  context to a container image (instead of from source code to executable).
- One significant way image building differs from compiling is this build
  context. `docker build` takes a reference to the host filesystem to perform
  actions such as `COPY`.

## The Design of Everyday Things (Don Norman)

### The Psychopathology of Everyday Things

> Two of the most important characteristics of good design are *discoverability*
> and *understanding*. Discoverability: Is it possible to even figure out what
> actions are possible and where and how to per- form them? Understanding: What
> does it all mean? How is the product supposed to be used? What do all the
> different controls do?

In that sense, gRPC method signatures are (potentially) clearer than HTTP paths.

> Services, lectures, rules and procedures, and the organizational structures of
> businesses and governments do not have physical mechanisms, but their rules
> of operation have to be designed, sometimes informally, sometimes precisely
> recorded and specified.

> It is very hard to remove features of a newly designed product that had
> existed in an earlier version. It’s kind of like physical evolution. If a
> feature is in the genome, and if that feature is not associated with any
> negativity (i.e., no customers gripe about it), then the feature hangs on for
> generations

> It is interesting that things like the ‘R’ button on a desk telephone are
> largely determined through examples. Somebody asks, 'What is the 'R' button
> used for?@ and the answer is to give an example: "You can push 'R' to access
> loudspeaker paging."
> If nobody can think of an example, the feature is dropped. Designers are
> pretty bright people, however. They can come up with a plausible-sounding
> example for almost anything. Hence, you get features, many many features, and
> these features hang on for a long time. The end result is complex interfaces
> for essentially simple things.

> Discoverability results from appropriate application of five fundamental
> psychological concepts covered in the next few chapters: *affordances,
> signifiers, constraints, mappings*, and *feedback*. But there is a sixth
> principle, perhaps most important of all: the *conceptual model* of the
> system. It is the conceptual model that provides true understanding.

# Less is exponentially more

Source: https://commandcenter.blogspot.com/2012/06/less-is-exponentially-more.html

About why Go programmers tend to come from languages like Ruby or Python rather
than C++. An opinionated and oversimplified (in a self-aware way) quote:

>  Python and Ruby programmers come to Go because they don't have to surrender
>  much expressiveness, but gain performance and get to play with concurrency.
>
> C++ programmers don't come to Go because they have fought hard to gain
> exquisite control of their programming domain, and don't want to surrender any
> of it. To them, software isn't just about getting the job done, it's about
> doing it a certain way.
>
> The issue, then, is that Go's success would contradict their world view.

About the lack of generics (which are present from Go 1.18 but still relevant as
they haven't been necessary for Go's success as a language):

> (...) what it [someone's doubt about being able to program without generics]
> says is that types are the way to lift that burden [writing containers like
> lists of ints and maps of strings]. Types. Not polymorphic functions or
> language primitives or helpers of other kinds, but types.

> I believe that's a preposterous way to think about programming. What matters
> isn't the ancestor relations between things but what they can do for you.

> If C++ and Java are about type hierarchies and the taxonomy of types, Go is
> about composition. (...). It is a language of composition and coupling.
>
> The obvious example is the way interfaces give us the composition of
> components. It doesn't matter what that thing is, if it implements method `M`
> I can just drop it in here.
>
> Another important example is how concurrency gives us the composition of
> independently executing computations.

About "programming in the large":

> Go was designed to help write big programs, written and maintained by big
> teams. (...) Big software needs methodology to be sure, but not nearly as much
> as it needs strong dependency management and clean interface abstraction and
> superb documentation tools, none of which is served well by C++ (although Java
> does noticeably better).

# Next in the list

- [ ] [Hashing](https://samwho.dev/hashing/)
- [ ] [Memory allocation](https://samwho.dev/memory-allocation/)
- [ ] [Google Go style guide](https://google.github.io/styleguide/go/)
- [ ] **[GopherCon 2021: Robert Griesemer & Ian Lance Taylor - Generics!](https://www.youtube.com/watch?v=Pa_e9EeCdy8&t=1250s)**
- [ ] [Go Proverbs](https://go-proverbs.github.io/)
- [ ] [TotT: Testing State vs. Testing Interactions](https://testing.googleblog.com/2013/03/testing-on-toilet-testing-state-vs.html)
- [ ] [TotT: Effective Testing](https://testing.googleblog.com/2014/05/testing-on-toilet-effective-testing.html)
- [ ] [TotT: Risk-driven Testing](https://testing.googleblog.com/2014/05/testing-on-toilet-risk-driven-testing.html)
- [ ] [TotT: Change-detector Tests Considered Harmful](https://testing.googleblog.com/2015/01/testing-on-toilet-change-detector-tests.html)
- [ ] [Go and Dogma](https://research.swtch.com/dogma)
- [ ] [Esmerelda’s Imagination](https://commandcenter.blogspot.com/2011/12/esmereldas-imagination.html)
- [ ] [Regular expressions for parsing](https://commandcenter.blogspot.com/2011/08/regular-expressions-in-lexing-and.html)
- [ ] [Gofmt’s style is no one’s favorite, yet Gofmt is everyone’s favorite](https://www.youtube.com/watch?v=PAAkCSZUG1c&t=8m43s)
- [ ] [Scale Web App](https://bytebytego.com/courses/system-design-interview/scale-from-zero-to-millions-of-users)
- [ ] [Caching strategies](https://codeahoy.com/2017/08/11/caching-strategies-and-how-to-choose-the-right-one/)
- [ ] [Non-relational databases](https://blog.teamtreehouse.com/should-you-go-beyond-relational-databases)
- [ ] [Scalability at Netflix](https://netflixtechblog.com/active-active-for-multi-regional-resiliency-c47719f6685b)
- [ ] [System design primer](https://github.com/donnemartin/system-design-primer)
- [ ] [Slack at scale](https://slack.engineering/flannel-an-application-level-edge-cache-to-make-slack-scale/)
- [ ] [Raft consensus algorithm](https://raft.github.io/)
- [ ] [How do Go channels work](https://levelup.gitconnected.com/how-does-golang-channel-works-6d66acd54753)
- [ ] [Organizing a Go module](https://go.dev/doc/modules/layout)
- [ ] [Structured logging](https://lukas.zapletalovi.com/posts/2023/about-structured-logging-in-go121/)
- [ ] [Advanced Go concurrency](https://encore.dev/blog/advanced-go-concurrency)
- [ ] [TOTP client in Go](https://rednafi.com/go/totp_client/)
- [ ] [Goroutines and buffered channels](https://rednafi.com/go/limit_goroutines_with_buffered_channels/)
- [ ] [Propagate context without cancellation](https://tyk.io/blog/how-to-propagate-context-without-cancellation/)
- [ ] [pipe and flow](https://rlee.dev/practical-guide-to-fp-ts-part-1)
- [ ] [Rust vs Go](https://www.shuttle.rs/blog/2023/09/27/rust-vs-go-comparison)
- [ ] [Deconstructing type parameters](https://go.dev/blog/deconstructing-type-parameters)
- [ ] [PRs about changesets](https://mitchellh.com/writing/github-changesets)
- [ ] [InfluxDB stack](https://old.reddit.com/r/rust/comments/16v13l5/influxdb_officially_made_the_switch_from_go_rust/k2qd8t9/)
- [ ] [Working without mocks](https://quii.gitbook.io/learn-go-with-tests/testing-fundamentals/working-without-mocks)
