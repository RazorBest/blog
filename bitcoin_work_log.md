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

# June 2026

# Work
- Fixed a bug in nfq-rs at the netlink layer that was causing modified packets to be dropped if intercepted at INPUT: https://github.com/nbdd0121/nfq-rs/pull/18
- Provided some data analysis for generic anomaly detection for peer observer. Plotted mean and standard deviation for various windows, for data sampled from demo.peer.observer and implemented a simple algorihtm for choosing the best window: https://github.com/peer-observer/peer-observer/issues/304#issuecomment-4772838037.

## Reviews for peer-observer
09.06.2026 - https://github.com/peer-observer/peer-observer/pull/418

## BIP-324 analysis (my personal project)
- 06.06.2026 - Merged https://github.com/RazorBest/bip324-mitm/pull/10 (credits to @muchai254)
- 17.06.2026 - Improved bip324-mitm: https://github.com/RazorBest/bip324-mitm/pull/19
    - Added a full working example that performs a MitM between two Bitcoin nodes using nfq-rs
    - Added `examples/test_bip324.rs`, that showcases the bi324 protocol parser, separately from the mitm library
    - Added `UserPacketRelay` in `relay.rs` (including related structures), so users of the library can actually see the packets. This one is read-only.
    - Added tests for the user packet relay (`mitmbip324_component_tests`)
    - `FakePeerRelay` now uses `remove_first_packet_if_empty()` when pushing packets, to clear the queue. Before, it was using `remove_first_packet_if_consumed()`, which included the additional condition that the packet should not be the last one in the queue. This was leftover logic from a previous implementation.
    - Added tests `test_client_sends_one_message_after_terminator` and `test_server_sends_one_message_after_terminator`.
    - Added `BIP324Mitm::ensure_terminator_not_split`, which assumes that the user of the library will not push the terminator by parts. This is useful when the user wants that, for each byte pushed to the mitm, one byte should be immediately available for pulling. The garbage terminator is a special case, because the mitm needs to replace the real terminators with its own ones, while also preserving the original garbage. This means that, theoretically, the mitm doesn't know if the last 15 bytes of the garbage phase are part of the garbage or the terminator. It has to receive another byte to confirm that.
    - Fixed the calculation of boundaries when moving garbage from buffers to buffers in `src/bip324/mod.rs`.
    - Added `bip324::test_util::WriteParser` and `bip324::test_util::ReadParser` that combine the handshake and data phases into one interface. Currently, it's only used for testing, but it's solves part of https://github.com/RazorBest/bip324-mitm/issues/14.
    - Added MIT license
- 29.06.2026 - Reviewed and merged: https://github.com/RazorBest/bip324-mitm/pull/22 (credits to @ajlrza)


# July 2026

## BIP-324 analysis (my personal project)
- 06.07.2026 - Started a first iteration of `snarf-rs`, a helper library for packet interception that should help implementing the bip324 interceptor: https://github.com/RazorBest/snarf-rs/pull/1
- 21.07.2026 - [Finished](https://github.com/RazorBest/snarf-rs/pull/1) the first iteration of `snarf-rs`.
    - I designed a generic API for network packet interception that lets you insert interceptors at different layers in the stack
    - I implemented and tested a TCP helper layer for out-of-order packets
    - Next step would be to implement a better Seq Indexed Ring Buffer, than will be used by the interceptor.

# August 2026

## BIP-324 analysis (my personal project)
- 08.08.2026 - Improvements session 2 for `snarf-rs`: https://github.com/RazorBest/snarf-rs/pull/8 (A lot of small feats and fixes)
- 09.08.2026 - Add byte by byte access to the decrypted packets: https://github.com/RazorBest/bip324-mitm/pull/27
- 13.08.2026 - Improvements session 3 for `snarf-rs`: https://github.com/RazorBest/snarf-rs/pull/9
- 14.08.2026 - Handle the RST TCP flag for `snarf-rs`: https://github.com/RazorBest/snarf-rs/pull/10

## Other
- 04.08.2026 - Reviewed https://github.com/rust-pcap/pcap/pull/399

# September 2026

## BIP-324 analysis (my personal project)
- 11.09.2026 - Allow RST flag in more cases and improvements at the TCP layer for `snarf-rs`: https://github.com/RazorBest/snarf-rs/pull/13

## Other
- 09.09.2026 - Created satsmonitor.org, a monitoring service for RoboSats: https://github.com/RazorBest/robosats-monitor
