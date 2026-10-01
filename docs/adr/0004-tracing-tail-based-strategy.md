# Default Tail-Based Sampling Strategy for Traces

## Status
**Proposed**  
#### Date  
2025-04-03
#### Owner  
[Denis Filatov](https://github.com/denifilatoff)
#### Participants and approvers  
- [IldarMinaev](https://github.com/IldarMinaev)
- [Alexey Karasev](https://github.com/asatt)
- [Vladimir Sitnikov](https://github.com/vlsi)
- [FedorProshin](https://github.com/FedorProshin)
#### Related ADRs  
- [0001: OpenTelemetry for Traces, Metrics, Logs](https://github.com/Netcracker/qubership-observability-operator/blob/main/docs/adr/0001-opentelemetry-for-traces-metrics-logs.md)  

## Context  
The current probabilistic sampling strategy (1-10% random sampling) fails to capture critical debugging data:
- Traces with errors are rarely retained  
- Long-running spans are systematically excluded  
- Full sampling is cost-prohibitive for storage/processing  

OpenTelemetry Collector (adopted per ADR 0001) provides two sampling approaches:  
1. **Head-based** (probabilistic/rate-limiting)  
2. **Tail-based** (post-trace decision-making)  

Key tradeoff: Tail-based sampling introduces computational overhead but enables intelligent retention of high-value traces.  

## Decision  
We will implement the [OpenTelemetry Tail Sampling Processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/tailsamplingprocessor/README.md) as the default strategy for DEV/QA/CI environments with these OOB configurations:  

| Parameter         | Value | Rationale                              |
| ----------------- | ----- | -------------------------------------- |
| Decision delay    | 31s   | Aligns with typical UI gateway timeout |
| Error retention   | 100%  | Ensure debug capability                |
| Long-span capture | >5s   | Identify performance issues            |
| Base sample rate  | 10%   | Balance cost vs utility                |

### Justification  
**Why Tail-Based Over Alternatives:**  
1. Captures full error traces (critical for debugging)
2. Retains performance-critical long spans
3. More cost-efficient than full sampling

**Performance Validation**:  
Tested ~200 traces/sec (50KB each = ~10MB/s load)

| Sampling policy name            | CPU consumption (millicores) | RAM consumption (Mb)     | Disc (Gb/Hour)        |
| ------------------------------- | ---------------------------- | ------------------------ | --------------------- |
| No Tail Sampling processor      | 169                          | 190                      | 36                    |
| **Tail Sampling (10% sampled)** | **165** (no overhead)        | **494** (x 2.5 overhead) | **3,6** (x 10 saving) |
| Tail Sampling (50% sampled)     | 325 (x 2 overhead)           | 466 (x 2.5 overhead)     | 18 (x 2 saving)       |
| Tail Sampling (100% sampled)    | 552 (x 3 overhead)           | 460 (x 2.5 overhead)     | 36                    |

The reason of high RAM consumption is buffer for traces before make a decision.
Raw theoretical calculation: 31 s * 200 traces/sec * 0.05 Mb = 310Mb overheads for buffer

We cannot clearly explain CPU overhead. But the reproductable trends are next:
- CPU strongly depends on Sapling rate (165 MIllicore for 10% --> 552 Millcore for 100%)
- CPU not depends on type of policy (Composite or none Composite). We perform test with 100% Probalilistic filter in Composite and standalone. CPU consumption was similar.
- CPU not depends on the number of subpolicies in Composite policy. We testes only 100% Probabilistic and 3 different sub-policies (attribute in trace, error, probabilistic). CPU consumption was similar. 
**For us it means that for Sampling rate near 1-10% CPU consumption will be closed to "none Tail Sampling" option.** 

### Rejected alternatives: 
- **Head-based sampling**: Fails core requirements for error retention  
- **Full sampling**: Prohibitively expensive at scale  

## Consequences  
**Positive:**  
- Guaranteed access to error traces for root-cause analysis  
- Systematic capture of performance-degrading long spans  
- 80% reduction in storage costs vs full sampling  

**Negative:**  
- Increased memory/CPU requirements for Collector  
- Added complexity in pipeline configuration  

**Neutral:**  
- Requires monitoring of Collector resource usage  
- Configuration must be environment-tuned (DEV vs PROD)  
