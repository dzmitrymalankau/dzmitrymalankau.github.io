---
title: Code Snippets
description: Useful code examples and utilities
pubDate: 'Oct 04 2024'
heroImage: '../../assets/hero-code-snippets.svg'
---

---
title: Code Snippets
description: Useful code examples and utilities
pubDate: 'Oct 04 2024'
heroImage: '../../assets/hero-code-snippets.svg'
---

# Code Snippets

A collection of useful code examples and utilities for common programming tasks.

## Flatten Nested Dictionaries (C#)

Convert nested dictionaries into a flat structure with compressed keys. Perfect for working with configuration hierarchies or nested JSON structures.

```csharp
Dictionary<string, string> Convert(Dictionary<string, object> input)
{
    var result = new Dictionary<string, string>();

    foreach (var key in input.Keys)
    {
        if (input[key] is Dictionary<string, string> last)
        {
            var endElement = last.FirstOrDefault();
            result.Add($"{key}.{endElement.Key}", endElement.Value);
            continue;
        }

        var current = (input[key] as Dictionary<string, object>)!;

        foreach (var next in Convert(current))
        {
            result.Add($"{key}.{next.Key}", next.Value);
        }
    } 
    
    return result;
}
```

See the [full article](/blog/flatten-nested-dictionaries/) for more details and sample data.

## Retry Logic with Exponential Backoff (C#)

Implement resilient retry logic for transient failures using exponential backoff strategy. Essential for handling transient network errors and temporary service unavailability.

```csharp
public async Task<T> RetryWithBackoff<T>(
    Func<Task<T>> operation,
    int maxRetries = 3,
    int initialDelayMs = 100)
{
    int delay = initialDelayMs;
    
    for (int attempt = 1; attempt <= maxRetries; attempt++)
    {
        try
        {
            return await operation();
        }
        catch (Exception ex) when (attempt < maxRetries)
        {
            await Task.Delay(delay);
            delay *= 2; // Exponential backoff
        }
    }
    
    return await operation(); // Final attempt without catch
}
```

## Async Stream Processing (C#)

Process items from an async enumerable with concurrency limits. Useful for batch processing, parallel downloads, or rate-limited API calls.

```csharp
public async Task ProcessAsyncStream<T>(
    IAsyncEnumerable<T> items,
    Func<T, Task> processor,
    int maxConcurrency = 5)
{
    var semaphore = new SemaphoreSlim(maxConcurrency);
    var tasks = new List<Task>();

    await foreach (var item in items)
    {
        await semaphore.WaitAsync();
        
        tasks.Add(Task.Run(async () =>
        {
            try
            {
                await processor(item);
            }
            finally
            {
                semaphore.Release();
            }
        }));
    }

    await Task.WhenAll(tasks);
}
```

## Generic Cache with TTL (C#)

Simple in-memory cache with automatic expiration. Useful for caching frequently accessed data with time-based invalidation.

```csharp
public class CacheEntry<T>
{
    public T Value { get; set; }
    public DateTime ExpirationTime { get; set; }
    
    public bool IsExpired => DateTime.UtcNow > ExpirationTime;
}

public class MemoryCache<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, CacheEntry<TValue>> _cache = new();
    private readonly ReaderWriterLockSlim _lock = new();
    
    public bool TryGet(TKey key, out TValue value)
    {
        _lock.EnterReadLock();
        try
        {
            if (_cache.TryGetValue(key, out var entry) && !entry.IsExpired)
            {
                value = entry.Value;
                return true;
            }
            
            value = default;
            return false;
        }
        finally
        {
            _lock.ExitReadLock();
        }
    }
    
    public void Set(TKey key, TValue value, TimeSpan ttl)
    {
        _lock.EnterWriteLock();
        try
        {
            _cache[key] = new CacheEntry<TValue>
            {
                Value = value,
                ExpirationTime = DateTime.UtcNow.Add(ttl)
            };
        }
        finally
        {
            _lock.ExitWriteLock();
        }
    }
    
    public void Remove(TKey key)
    {
        _lock.EnterWriteLock();
        try
        {
            _cache.Remove(key);
        }
        finally
        {
            _lock.ExitWriteLock();
        }
    }
    
    public void Clear()
    {
        _lock.EnterWriteLock();
        try
        {
            _cache.Clear();
        }
        finally
        {
            _lock.ExitWriteLock();
        }
    }
}

// Usage
var cache = new MemoryCache<string, User>();
cache.Set("user:123", user, TimeSpan.FromMinutes(5));

if (cache.TryGet("user:123", out var cachedUser))
{
    Console.WriteLine($"Found cached user: {cachedUser.Name}");
}
```

## Batch Processing Helper (C#)

Process large collections in batches with progress tracking. Ideal for database bulk operations or batch API submissions.

```csharp
public static class BatchProcessor
{
    public static async Task<List<TResult>> ProcessInBatches<TItem, TResult>(
        IEnumerable<TItem> items,
        int batchSize,
        Func<List<TItem>, Task<List<TResult>>> batchProcessor,
        IProgress<(int processedCount, int totalCount)> progress = null)
    {
        var allItems = items.ToList();
        var results = new List<TResult>();
        
        for (int i = 0; i < allItems.Count; i += batchSize)
        {
            var batch = allItems.Skip(i).Take(batchSize).ToList();
            var batchResults = await batchProcessor(batch);
            results.AddRange(batchResults);
            
            progress?.Report((i + batch.Count, allItems.Count));
        }
        
        return results;
    }
}

// Usage
var users = Enumerable.Range(1, 10000).Select(i => new User { Id = i }).ToList();
var progress = new Progress<(int, int)>(p => 
    Console.WriteLine($"Processed {p.Item1}/{p.Item2}"));

var results = await BatchProcessor.ProcessInBatches(
    users,
    batchSize: 100,
    async batch => await _database.BulkInsertAsync(batch),
    progress);
```

## Retry with Jitter (C#)

Prevent thundering herd problem by adding randomization to retry delays.

```csharp
public async Task<T> RetryWithJitter<T>(
    Func<Task<T>> operation,
    int maxRetries = 3,
    int baseDelayMs = 100)
{
    var random = new Random();
    
    for (int attempt = 1; attempt <= maxRetries; attempt++)
    {
        try
        {
            return await operation();
        }
        catch (Exception ex) when (attempt < maxRetries)
        {
            // Exponential backoff with jitter
            var exponentialDelay = baseDelayMs * Math.Pow(2, attempt - 1);
            var jitter = random.Next(0, (int)exponentialDelay);
            var totalDelay = exponentialDelay + jitter;
            
            await Task.Delay((int)totalDelay);
        }
    }
    
    return await operation(); // Final attempt
}
```

## Concurrent Execution with Timeout (C#)

Execute operations concurrently with a global timeout to prevent indefinite hangs.

```csharp
public async Task<List<T>> ExecuteConcurrentWithTimeout<T>(
    List<Func<Task<T>>> operations,
    TimeSpan timeout)
{
    using var cts = new CancellationTokenSource(timeout);
    var tasks = operations.Select(op => ExecuteWithCancellation(op, cts.Token));
    
    try
    {
        return (await Task.WhenAll(tasks)).ToList();
    }
    catch (OperationCanceledException)
    {
        throw new TimeoutException($"Operations exceeded timeout of {timeout.TotalSeconds}s");
    }
}

private async Task<T> ExecuteWithCancellation<T>(
    Func<Task<T>> operation,
    CancellationToken cancellationToken)
{
    try
    {
        return await operation();
    }
    catch (OperationCanceledException)
    {
        // Return default or throw for specific handling
        throw;
    }
}
```

## Polly Resilience Patterns (C#)

Using the Polly library for advanced resilience patterns (install: `dotnet add package Polly`):

```csharp
// Combine multiple resilience strategies
var policy = Policy
    .Handle<HttpRequestException>()
    .Or<TimeoutException>()
    .Retry(3)
    .Wrap(
        Policy
            .Handle<HttpRequestException>()
            .CircuitBreaker(3, TimeSpan.FromSeconds(30))
    )
    .Wrap(
        Policy
            .Handle<HttpRequestException>()
            .Timeout(TimeSpan.FromSeconds(5))
    );

// Usage
var result = await policy.ExecuteAsync(() => 
    _httpClient.GetAsync("https://api.example.com/data"));
```

## Useful Extensions (C#)

Helper extension methods for common operations:

```csharp
public static class CollectionExtensions
{
    public static IEnumerable<IEnumerable<T>> Batch<T>(
        this IEnumerable<T> source,
        int batchSize)
    {
        var batch = new List<T>();
        foreach (var item in source)
        {
            batch.Add(item);
            if (batch.Count == batchSize)
            {
                yield return batch;
                batch = new List<T>();
            }
        }
        
        if (batch.Count > 0)
            yield return batch;
    }
    
    public static bool TryRemove<T>(this List<T> list, Predicate<T> predicate, out T removed)
    {
        var index = list.FindIndex(predicate);
        if (index >= 0)
        {
            removed = list[index];
            list.RemoveAt(index);
            return true;
        }
        
        removed = default;
        return false;
    }
}

public static class DictionaryExtensions
{
    public static TValue GetOrAdd<TKey, TValue>(
        this Dictionary<TKey, TValue> dict,
        TKey key,
        Func<TKey, TValue> factory) where TKey : notnull
    {
        if (!dict.TryGetValue(key, out var value))
        {
            value = factory(key);
            dict[key] = value;
        }
        
        return value;
    }
}

// Usage
var batches = items.Batch(100);
var result = dict.GetOrAdd("key", k => ExpensiveComputation());
```
