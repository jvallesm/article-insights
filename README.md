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

## [Google's Go style guide](https://google.github.io/styleguide/go/)

- No `Get` in getters
  ([source](https://google.github.io/styleguide/go/decisions#getters))
- No [line length](https://google.github.io/styleguide/go/guide#line-length)
- No [util
  package](https://google.github.io/styleguide/go/best-practices#util-packages)
- Use [option
  structs](https://google.github.io/styleguide/go/best-practices#option-structure)
  for long lists of arguments.

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

# Next in the list

- [ ] [Hashing](https://samwho.dev/hashing/)
- [ ] [Memory allocation](https://samwho.dev/memory-allocation/)
- [ ] [Load balancing](https://samwho.dev/load-balancing/)
- [ ] [Google Go style guide](https://google.github.io/styleguide/go/)
- [ ] **[GopherCon 2021: Robert Griesemer & Ian Lance Taylor - Generics!](https://www.youtube.com/watch?v=Pa_e9EeCdy8&t=1250s)**
- [ ] [Go Proverbs](https://go-proverbs.github.io/)
- [ ] [TotT: Testing State vs. Testing Interactions](https://testing.googleblog.com/2013/03/testing-on-toilet-testing-state-vs.html)
- [ ] [TotT: Effective Testing](https://testing.googleblog.com/2014/05/testing-on-toilet-effective-testing.html)
- [ ] [TotT: Risk-driven Testing](https://testing.googleblog.com/2014/05/testing-on-toilet-risk-driven-testing.html)
- [ ] [TotT: Change-detector Tests Considered Harmful](https://testing.googleblog.com/2015/01/testing-on-toilet-change-detector-tests.html)
- [ ] [Go and Dogma](https://research.swtch.com/dogma)
- [ ] [Less is exponentially more](https://commandcenter.blogspot.com/2012/06/less-is-exponentially-more.html)
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
