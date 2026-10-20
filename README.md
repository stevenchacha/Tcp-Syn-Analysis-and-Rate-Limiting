
# TCP SYN Analysis and Rate Limiting

## Overview

I analyzed controlled TCP SYN traffic against an Apache web server in an isolated VirtualBox lab. I measured HTTP response times, inspected captured packets, checked TCP counters, and tested a temporary firewall rate limit.

The bounded probes did not demonstrate a denial of service. The firewall filtered SYN traffic, but it also caused legitimate HTTP requests from the same client to time out.

## Lab Environment

- **Virtualization:** Oracle VirtualBox
- **Network:** Internal Network named `dos-lab`
- **Client:** Kali Linux, `192.168.77.10`
- **Victim:** Kali Linux clone, `192.168.77.20`
- **Web server:** Apache
- **Tools:** hping3, tcpdump, curl, iptables, and nstat
- **Test date:** October 20, 2026

Both VMs used the same internal network with their other adapters disabled.

## Objectives

- Observe TCP SYN probe behavior.
- Measure HTTP availability before, during, and after testing.
- Check whether SYN cookies or listen-queue pressure counters increased.
- Evaluate a firewall rate limit and its effect on legitimate requests.
- Restore the original firewall configuration.

## Baseline

Apache was active and returned HTTP 200.

Ten normal HTTP requests produced:

- **Successful responses:** 10 out of 10
- **Average total response time:** 1.080 ms
- **Minimum:** 0.634 ms
- **Maximum:** 3.754 ms

Before testing, the victim had:

- `net.ipv4.tcp_syncookies = 1`
- `TcpExtSyncookiesSent = 0`
- `TcpExtListenOverflows = 0`
- `TcpExtListenDrops = 0`

SYN cookies remained enabled throughout the experiment.

## Bounded SYN Probe Test

The client sent up to 1,000 SYN probes with a requested interval of 10 milliseconds and a 15-second timeout:

```bash
sudo timeout -s INT 15 hping3 -S -p 80 -i u10000 -c 1000 192.168.77.20
```

The first run reported:

- 1,000 packets transmitted
- 1,000 replies received
- 0% reported packet loss

The victim captured:

- 3,000 packets
- 0 packets dropped by the capture kernel

Capture drops describe packet-capture loss, not firewall drops.

## Packet Analysis

The inspected packet sample showed this sequence:

1. The client sent a SYN.
2. The server returned a SYN-ACK.
3. The client sent a RST.

The client immediately reset these connections rather than completing a normal TCP handshake. This behavior did not demonstrate sustained half-open connection buildup.

After testing, all three checked TCP counters remained zero:

- SYN cookies sent
- Listen overflows
- Listen drops

These observations do not establish a successful SYN-flood denial of service or demonstrate SYN-cookie effectiveness.

## HTTP Requests During Probes

An initial manual overlap test returned HTTP 200 for all ten requests, averaging 1.650 ms. The exact overlap of every sample with the probes was not recorded.

I repeated the test with automatic sequencing: the probes ran in the background, and HTTP checks began after a one-second delay.

The five-request retest produced:

- **Successful responses:** 5 out of 5
- **Average total response time:** 1.596 ms
- **Minimum:** 0.730 ms
- **Maximum:** 3.075 ms

The HTTP checks completed before the background job reported completion, consistent with the intended overlap.

These small samples are descriptive observations, not evidence of a statistically significant performance change.

## Measurement Limitation

Later hping3 runs reported 976 packets transmitted but 1,000 received, producing a negative loss percentage.

I treated these inconsistent counts as unreliable and retained the output as a measurement limitation. I did not interpret negative packet loss as a valid result.

## Firewall Rate-Limit Experiment

The victim initially had ACCEPT policies and no added filter rules.

I created a temporary chain named `LAB_SYN_LIMIT` and applied it only to:

- Incoming interface `eth0`
- Source IP `192.168.77.10`
- TCP SYN traffic
- Destination port 80

The chain allowed an average of 20 matching packets per second with an initial burst allowance of 20. Excess matching packets were dropped.

The same client's normal HTTP connections shared this allowance with the SYN probes.

### Results With Filtering

The firewall counters recorded:

- **Accepted matching packets:** 281
- **Dropped matching packets:** 728

The five HTTP checks produced:

- **HTTP 200 responses:** 1
- **Timeouts:** 4
- **Timeout threshold:** 2 seconds

Curl displayed HTTP 000 for the timed-out requests. This means it received no HTTP response; 000 is not a server HTTP status code.

Hping3 reported 1,000 packets transmitted and 285 received. Its reply counts did not exactly match the firewall counters, so I did not equate those measurements.

### Interpretation

The firewall demonstrated filtering, but it did not improve measured HTTP availability.

The shared allowance caused legitimate connections from the test client to compete with the probes. This illustrates how a restrictive rate limit can harm legitimate traffic.

Because timeouts extended the HTTP loop, the filtered and unfiltered runs did not have identical sampling durations. Probe timing was not captured precisely enough to establish overlap for every filtered request.

## Cleanup and Recovery

I removed the INPUT jump and deleted the temporary `LAB_SYN_LIMIT` chain.

The firewall returned to:

```text
-P INPUT ACCEPT
-P FORWARD ACCEPT
-P OUTPUT ACCEPT
```

With no probes running and the temporary rules removed, five recovery requests produced:

- **Successful responses:** 5 out of 5
- **Average total response time:** 1.459 ms
- **Minimum:** 0.646 ms
- **Maximum:** 4.390 ms

This confirms successful HTTP access after cleanup. Since both the probes and filtering had stopped, this recovery check does not isolate the effect of either change.

## Evidence

Screenshots document:

- Initial HTTP response and ten-request baseline
- First SYN probe summary
- Packet-capture statistics
- SYN, SYN-ACK, and RST packet sequence
- TCP counters
- Automatically timed HTTP retest
- HTTP timeouts with filtering enabled
- Firewall ACCEPT and DROP counters
- Successful HTTP requests after cleanup

The earlier manual overlap run and inconsistent hping3 summary are supplementary evidence if included.

## Limitations

- This was a small, single-client virtual lab, not a distributed attack.
- No service outage was demonstrated during the unfiltered bounded test.
- The inspected packet sample showed immediate client resets.
- SYN cookies were enabled but their sent counter remained zero.
- Small sample sizes and VM scheduling limit timing conclusions.
- The same client generated probes and legitimate requests.
- The firewall experiment demonstrated filtering and collateral impact, not production-ready DDoS protection.

## Skills Demonstrated

- Isolated virtual-network configuration
- Linux network troubleshooting
- TCP flag and packet analysis
- HTTP availability measurement
- Firewall configuration and counter interpretation
- Controlled testing and cleanup
- Evidence-based technical documentation

## What I Learned

Traffic volume alone does not prove a successful denial of service. Packet behavior, HTTP results, and server counters must be considered together.

I also learned that dropping more traffic does not automatically mean a defense is better. A useful control must be evaluated for its effect on legitimate users.
