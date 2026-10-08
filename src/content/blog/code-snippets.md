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

Implement resilient retry logic for transient failures using exponential backoff strategy.

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

Process items from an async enumerable with concurrency limits.

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
