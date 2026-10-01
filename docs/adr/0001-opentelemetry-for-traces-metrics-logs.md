# OpenTelemetry format for Traces, Metrics, Logs

## Status
Accepted  
#### Date
2025-01-31
#### Owner
[Denis Filatov](https://github.com/denifilatoff)
#### Participants and approvers
- [pankratovsa](https://github.com/pankratovsa)
- [shumnic](https://github.com/shumnic)
- [Alexey Karasev](https://github.com/asatt)
- [IldarMinaev](https://github.com/IldarMinaev)
- [Vladimir Sitnikov](https://github.com/vlsi)
- [egbu](https://github.com/egbu)
#### Related ADRs
None

## Context
In the Qubership platform, there is a demand to standardize the formats for Metrics, Traces, and Logs. There are over a hundred services, based on various languages and libraries, from which telemetry data needs to be collected in order to provide useful insights for OPS teams. 

Qubership will also be used by business applications, which similarly require telemetry data collection. This will be achieved through both auto-instrumentation and custom instrumentation within the applications themselves. 

Telemetry data needs to be viewable in both Qubership’s Observability Backends and leading market APMs, with Qubership not having control over which backend the business applications choose. 

Additionally, a format that is compatible with modern LLMs (Large Language Models) is necessary for seamless integration where applicable, allowing easy use of AI-driven insights and analytics.

## Decision
We will adopt OpenTelemetry as the sole framework for collecting and exporting Metrics, Traces, and Logs across all services in the Qubership platform.

### Justification
- OpenTelemetry is widely adopted and supported by numerous tools, reducing the risk of vendor lock-in and future-proofing our system. (OpenTelemetry Ecosystem: https://opentelemetry.io/ecosystem/registry/) 
- Most modern languages and technologies already support OpenTelemetry SDKs for code-based instrumentation. (https://opentelemetry.io/docs/languages/)
- Using OpenTelemetry allows us to leverage existing collector pipelines for consistent data management and reduces the overhead of maintaining proprietary solutions (https://github.com/open-telemetry/opentelemetry-collector)
- Major APM vendors support OpenTelemetry, providing flexibility in how we choose to monitor and analyze our data.
	- Dynatrace:  https://www.dynatrace.com/integrations/opentelemetry/
	  - New Relic: https://newrelic.com/solutions/opentelemetry
	  - DataDog: https://docs.datadoghq.com/opentelemetry/
	  - Splunk:  https://www.splunk.com/en_us/solutions/opentelemetry.html

Alternatives considered:
- In telemetry data layer there are no any vendor-agnostic format or it's outdated. Specific cloud providers or APM vendors suppors their own formats.

## Consequences
- We will rely on OpenTelemetry-compatible tools and libraries across our tech stack.
- Our observability system will be cloud-agnostic and compatible with multiple APM vendors.
- We will have to train teams on using OpenTelemetry tools and maintain compatibility with future OpenTelemetry updates.
