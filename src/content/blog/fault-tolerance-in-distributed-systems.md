---
title: Fault Tolerance in Distributed Systems
description: Patterns and techniques for building fault-tolerant systems
pubDate: 'Oct 05 2024'
heroImage: '../../assets/hero-fault-tolerance.svg'
---

## Fault Tolerance in Distributed Systems

Building systems that gracefully handle failures is essential for production applications. Fault tolerance ensures that systems continue operating when individual components fail, maintaining availability and preventing cascading failures across the entire system.

## Redundancy and Replication

### Data Replication

Data is duplicated across multiple nodes or locations to ensure availability and durability. If one node fails, the system can still access the data from another node.

**Replication Strategies:**

- **Master-Slave**: One primary node accepts writes, replicas handle reads
- **Master-Master**: Multiple nodes accept writes with conflict resolution
- **Quorum-based**: Majority vote ensures consensus before committing changes

**Example - Master-Slave Setup:**
```csharp
public class ReplicatedDataStore
{
    private readonly string _primaryNode;
    private readonly string[] _replicaNodes;
    
    public async Task WriteData(string key, string value)
    {
        // Write to primary node
        await _primaryNode.WriteAsync(key, value);
        
        // Asynchronously replicate to all replicas
        var replicationTasks = _replicaNodes.Select(replica =>
            replica.ReplicateAsync(key, value)
        );
        
        // Don't wait for replicas to complete (async replication)
        _ = Task.WhenAll(replicationTasks);
    }
    
    public async Task<string> ReadData(string key)
    {
        // Try replicas first for load distribution
        foreach (var replica in _replicaNodes.Shuffle())
        {
            try
            {
                return await replica.ReadAsync(key);
            }
            catch (TimeoutException) { }
        }
        
        // Fall back to primary
        return await _primaryNode.ReadAsync(key);
    }
}
```

### Component Redundancy

Critical system components are duplicated so that if one component fails, others can take over. This includes redundant servers, network paths, or services.

**Example - Load Balancer with Redundancy:**
```csharp
public class LoadBalancer
{
    private readonly List<ServiceInstance> _instances;
    private readonly IHealthCheck _healthCheck;
    
    public async Task<ServiceInstance> GetHealthyInstance()
    {
        var healthyInstances = await Task.WhenAll(
            _instances.Select(async inst => new
            {
                Instance = inst,
                IsHealthy = await _healthCheck.IsAliveAsync(inst)
            })
        );
        
        var available = healthyInstances
            .Where(x => x.IsHealthy)
            .Select(x => x.Instance)
            .ToList();
        
        if (available.Count == 0)
            throw new ServiceUnavailableException("No healthy instances available");
        
        // Return instance with lowest load
        return available.OrderBy(x => x.CurrentLoad).First();
    }
}
```

## Failover Mechanisms

### Active-Passive Failover

One component (active) handles the workload while another component (passive) remains on standby. If the active component fails, the passive component takes over.

**Characteristics:**
- Simpler setup and consistency management
- Resource underutilization (passive node idle)
- Faster recovery but slower failover detection
- Suitable for stateful services

**Example:**
```csharp
public class ActivePassiveFailover
{
    private ServiceNode _activeNode;
    private ServiceNode _passiveNode;
    private Timer _healthCheckTimer;
    
    public ActivePassiveFailover(ServiceNode active, ServiceNode passive)
    {
        _activeNode = active;
        _passiveNode = passive;
    }
    
    public void StartMonitoring()
    {
        _healthCheckTimer = new Timer(async _ => await CheckHealth(), null, 
            TimeSpan.Zero, TimeSpan.FromSeconds(5));
    }
    
    private async Task CheckHealth()
    {
        bool isHealthy = await _activeNode.PingAsync();
        
        if (!isHealthy)
        {
            await FailoverToPassive();
        }
    }
    
    private async Task FailoverToPassive()
    {
        // Promote passive to active
        _activeNode = _passiveNode;
        
        // Provision new passive node
        _passiveNode = await ServiceNode.CreateNewAsync();
        
        // Sync state to new passive
        await SyncState(_passiveNode);
    }
    
    private async Task SyncState(ServiceNode target)
    {
        var state = await _activeNode.GetStateAsync();
        await target.RestoreStateAsync(state);
    }
}
```

### Active-Active Failover

Multiple components actively handle workloads and share the load. If one component fails, others continue to handle the workload.

**Characteristics:**
- Better resource utilization
- Improved scalability and performance
- More complex consistency management
- No single point of failure

**Example:**
```csharp
public class ActiveActiveCluster
{
    private readonly List<ServiceNode> _nodes;
    private readonly IDistributedLock _lockService;
    
    public async Task<T> ExecuteWithConsistency<T>(
        Func<ServiceNode, Task<T>> operation)
    {
        var availableNodes = _nodes.Where(n => n.IsHealthy).ToList();
        
        if (availableNodes.Count == 0)
            throw new ServiceUnavailableException();
        
        // Use distributed lock for critical sections
        using (var @lock = await _lockService.AcquireAsync("critical-section", 
               TimeSpan.FromSeconds(10)))
        {
            // Execute operation on all nodes
            var tasks = availableNodes.Select(node => operation(node));
            var results = await Task.WhenAll(tasks);
            
            return results.First();
        }
    }
}
```

## Error Detection Techniques

### Heartbeat Mechanisms

Regular signals (heartbeats) are sent between components to detect failures. If a component stops sending heartbeats, it is considered failed.

**Implementation Patterns:**
- **Periodic heartbeats**: Fixed intervals (every 5 seconds)
- **Adaptive heartbeats**: Interval adjusts based on network conditions
- **Timeout escalation**: Increased timeout before declaring node failed

```csharp
public class HeartbeatMonitor
{
    private readonly Dictionary<string, DateTime> _lastHeartbeats;
    private readonly TimeSpan _heartbeatTimeout = TimeSpan.FromSeconds(10);
    private Timer _checkTimer;
    
    public HeartbeatMonitor()
    {
        _lastHeartbeats = new Dictionary<string, DateTime>();
        _checkTimer = new Timer(_ => CheckFailedNodes(), null, 
            TimeSpan.FromSeconds(1), TimeSpan.FromSeconds(1));
    }
    
    public void RecordHeartbeat(string nodeId)
    {
        _lastHeartbeats[nodeId] = DateTime.UtcNow;
    }
    
    private void CheckFailedNodes()
    {
        var now = DateTime.UtcNow;
        var failedNodes = _lastHeartbeats
            .Where(kvp => now - kvp.Value > _heartbeatTimeout)
            .Select(kvp => kvp.Key)
            .ToList();
        
        foreach (var nodeId in failedNodes)
        {
            OnNodeFailure(nodeId);
            _lastHeartbeats.Remove(nodeId);
        }
    }
    
    protected virtual void OnNodeFailure(string nodeId)
    {
        // Trigger failover or recovery procedures
    }
}
```

### Checkpointing

Periodic saving of the system's state so that if a failure occurs, the system can be restored to the last saved state.

**Checkpoint Strategies:**
- **Synchronous**: Block operations until checkpoint completes (consistent, slow)
- **Asynchronous**: Checkpoint in background while operations continue (faster, complex)
- **Incremental**: Only save changes since last checkpoint (efficient)

```csharp
public class CheckpointManager
{
    private readonly IPersistence _storage;
    private SystemState _lastCheckpoint;
    private DateTime _lastCheckpointTime;
    
    public async Task CreateCheckpoint(SystemState currentState)
    {
        // Quiesce writes temporarily
        using (var writeLock = await _writeLock.AcquireAsync())
        {
            // Create checkpoint atomically
            var checkpoint = new Checkpoint
            {
                State = currentState,
                Timestamp = DateTime.UtcNow,
                Version = _lastCheckpoint?.Version + 1 ?? 1
            };
            
            await _storage.SaveAsync(checkpoint);
            _lastCheckpoint = checkpoint.State;
            _lastCheckpointTime = checkpoint.Timestamp;
        }
    }
    
    public async Task<SystemState> RestoreFromCheckpoint()
    {
        var checkpoint = await _storage.GetLatestAsync();
        return checkpoint?.State ?? new SystemState();
    }
}
```

## Error Recovery Methods

### Rollback Recovery

The system reverts to a previous state after detecting an error, using saved checkpoints or logs. This restores consistency but loses work since the checkpoint.

**Use cases:** Database transactions, distributed sagas, state machines

```csharp
public class TransactionWithRollback
{
    public async Task<bool> ExecuteWithRollback(List<Action> operations)
    {
        var completedOperations = new Stack<Action>();
        
        try
        {
            foreach (var operation in operations)
            {
                operation();
                completedOperations.Push(operation);
            }
            return true;
        }
        catch (Exception ex)
        {
            // Rollback in reverse order
            while (completedOperations.Count > 0)
            {
                var rollbackOp = completedOperations.Pop();
                await RollbackOperationAsync(rollbackOp);
            }
            
            return false;
        }
    }
}
```

### Forward Recovery

The system attempts to correct or compensate for the failure to continue operating. This preserves partial progress but may result in inconsistent states.

**Use cases:** Compensating transactions, idempotent operations, event-driven systems

```csharp
public class SagaWithCompensation
{
    public async Task ExecuteSaga(SagaContext context)
    {
        try
        {
            await StepOne(context);
            await StepTwo(context);
            await StepThree(context);
        }
        catch (Exception ex)
        {
            // Compensate instead of rollback
            if (context.StepOneCompleted)
                await CompensateStepOne(context);
            if (context.StepTwoCompleted)
                await CompensateStepTwo(context);
            
            throw;
        }
    }
    
    private async Task CompensateStepOne(SagaContext context)
    {
        // Undo effects of step one (reverse transfer, etc.)
        await _compensationService.UndoStepOneAsync(context);
    }
}
```

## Monitoring and Alerting

Proactive monitoring enables early detection of issues before they become critical:

- **Availability**: System uptime and component health
- **Latency**: Request response times and degradation patterns
- **Error rates**: Frequency and types of failures
- **Resource utilization**: CPU, memory, disk usage trends
- **Recovery times**: Time to detect and recover from failures

Set aggressive alert thresholds to catch issues during degradation, not after complete failure.
