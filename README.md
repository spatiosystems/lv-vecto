# Vecto

*Latin for "carry" or "convey".*

Vecto is a distributed system communications library optimized for both reliable commanding and efficient periodic telemetered data by utilizing 3 primary links for data. Configuration and logs are transferred via HTTP, commands are transferred via TCP, and periodic telemetered data is transferred via multicast UDP. This ensures that larger transfers like config files, test data logs, and other larger transactions do not interrupt command channels or telemetered data.

To ensure synchronization across the transports, all data is correlated with a sequence number to identify data which may be stale, such as data updates being processed after newer scaling configurations have been transferred; the earlier data with invalid scaling can be discarded. While time sync will be beneficial to communications and payloads are expected to be timestamped for measuring latency and other system characteristics, the sequencing counts alone will be used for staleness determinations to avoid situations that have been seen when time sync gets screwey with other comms systems.

While the base library won't dictate configuration directly it does provide capabilities related to configuration management and expects to report software and configuration versions throughout a system. As such it will be up to applications (nodes) to supply versions as appropriate to the Vecto for proper reporting. As Vecto is the part of the software that interacts between all the separate applications, it is sensible to make these version details published as part of node identification with some functionality to allow applications to identify mismatched / unexpected configurations and block/limit functionality appropriately.

## Design Pillars

### Ahead of Time Confguration

While some amount of automatic discovery will take place for the various components in a system, all of the data, commands, and high level system organization shall be configured ahead of time and shared with every component. This prevents issues such as GUIs being created incorrectly and not being able to produce helpful error messages because it can't determine if data not being available is due to another piece of software not running, network faults, or bad configuration. Other issues such as channelization configuration mismatches that can cause silent data alignment problems can also be detected by ensuring configurations across a system are in sync and deployed properly. When mismatches are detected the components can complain loudly and prevent operating the system in a misconfigured state.

### Periodic Telemetry is Lossy

Most of the communication in an active test system will be telemetered data sent at regular intervals. As this is the information that is used for situational awareness, performance and latency are the biggest priorities for this type of data so Multicast UDP will be used to share this data. This eliminates the situation where a central message broken can be a single point of failure and also eliminates the need to maintain peer-to-peer connections for every node that is interested in the data. If a packet of data is lost the expectation is that newer data will be arriving shortly regardless and problems only arise during sustained data loss.

### Periodic Telemetry is Grouped

Groups of telemetered data that originate from a single process and update at the same rate shall be grouped together for more efficient configuration and communication. As an example, within a closed loop control component, all intermediate values, parameter selection criteria, output results, and any other values used/generated during an interaction should all be collected and published together.

### Commands are Targetted and Transactional

When a command is issued, the sender should know the target of a command and address the message appropriately. To facilitate easier implementation of the communication libraries, commands are transferred via TCP via a centralized message broker. This eliminates the possibliity of commands being handled out of order, provides the ability to perform centralized logging and authorization of all commands in a cluster, and provides a reliable channel for heartbeats and node connection status reporting. Since periodic data is not also streamed via the message broker, this should remain efficient and latency issues are not expected with suitable implementations. A previous test system with dozens of connected clients to a central broker was able to keep up with 24000 active telemetered values being published at 5Hz along with all of the other command and control messaging using the same transports.

Note: For simpler systems, ideally it would also be possible for a single GUI application to directly communicate with I/O node(s) when larger scale distributed communications are not needed. In this situation there wouldn't be a separate message broker node but that functionality could be built into the GUI application directly. This would ensure that smaller systems with smaller channel counts require much less configuration and software setup.

### Command/Response Communications are Network Organization Agnostic

At no point should a component have to know the endpoint address of another component to address a command to. Internally message routing will rely on integer IDs to faciliate array lookups when possible which is very efficient for dispatching messages.

## Organization

An entire system is broken up into several layers of components:

### System

The entire collection of software that enables conducting tests for a specific test stand.

### Cluster (Domain)

Clusters are groups of nodes directly interacting with each other. The Domain nomenclature is specific to the network addressing configurations and may be used when describing how communication and message routing is configured. Nearly all systems will be comprised of a single cluster but nodes can be separated for organizational or performance needs. For instance, having a dedicated cluster for a shared testing resource, such as shared facility components and commodity supply control, can be bridged to separate test cells supplied by the common facility components. However this configuration can get complex quickly and in practice is not likely to be necessary. As such there will likely be little configuration / implementation around this aspect until it's actually needed.

### Node

A node is a single application that generates and/or consumes data. Nodes can be comprised of any amount of functionality such as data acquisition, control logic, logging, automation, communication, and user interfacing. Nodes are the first standard level of command addressing within a cluster and is the level of the hierarchy where connection health is monitored and reported.

A node is the most granular level for a configuration version to apply to; tasks within a node will not have more granular tracked configuration versions.

### Task

A task is a software component that has specific functionality implemented. Examples could be logging, command routing, closed-loop-control, DAQmx I/O, etc. A task can also have its status and health reported but generally this will be available for troubleshooting purposes. Tasks are the second standard level of command addressing within a cluster and any further addressing and parameter information will be task specific.

### Group

Groups are collections of channels that are updated together to the cluster. It is not strictly necessary that all channels within a group be sourced at the same data rate as some intermediary may collate chanenls from multiple components and publish them as a single group. An example of this could be a Node's health monitoring that collects the state of all tasks and publishes that data as a single group for that node. The different tasks may operate and update status within the node at different rates but this data can then be collated and decimated to a lower rate for network publishing. The vast majority of groups will be comprised of data generated from a single source / driver and all be sourced at the same rate. And example of this would be a DAQmx task that generates analog input data all at 1KHz that gets collected and published over the network at 5Hz. It is also not required for a group to be regularly published. If a group contains String channels it is highly recommended that those groups are published at slower rates, e.g. 1Hz or less.

### Channel

Channel represents a single data element that can be either an input or output from some task. A channel can be for any kind of data from analog inputs or outputs, digital inputs or outputs, closed loop control state, task iteration rate, logging status, etc.

Channels can be several types: BOOL (U8), I32, U32, I64, U64, Single, Double, or String. As mentioned, string channel handling is done a little differently from all of the numeric types due to their variably sized nature and how that can impact performance. Strings are helpful for reporting values such as configuration checksums, status strings, component names, logged-in user, etc. Whenever possible, string values should be mapped to some kind of integer ID value to avoid the costs of string handling and variably sized memory operations.

## Communication Transports

There are 3 tiers of communication transports (really its 4), each more or less following a priority hierarchy. The highest priority is the commanding transport which is done over TCP. This is effectively "high" priority with lossless communication. Next is telemetered data publishing which is "high" priority but lossy. Last is HTTP transactions which are "low" priority and should yield as much as possible to all other functionality.

Additionally there are other communication channels used for discovery and gossip protocols used for a system to establish connections and high level configuration information. For the sake of labeling, these can be viewed as "Normal" priority but follow different enough communication schemes that the priority really doesn't apply as well. These discovery capabilities prevent the need to rely on hard-coded (configured) endpoint addresses for components in the hopes of making a system more robust against network addressing changes.

### HTTP

One initial design decision is to use HTTP for transactional exchanges outside of the scope of commands. HTTP communication can be used for querying status, configuration information, transferring files, and other transactions that do not require timely handling or centralized control. HTTP overhead can be substantial for small exchanges so it is better suited for larger exchanges but as long as header content can be kept to a minimum and HTTP transactions are not performed frequently it should provide minimal impact.

A benefit of providing HTTP access is that it is incredibly easy to create tooling in nearly any environment or language that can perform HTTP requests so that debugging tools, status dashboards, and more can leverage the HTTP communication for messaging. External interfacing like this can have inadvertent performance impacts however so it is important to keep the HTTP usage to the minimum required, particularly for embedded RT targets. Centralized configuration management can push configurations to nodes over HTTP (with proper lockouts to prevent inadvertent updates during operations)