---
title: Flatten Nested Dictionaries
description: C# code to flatten nested dictionaries by compressing keys
pubDate: 'Oct 06 2024'
heroImage: '../../assets/hero-flatten-dict.svg'
---

## Flatten Nested Dictionaries, Compressing Keys

This technique is useful when working with configuration hierarchies, JSON structures, or any deeply nested key-value data that needs to be flattened into a single level with dot-notation keys.

### The Problem

You have nested dictionaries like:
```csharp
var nested = new Dictionary<string, object>
{
    { "database", new Dictionary<string, object>
        {
            { "connection", new Dictionary<string, string>
                {
                    { "host", "localhost" },
                    { "port", "5432" }
                }
            }
        }
    }
};
```

And you want to flatten it to:
```
database.connection.host = localhost
database.connection.port = 5432
```

### The Solution

Below is a recursive method that unpacks nested dictionaries and creates compressed dot-notation keys:

```csharp
public static Dictionary<string, string> Flatten(
    Dictionary<string, object> input,
    string prefix = "")
{
    var result = new Dictionary<string, string>();

    foreach (var key in input.Keys)
    {
        var fullKey = string.IsNullOrEmpty(prefix) 
            ? key 
            : $"{prefix}.{key}";

        if (input[key] is Dictionary<string, string> stringDict)
        {
            // Leaf node - add to result
            foreach (var entry in stringDict)
            {
                result.Add($"{fullKey}.{entry.Key}", entry.Value);
            }
        }
        else if (input[key] is Dictionary<string, object> nestedDict)
        {
            // Recursively flatten nested dictionaries
            var flattened = Flatten(nestedDict, fullKey);
            foreach (var entry in flattened)
            {
                result.Add(entry.Key, entry.Value);
            }
        }
        else if (input[key] is string stringValue)
        {
            result.Add(fullKey, stringValue);
        }
    }

    return result;
}
```

### Usage Example

```csharp
var flattened = Flatten(nestedDictionary);

foreach (var kvp in flattened)
{
    Console.WriteLine($"{kvp.Key} = {kvp.Value}");
}
```

### Real-World Applications

- **Configuration Management**: Flatten hierarchical app settings
- **Environment Variables**: Convert nested configs to env var format
- **Database Schemas**: Flatten JSON columns for query indexing
- **API Payloads**: Normalize nested response structures

For sample data and a complete working example, see [this gist](https://gist.github.com/dzmitrymalankau/a88be59644b74f7cb76743f0406393b3).
