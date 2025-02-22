## Objectives

### 1. Establish Basic EIGRP Adjacencies

- **Description**: Configure EIGRP so that routers become neighbors on directly connected interfaces. Confirm that adjacencies form and routes are exchanged.
- **Why it matters**: Understanding how adjacency is formed is the foundation of EIGRP. You’ll learn the core commands and verify successful neighbor relationships.

### 2. Use EIGRP Named Mode (Modern Configuration)

- **Description**: Configure EIGRP in **named mode**, which consolidates IPv4 and IPv6 under a single process. Compare it briefly with “classic mode” if you like.
- **Why it matters**: Named mode is the more recent and flexible approach to configuring EIGRP and aligns with current best practices.

### 3. Verify and Tune EIGRP Timers

- **Description**: Explore **Hello** and **Hold** timers. Adjust them to see how EIGRP reacts to faster or slower intervals.
- **Why it matters**: Timing directly affects EIGRP’s convergence speed and stability. It’s a key element in network performance tuning.

### 4. Summarization (Auto & Manual)

- **Description**: Observe what happens with **auto-summary** (if supported by your IOS version), then implement **manual summarization** on specific interfaces.
- **Why it matters**: Summarization limits route advertisement scope, reduces routing table size, and improves convergence—essential in large networks.

### 5. EIGRP Stub Routing

- **Description**: Configure R4 and/or R5 as **stub** routers (e.g., `eigrp stub connected summary`) to see how they respond to and forward queries.
- **Why it matters**: Stub routing prevents queries from flooding to non-transit routers and is widely used in branch/edge scenarios.

### 6. EIGRP Authentication

- **Description**: Implement MD5 or SHA-based key authentication on selected interfaces. Ensure neighbors still form adjacency with the correct credentials.
- **Why it matters**: Security is crucial in production. EIGRP authentication prevents unauthorized or rogue EIGRP peers from injecting routes.

### 7. Filtering and Route Control

- **Description**: Use **distribute-lists** or **route-maps** to filter or modify routes inbound/outbound.
- **Why it matters**: Real-world networks often require selective route advertisement (e.g., not leaking private subnets). Filtering is a core skill in EIGRP administration.

### 8. Metric Manipulation and Load Balancing

1. **Offset Lists**: Adjust metrics to influence route preference.
2. **Variance**: Allow unequal-cost load balancing when multiple paths exist (e.g., R1→R2→R4 vs. R1→R3→R4).

- **Why it matters**: Manipulating EIGRP metrics is key for traffic engineering. Variance demonstrates EIGRP’s unique approach to balancing traffic across different cost paths.

### 9. EIGRP for IPv6

- **Description**: Assign IPv6 addresses to some interfaces and enable **EIGRP IPv6** (typically in named mode). Confirm neighbor formation and IPv6 route exchanges.
- **Why it matters**: IPv6 adoption continues to grow. EIGRP for IPv6 is almost identical to IPv4 but has subtle differences worth exploring.

### 10. Troubleshooting and Advanced Observability

- **Description**: Practice using **show** and **debug** commands (`show ip eigrp neighbors`, `show ip eigrp topology`, `debug eigrp fsm`, etc.). Simulate link failures or misconfigurations and analyze EIGRP’s behavior (e.g., handling SIA, queries).
- **Why it matters**: Real-world issues happen, and understanding how to pinpoint and fix EIGRP problems is essential for any network engineer.

## Instructions

### 1. Establish Basic EIGRP Adjacencies