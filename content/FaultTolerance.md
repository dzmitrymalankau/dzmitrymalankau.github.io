## Fault tolerance in distributed systems

#### Redundancy and Replication
*Data Replication:* Data is duplicated across multiple nodes or locations to ensure availability and durability. If one node fails, the system can still access the data from another node.
*Component Redundancy:* Critical system components are duplicated so that if one component fails, others can take over. This includes redundant servers, network paths, or services.

#### Failover Mechanisms
*Active-Passive Failover:* One component (active) handles the workload while another component (passive) remains on standby. If the active component fails, the passive component takes over.
*Active-Active Failover:* Multiple components actively handle workloads and share the load. If one component fails, others continue to handle the workload.

#### Error Detection Techniques
*Heartbeat Mechanisms:* Regular signals (heartbeats) are sent between components to detect failures. If a component stops sending heartbeats, it is considered failed.
*Checkpointing: Periodic* saving of the system's state so that if a failure occurs, the system can be restored to the last saved state.

#### Error Recovery Methods
*Rollback Recovery:* The system reverts to a previous state after detecting an error, using saved checkpoints or logs.
*Forward Recovery:* The system attempts to correct or compensate for the failure to continue operating. This may involve reprocessing or reconstructing data.
