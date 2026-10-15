# Structure
1. The challenge: the low-level algorithms shouldn't slow down, any performance downgrade in SQL function core implementations, core data structure algorithms, data storage structures should fail loudly
2. Proposed solution: GoogleBenchmark integrated into CI/CD on bare metal server
3. Caveats: SIMD benchmarking
4. Impact: 10% of weighted code is covered


## Fast queries need fast building blocks

An SQL database manager software runs on different layers of abstraction to be fast. You can't implement SQL functions without parser wrapping or store data in plain rows. So benchmarking queries would be insufficient to get the right picture of how efficient its code is written.

I had to introduce performance testing for low-level algorithms to catch a slowdown in the pull request that caused it. A bare metal server runs the tests with a fixed CPU frequency and turbo boost off. Every test goes through a warm-up stage and a tuned number of repetitions, which keeps the run-to-run variance under 1%.

## Flag every slowdown in review

The team already established CI/CD framework for functionality testing and I aimed to expand it for performance tests. Performance tests for such low-level algorithms can be set on a fixed machine as threshold tests which was aligned with the project's QA approach.

I integrated Google Benchmark into the project and developed custom metrics to be checked with its comparison tools. I set thresholds on the median and the worst-case runtime instead of the average. The worst-case runtime models the worst user experience and the median models the expected algorithm performance. Each metric must stay within a 5% margin of a stored baseline.

More than 1000 benchmarks run sequentially on every pull request in under 10 minutes. Pull requests wait in a queue for the server. Only the release compilation preset is tested. A failed run doesn't block the pull request; it triggers an extra review. According to the CI/CD history, under 10% of runs are false positives. If a change is slower on purpose, the developer can rewrite the baseline value, but only after another developer approves the change. In practice, baselines are rewritten only after hardware updates.

One limitation is known: there is no routine yet to update the baseline when the code gets faster. A speed-up leaves extra room under the threshold, so a later slowdown of the same size can pass unnoticed.

## Test where SIMD code breaks

Most low-level functions behave the same way for any input, but SIMD code doesn't. It processes data in vectors of a fixed width, so its speed depends on how the input fits into these vectors.That is why I set up dedicated benchmarks for the SIMD algorithms.

Some of these benchmarks override the project's main build target and run once per instruction set: SSE, AVX2 and AVX-512. The product ships with only one of them, but the project prepares to switch targets, and the benchmarks show the cost of the switch in advance.

The benchmarks vary three things. The first is the input size around the vector width: one element less or more than a full vector or a full unrolled loop, and odd sizes. The second is memory alignment: the input starts at every byte offset inside a vector, single vector loads fall across the boundary between two 64-byte cache lines or two 4 KB pages, and source and destination buffers sit exactly 4 KB apart, which causes 4K aliasing. The third is the data itself. Functions that stop early, such as find or string parsing, get the match at the start, in the middle, at the end, or nowhere. Functions with masks and branches get all-true, all-false and random data. Floating-point functions get NaN and infinity. These benchmarks were generated with AI and cross-verified by it.

## Cover the code customers run

The number of benchmarks says little about protection: a thousand benchmarks of rarely used code protect nothing. So I measure coverage weighted by the CPU time that each function takes in customer workloads.

The weights come from production profiles. Perf records customer workloads, and I count the samples in which each function itself runs, the self samples. Inclusive time would count the time of a caller twice. The coverage comes from a coverage build of the benchmark suite: llvm-cov exports the list of functions that the benchmarks execute. Inlining is the main pitfall here: an inlined function disappears from the sampling profile unless the profiler resolves inline frames.

The weighted coverage is the share of self samples that falls into covered functions. The uncovered functions, sorted by their share, form the backlog for new benchmarks.

The project started at 0%. After two months, the benchmarks cover 10% of the weighted code, and all of it falls into the hottest functions. I recalculate the metric at each milestone, every few months, to check that the coverage doesn't drop. No regression has been caught yet, so the value so far is prevention.
