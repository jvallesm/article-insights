# Article insights

This document is a compilation of insightful articles about coding practices and
design patterns. In order to persist the key takeaways after the reading, I'll
write some conclusions for each article and add some linked resources.

## Table of contents
- [Article insights](#article-insights)
  - [Table of contents](#table-of-contents)
  - [Articles](#articles)
    - [Retry strategies](#retry-strategies)
      - [Discussion](#discussion)
  - [Next in the list](#next-in-the-list)

## Articles

### [Retry strategies](https://encore.dev/blog/retries)

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

#### Discussion

- How would the code snippet change if we wanted to retry only on certain HTTP
  codes?
- If a request is a blocking process in a client, we'll seldom want an infinite
  retry strategy. We might want to add a maximum retry count or a maximum delay
  in order to free that process.

## Next in the list

- [ ] [Hashing](https://samwho.dev/hashing/)
- [ ] [Memory allocation](https://samwho.dev/memory-allocation/)
- [ ] [Load balancing](https://samwho.dev/load-balancing/)