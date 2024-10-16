## Flatten nested dictionaries, compressing keys

Below is the recursive method that uppacks the nested list of dictionaries. Also [here](https://gist.github.com/dzmitrymalankau/a88be59644b74f7cb76743f0406393b3) is the gist with sample data.

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
}

```
