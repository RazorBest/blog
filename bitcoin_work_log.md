# Bitcoin Work Log

# March-April 2026

My plans for April and May are to work at [peer-observer](https://github.com/peer-observer/peer-observer) and [bip324-mitm](https://github.com/RazorBest/bip324-mitm). The time shares will be around 80%/20%.

For peer-observer, there are two main tracks that I have: anomaly detection with Prometheus, and improving the performance of the ebpf-extractor. If I have new ideas for ebpf-extractor, I will write them here: https://bnoc.zulipchat.com/#narrow/channel/563464-peer-observer-dev/topic/ebpf-extractor-discussion.
For anomaly detection, I already started looking at some metrics for which the anomaly detection would work. I will look for historical data where more interesting events happened and see if the anomaly detection works well. Other than that, we'll have to see how big are the false-positives by letting these recording rules run on our nodes.
After I get some feedback, I can try a second iteration where we focus on the more imporant metrics. I'd say that finding even one metric for which this works would be a success. Because, after that, adding new metrics for anomaly detection will not be as hard.

For bip324-mitm, there are two goals. First, to merge the current PR that separates the mitm layer from the protocol layer, with the help of @muchai254, tracked by https://github.com/RazorBest/bip324-mitm/issues/5. Second, to add a hook-like functionality where the user can read the P2P intercepted messages; tracked by https://github.com/RazorBest/bip324-mitm/issues/11.

## PRs
- Added a dynamic reload functionality to the ebpf extractor of peer-ovbserver: https://github.com/peer-observer/peer-observer/pull/408
- Provided a first implementation of generic anomaly detection for peer-observer, with using Prometheus: https://github.com/peer-observer/peer-observer/pull/400.
- Fixed an integer overlflow parsing error in corepc: https://github.com/rust-bitcoin/corepc/pull/547.
- A small fix that decreased the build times of the rust crates that depend on the shared crate in peer-observer: https://github.com/peer-observer/peer-observer/pull/393
- Documented what rpc methods are used by the rpc extractor and wrote a CI test that detects changes: https://github.com/peer-observer/peer-observer/pull/390

## Reviews for peer-observer
- https://github.com/peer-observer/peer-observer/pull/386
- https://github.com/peer-observer/peer-observer/pull/383
- 26.04.2026 - Reviewed https://github.com/peer-observer/peer-observer/pull/373
- 27.04.2026 - Reviewed https://github.com/peer-observer/peer-observer/pull/418

## Discovered issues
- Opened an issue in libbpf-rs after discovering a use-after-free bug when working on the ebpf extractor of peer-observer: https://github.com/libbpf/libbpf-rs/issues/1381
- Discovered a small regression bug in the Ubuntu Docker image of Grafana: https://github.com/grafana/grafana/issues/122626


## BIP-324 analysis (my personal project)
- Reviewed the separation between the bip324 protocol and the mitm logic: https://github.com/RazorBest/bip324-mitm/pull/10
- Represented the errors in the bip324-mitm library using enums: https://github.com/RazorBest/bip324-mitm/pull/6/changes
- Rewritten the bip324-mit library using a more structured state machine: https://github.com/RazorBest/bip324-mitm/pull/10
- Catch the errors in the python scripts of the main repo that kept crashing the docker containers: https://github.com/RazorBest/bip324-traffic-analysis/pull/16; https://github.com/RazorBest/bip324-traffic-analysis/pull/15.
- Decrease the storage used by Prometheus by limiting it to 5 days: https://github.com/RazorBest/bip324-traffic-analysis/pull/14
- 24.04.2026 - Reviewed https://github.com/RazorBest/bip324-mitm/pull/10

# May 2026

## Work
- Written blog about Prometheus' increase function: https://github.com/RazorBest/blog/blob/main/data_science/prometheus_increase/analyze_increase1.md

## Reviews for peer-observer
- 22.05.2026 - https://github.com/peer-observer/peer-observer/pull/438
- 22.05.2026 - https://github.com/peer-observer/peer-observer/pull/418
- 29.05.2026 - https://github.com/peer-observer/peer-observer/pull/420

## BIP-324 analysis (my personal project)
- 24.05.2026 - Reviewed https://github.com/RazorBest/bip324-mitm/pull/10
- 28.05.2026 - Reviewed https://github.com/RazorBest/bip324-mitm/pull/10
