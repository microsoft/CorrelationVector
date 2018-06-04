# Correlation Vector - Scenarios

## Background

**Correlation Vector** (a.k.a. **cV**) is a Microsoft tracing standard based on a light weight vector clock. The standard is widely used internally at Microsoft for first party applications and services and supported across multiple logging libraries and platforms (Services, Clients - Native, Managed, Js, iOS, Android etc). The volume of cV instrumented trace data generated daily is in excess of a Petabyte and powers a variety of different data processing needs ranging from distributed tracing & debugging to system and business intelligence.

Tracing formats like the cV that are built around vector clocks make different design tradeoffs and have significant differences from those used in alternative tracing systems. This document covers some of the use cases and design constraints that shaped the cV's design and the unique capabilities it provides.

## Design Constraints

Use cases by themselves do not shape a solution. To appreciate the vector’s design, it is important to understand the various constraints imposed on the design of a tracing construct, in conjunction with the need to address the scenarios explored in this document. These constraints are driven by organizational realities many of which are not unique to Microsoft and as such apply to a variety of organizations.

1. Client sourced telemetry: Organizations critically depend on telemetry data, sourced from components operating outside the Data Center (such as Apps, Websites, Agents on Devices etc.). Such data is often uploaded to processing pipelines inside the data center via metered connections which places utmost importance on *wire efficiency*. Such data transfer is also *unreliable*, *susceptible to delays* and may reflect local clock times that have significant variance from official time servers. A trace that spans such sources as well as sources within the data center will inevitably have gaps due to missing or delayed events.

2. Event sampling: Telemetry is often sampled for further processing to lower cost. Sampling strategies are dictated by business priorities. Often, such strategies will incur gaps in a trace which imposes challenges for fully exploiting such partial traces. Moreover, such strategies may not even be trace centric – i.e. it may not prioritize keeping traces intact over other considerations.

3. Disjoint/Isolated Event Sinks: Telemetry is often collated into event sinks in the data center. In organizations that are computationally heterogenous, it is typical to have disjoint or isolated event sinks for a host of several reasons including privacy, organizational boundaries, protection of intellectual property, hybrid systems spanning cloud and in-house deployments etc. The implications of such a reality is that telemetry from a single operational flow may end up in different sinks. In such a situation, being able to causally relate events available in a single sink without the need to compile trace events from other sinks, avoids the additional overhead of gathering data from other sinks during processing.

4. Business/System Intelligence Workloads: Telemetry emitted during a trace may be used in data processing that feed into several different business-related metrics and KPIs. A typical pattern for consumption involves selecting a specific subset of events in a trace, that are transformed into one or more fact tuples, which are subsequently aggregated. Often, this processing will require an understanding of how these events within the same trace are *causally* related to each other. Ideally, it should be possible to do this without reconstructing the full trace.

5. Framework Heterogeneity/Lack of RPC standardization: Organizations may have considerable diversity in the platforms, frameworks & SDKs that are used for building clients and services. Seemingly simple use cases may involve a variety of such clients and services with may also belong to different technology eras. In the absence of a single RPC component (or a small number) that forms the basis of these different frameworks in such organizations, there is a significant developer overhead for adding support for instrumentation. In such an environment, any tracing construct would have to be easily portable, relatively simple to implement and easy to audit for violations.

To summarize the impact of these constraints; organizations need to *reliably causally relate events* from _**partial traces**_ from a diverse set of components, some of which may be built on frameworks that are quite unlikely to be updated to support a unified tracing protocol. Additionally, some of these sources may be overly sensitive to wire cost overhead involved in addressing this gap.


## Use Cases

### Trace Based Debugging

Providing (partially) ordered traces of available events spanning client and services for debugging purposes. The available data maybe retrieved from event sinks with as little delay as possible from the time they were generated. The trace reconstruction should be able to establish a sort with causality based on just the available events. 

### Incident Triangulation & Impact Assessment

Large scale distributed systems may involve thousands of APIs, involving hundreds or thousands of developers across dozens of teams working on individual micro services, UX components, back end storage and business processing tiers etc. to fulfill a variety of customer and business use cases. When components fail in such a system, its impact maybe experienced by end users in a variety of ways. Often times, failures can have ripple effects up the service stack and trigger a wide range of alerts across multiple customer surfaces.

The relevant use case is to provide support engineers a prioritized list of APIs/Components to investigate, when alerted to the possibility that the system is experiencing issues. The basis for this near real time prioritization is a quantifiable understanding of how components _**depend**_ on each other, derived purely from telemetry. Such a _dynamically inferred dependency model_ would support automated diagnostics that shorten the **Mean Time to Understand** the root issue and mitigate accordingly.

Beyond identifying the root cause of an incident, typical post incident processes will attempt to quantify the actual impact to the end users of the system. Here again, being able to attribute failures in the stack all the way to its impact on customer surface is a key requirement.

### System Intelligence

There are broad applications in isolating performance bottlenecks, attribution of capacity consumption, failure mode analysis, security audits etc. where a _dependency model_ inferred from telemetry is useful. Any major update in a Service stack such as a modification of an existing API or the introduction of new APIs, typically raises questions around the impact of those changes across a number of engineering concerns.

The use case would be to provide relevant data and visualization tools that help devops engineers reason over the dependencies across the system. Parts of the system may have telemetry gaps. End users may only be interested in reasoning over the relationship between specific parts of the system, such as between public facing APIs and the internal APIs that they are responible for.

### Funnel analysis

Funnel analysis is a popular way of assessing the health of customer scenarios. Such analysis requires identifying key events from potentially different layers in the stack and associating them to form tuples which would be indicative of the extent to which the funnel is completed for a target scenario.

The use case would be to provide the means to reliably associate any arbitrary set of causally related events emitted from any number of components from anywhere in the stack.

## Benefits of the Vector

To tackles the above-mentioned scenarios under the given constraints, the vector has the following key benefits:

1. The vector supports a sort. The vector associated with any two events in the same trace can be used to establish or rule out a causal relationship between them. Moreover any arbitrary subset of events in the trace can be sorted. Without a sort capability, it is not possible to exploit partial traces or rely on just selected events for the kinds of scenarios listed above.

2. For large telemetry-based systems that rely on client telemetry, like those found in first party systems (or potentially those in the IoT space), the vector’s wire format is designed to be cheaper for components higher up in the trace. Typically, telemetry sourced from client may have only 1 - 2 elements, each of which may have 1-2 decimal digits. In contrast, typical Dapper like implementations maintain a constant 8-byte allocation for the Span Id. 

    For reference, at Microsoft, the average length of the Vector across the entire corpus of generated cVs in the entire ecosystem on a given day, is a little over 3 elements. The 90th percentile is 4, and the 99th percentile is 9 elements.

3. The cV is embodied in a single string field and has typically the same format on the wire as well as in its serialized state. It is easy to port and implement from a scratch if need be. Its protocol is simple. This lets developers work around gaps in framework support if required. With standardized event schemas to descibe a span, its use is also straightforward to audit.