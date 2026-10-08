---
title: Scaling Out Applications
description: Strategies and patterns for scaling distributed applications
pubDate: 'Oct 07 2024'
heroImage: '../../assets/hero-scaling.svg'
---

## Strategies for Scaling Out an Application

As applications grow, single-instance deployments become bottlenecks. Scaling out (horizontal scaling) distributes load across multiple instances. This requires resilience patterns to handle partial failures, degradation, and recovery in distributed environments.

## Circuit Breaker Pattern

This pattern prevents a system from making calls to a failing service by wrapping it in a 'circuit breaker'. When the service fails, the circuit breaker trips, causing further calls to fail fast instead of trying to connect to a failing service repeatedly.

**States:**
- **Closed**: Normal operation, calls pass through
- **Open**: Service failed, calls are rejected immediately (fail fast)
- **Half-Open**: Testing if service recovered, limited calls allowed

**Benefits:**
- Prevents cascading failures
- Reduces load on failing services
- Enables faster recovery through fail-fast approach
- Improves user experience (quick response vs. timeout)

**Example:**
```csharp
public class CircuitBreaker
{
    private CircuitState _state = CircuitState.Closed;
    private int _failureCount = 0;
    private DateTime _lastFailureTime;
    private readonly int _failureThreshold = 5;
    private readonly TimeSpan _timeout = TimeSpan.FromSeconds(30);
    
    public async Task<T> ExecuteAsync<T>(Func<Task<T>> operation)
    {
        if (_state == CircuitState.Open)
        {
            // Check if timeout has passed to enter half-open state
            if (DateTime.UtcNow - _lastFailureTime > _timeout)
                _state = CircuitState.HalfOpen;
            else
                throw new CircuitBreakerOpenException("Service is temporarily unavailable");
        }
        
        try
        {
            var result = await operation();
            OnSuccess();
            return result;
        }
        catch (Exception ex)
        {
            OnFailure();
            throw;
        }
    }
    
    private void OnSuccess()
    {
        _failureCount = 0;
        _state = CircuitState.Closed;
    }
    
    private void OnFailure()
    {
        _failureCount++;
        _lastFailureTime = DateTime.UtcNow;
        
        if (_failureCount >= _failureThreshold)
            _state = CircuitState.Open;
    }
}

public enum CircuitState { Closed, Open, HalfOpen }
```

**When to use:**
- Calling external APIs or services
- Database operations prone to transient failures
- Operations with known failure patterns

## Bulkhead Pattern

This pattern isolates different components or services to prevent a failure in one part of the system from affecting others. It's similar to the bulkheads in a ship that prevent flooding in one compartment from sinking the entire vessel.

**Isolation Strategies:**
- **Thread pools**: Separate thread pools per service (limited resources per tenant)
- **Processes**: Separate processes for critical services
- **Resources**: Dedicated CPU, memory, disk per service
- **Network**: Separate networks or VLANs per service tier

**Benefits:**
- Prevents resource exhaustion from cascading
- Enables independent scaling of services
- Improves system stability under load
- Simplifies troubleshooting and monitoring

**Example:**
```csharp
public class BulkheadIsolation
{
    private readonly SemaphoreSlim _bulkhead;
    private readonly int _maxConcurrentCalls;
    
    public BulkheadIsolation(int maxConcurrentCalls = 10)
    {
        _maxConcurrentCalls = maxConcurrentCalls;
        _bulkhead = new SemaphoreSlim(maxConcurrentCalls, maxConcurrentCalls);
    }
    
    public async Task<T> ExecuteAsync<T>(Func<Task<T>> operation)
    {
        if (!await _bulkhead.WaitAsync(TimeSpan.FromSeconds(5)))
            throw new BulkheadRejectedException(
                $"Operation rejected: max concurrent calls ({_maxConcurrentCalls}) reached");
        
        try
        {
            return await operation();
        }
        finally
        {
            _bulkhead.Release();
        }
    }
}

// Usage with separate bulkheads per service
public class ServiceClients
{
    private readonly BulkheadIsolation _userServiceBulkhead = new(10);
    private readonly BulkheadIsolation _orderServiceBulkhead = new(15);
    private readonly BulkheadIsolation _paymentServiceBulkhead = new(5); // Critical service
    
    public async Task<User> GetUserAsync(int userId)
    {
        return await _userServiceBulkhead.ExecuteAsync(
            () => _userService.GetAsync(userId));
    }
    
    public async Task<Order> GetOrderAsync(int orderId)
    {
        return await _orderServiceBulkhead.ExecuteAsync(
            () => _orderService.GetAsync(orderId));
    }
    
    public async Task<PaymentResult> ProcessPaymentAsync(Payment payment)
    {
        return await _paymentServiceBulkhead.ExecuteAsync(
            () => _paymentService.ProcessAsync(payment));
    }
}
```

**When to use:**
- Multi-tenant systems
- Microservices with interdependencies
- Services with different criticality levels
- Systems with variable load patterns

## Retry Pattern

This pattern involves automatically retrying an operation that has failed due to transient errors. Retries must be carefully designed with exponential backoff to avoid overwhelming the system.

**Retry Strategies:**
- **Immediate**: Retry immediately (rarely useful)
- **Linear backoff**: Retry after n seconds, then n+x seconds, etc.
- **Exponential backoff**: Retry delays double with each attempt (1s, 2s, 4s, 8s...)
- **Exponential backoff with jitter**: Random delay added to prevent thundering herd

**Benefits:**
- Handles transient failures transparently
- Reduces need for manual intervention
- Improves success rates for network operations

**Risks:**
- Can amplify load during outages if not careful
- Retry storms without proper backoff
- Operations must be idempotent

**Example:**
```csharp
public class RetryPolicy
{
    private readonly int _maxRetries;
    private readonly TimeSpan _initialDelay;
    private readonly Random _random = new();
    
    public RetryPolicy(int maxRetries = 3, TimeSpan? initialDelay = null)
    {
        _maxRetries = maxRetries;
        _initialDelay = initialDelay ?? TimeSpan.FromMilliseconds(100);
    }
    
    public async Task<T> ExecuteAsync<T>(
        Func<Task<T>> operation,
        Func<Exception, bool> isTransient = null)
    {
        isTransient ??= ex => ex is TimeoutException or HttpRequestException;
        
        for (int attempt = 1; attempt <= _maxRetries; attempt++)
        {
            try
            {
                return await operation();
            }
            catch (Exception ex) when (isTransient(ex) && attempt < _maxRetries)
            {
                var delay = CalculateBackoff(attempt);
                await Task.Delay(delay);
            }
        }
        
        // Final attempt without catch
        return await operation();
    }
    
    private TimeSpan CalculateBackoff(int attempt)
    {
        // Exponential backoff: 100ms, 200ms, 400ms
        var baseDelay = _initialDelay.TotalMilliseconds * Math.Pow(2, attempt - 1);
        
        // Add jitter: ±25% of delay
        var jitter = (baseDelay * 0.5) * (_random.NextDouble() - 0.5);
        
        return TimeSpan.FromMilliseconds(baseDelay + jitter);
    }
}

// Usage
var policy = new RetryPolicy(maxRetries: 3);
var user = await policy.ExecuteAsync(() => _userService.GetAsync(userId));
```

**When to use:**
- Network calls to remote services
- Transient database connection issues
- API calls with temporary rate limiting
- Not for permanent failures (auth errors, 404s)

## Rate Limiting Pattern

This pattern controls the number of requests a system or service can handle within a specific time window to prevent overload and ensure fair usage.

**Algorithms:**
- **Token Bucket**: Tokens accumulate at fixed rate, consumed per request
- **Sliding Window**: Track all requests in time window
- **Leaky Bucket**: Fixed rate of processing regardless of input rate
- **Fixed Window**: Simple counter per time period

**Benefits:**
- Prevents system overload
- Ensures fair resource allocation
- Protects against abuse and DoS attacks
- Enables predictable performance

**Example:**
```csharp
public class TokenBucketRateLimiter
{
    private double _tokens;
    private readonly double _capacity;
    private readonly double _tokensPerSecond;
    private DateTime _lastRefillTime;
    private readonly object _lock = new();
    
    public TokenBucketRateLimiter(int capacity, double tokensPerSecond)
    {
        _capacity = capacity;
        _tokensPerSecond = tokensPerSecond;
        _tokens = capacity;
        _lastRefillTime = DateTime.UtcNow;
    }
    
    public bool AllowRequest(int tokensRequired = 1)
    {
        lock (_lock)
        {
            RefillTokens();
            
            if (_tokens >= tokensRequired)
            {
                _tokens -= tokensRequired;
                return true;
            }
            
            return false;
        }
    }
    
    private void RefillTokens()
    {
        var now = DateTime.UtcNow;
        var timePassed = (now - _lastRefillTime).TotalSeconds;
        
        var tokensToAdd = timePassed * _tokensPerSecond;
        _tokens = Math.Min(_capacity, _tokens + tokensToAdd);
        
        _lastRefillTime = now;
    }
}

// Per-user rate limiting
public class RateLimitedService
{
    private readonly Dictionary<int, TokenBucketRateLimiter> _userLimiters;
    
    public async Task<Response> ProcessAsync(int userId, Request request)
    {
        var limiter = _userLimiters.GetOrAdd(userId, 
            _ => new TokenBucketRateLimiter(capacity: 100, tokensPerSecond: 10));
        
        if (!limiter.AllowRequest())
            throw new RateLimitedException("Too many requests. Try again later.");
        
        return await _service.ProcessAsync(request);
    }
}
```

**When to use:**
- Public APIs
- Protecting backend services
- Multi-tenant systems
- Preventing resource exhaustion

## Failover Pattern

This pattern involves switching to a backup system or component when the primary one fails. Failover works in conjunction with health checking and service discovery.

**Strategies:**
- **Automatic failover**: System detects failure and switches automatically
- **Graceful degradation**: Reduce functionality rather than complete failure
- **Fallback**: Use cached or default values when primary unavailable

**Example:**
```csharp
public class FailoverClient
{
    private readonly List<ServiceEndpoint> _endpoints;
    private ServiceEndpoint _primaryEndpoint;
    private int _currentIndex = 0;
    
    public async Task<T> ExecuteWithFailoverAsync<T>(
        Func<ServiceEndpoint, Task<T>> operation)
    {
        var maxAttempts = _endpoints.Count;
        
        for (int attempt = 0; attempt < maxAttempts; attempt++)
        {
            try
            {
                return await operation(_endpoints[_currentIndex]);
            }
            catch (ServiceUnavailableException)
            {
                _currentIndex = (_currentIndex + 1) % _endpoints.Count;
            }
        }
        
        throw new ServiceUnavailableException("All endpoints failed");
    }
}
```

**When to use:**
- Critical services with strict availability requirements
- Database replicas and read-write splitting
- Load balancing across multiple datacenters

## Combining Patterns

Effective scaling requires combining multiple patterns:

```csharp
public class ResilientServiceClient
{
    private readonly CircuitBreaker _circuitBreaker = new();
    private readonly RetryPolicy _retryPolicy = new(maxRetries: 3);
    private readonly BulkheadIsolation _bulkhead = new(maxConcurrentCalls: 10);
    private readonly TokenBucketRateLimiter _rateLimiter = 
        new(capacity: 100, tokensPerSecond: 10);
    
    public async Task<T> CallServiceAsync<T>(Func<Task<T>> operation)
    {
        // Check rate limit first
        if (!_rateLimiter.AllowRequest())
            throw new RateLimitedException();
        
        // Isolate with bulkhead
        return await _bulkhead.ExecuteAsync(async () =>
        {
            // Retry with exponential backoff
            return await _retryPolicy.ExecuteAsync(async () =>
            {
                // Circuit breaker prevents cascading failures
                return await _circuitBreaker.ExecuteAsync(operation);
            });
        });
    }
}
```

## Best Practices

1. **Start with the right architecture**: Stateless services scale better than stateful ones
2. **Monitor key metrics**: Latency, error rates, resource utilization
3. **Test failure scenarios**: Chaos engineering, failure injection
4. **Use timeouts**: Always set reasonable timeouts on external calls
5. **Implement observability**: Distributed tracing, centralized logging
6. **Progressive rollout**: Deploy with canary deployments and feature flags
7. **Scale databases carefully**: Caching and read replicas before sharding
