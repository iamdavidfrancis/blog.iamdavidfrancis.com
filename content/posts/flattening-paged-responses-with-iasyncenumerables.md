---
title: "Flattening Paged Responses with IAsyncEnumerables"
date: 2020-12-07T23:15:00-8:00
publish-date: 2020-12-08T00:05:00-8:00
draft: false
categories: ["development"]
tags: ["dotnet", "c#"]
---

I've recently found myself in a situation where I'm calling APIs that return a collection of items in pages. Especially when calling OData APIs (looking at you Microsoft Graph). Usually these come in the form of a response model that wraps the array of items and a continuation token, or a next link. Something like this:

```json
{
    "values": [...],
    "nextLink": "someUrl"
}
```

we can represent that easily with a Generic class:

```csharp
public class PagedResponse<TResult> where TResult : class
{
    public IList<TResult> Values { get; set; }

    public string NextLink { get; set; }
}
```

When I'm building APIs that need to iterate across the entire list, I have to do some logic to loop over the values, check if there's a nextLink and then repeat the process again. Now, I'm not a huge fan of repeating code everywhere, so I think it would be nice if I could move that logic somewhere nice and get the entire list. I *could* build a method that looks like this:

```csharp
Task<IEnumerable<TResult>> GetEntireList<TResult>(Func<string, Task<PagedResponse<TResult>>> getPage);
```

The downside to doing this is that I need to download the entire collection into memory before I can start iterating. This is terrible if I only end up wanting to work with a subset of the items. Especially if the full collection could be arbitrarily large (trying to load a few GB into memory could be problematic). What would be nicer is if I could return an `IEnumerable` that was also awaitable.

This is the exact scenario where `IAsyncEnumerable` saves the day. Introduced in C# 8, `IAsyncEnumerable` is both an `IEnumerable` and awaitable. Now I can make a method like:

```csharp
IAsyncEnumerable<TResult> GetEntireList<TResult>(Func<string, Task<PagedResponse<TResult>>> getPage);
```

Now all we have to do is build a loop to fetch a page, return the elements from that page, then loop back around and do it again. The implementation for this is actually super clean:

```csharp
public static async IAsyncEnumerable<TResult> GetEntireList<TResult>(Func<string, Task<PagedResponse<TResult>>> getPage) {
    string nextLink = null;

    do {
        var currentPage = await getPage(nextLink);

        // Stop if the page is missing
        if (currentPage == null) {
            yield break;
        }

        nextLink = currentPage.NextLink;

        foreach (var item in currentPage.Values) {
            yield return item;
        }
    } while (!string.IsNullOrEmpty(nextLink));
}
```

Let's go over what all of that is doing. 

```csharp
public static async IAsyncEnumerable<TResult> GetEntireList<TResult>(Func<string, Task<PagedResponse<TResult>>> getPage) {
```

For the purposes of this example, the `getPage` delegate will be a function that takes in a nextLink and returns an awaitable Task containing the next `PagedResponse`. This allows the caller of this method to used whatever source they want to fetch the pages, as long as that source returns a `PagedResponse`. We'll also assume that a `null` value passed in means we want the first page. We could also define the delegate to take in an `int` and then tell it which page number we want to use next. This is nice for situations where instead of getting a Next Link, we get a flag indicating more pages.

```csharp
    string nextLink = null;

    do
    {
        // ...
    } while (!string.IsNullOrEmpty(nextLink));
```

Next up, we're defining a variable we'll use to build our loop. Because we're using a `nextLink` to determine if we have pages, we need to keep track of the current `nextLink`. We'll use a `do-while` loop to ensure the loop always runs at least once. After the first run, we want to loop until there isn't a `nextLink`. 

```csharp
        var currentPage = await getPage(nextLink);

        // Stop if the page is missing
        if (currentPage == null) {
            yield break;
        }
```

Inside the loop, the first thing we're going to do is load the next page from the `getPage` delegate. If the delegate returns `null`, we'll treat that as the end of the pages and call `yield break` to indicate we're done returning elements. Otherwise we'll keep going.

```csharp
        nextLink = currentPage.NextLink;

        foreach (var item in currentPage.Values) {
            yield return item;
        }
``` 

Now that we've loaded the page, we'll store the current page's `NextLink` in our variable. Then we will iterate over all the elements in the `Values` collection. We'll use a `foreach` to iterate over every item and `yield return` to return that item. 

That's it, we're done with the implementation. Obviously this is a very light implementation and lacks things like error handling and a bunch of edge cases or other handy features, but it shows the power of `IAsyncEnumerable`. Now let's see how we could consume it:

We'll use the `System.Linq.Async` package from [dotnet/reactive](https://github.com/dotnet/reactive) to give us the traditional Linq expressions that we're used to:

```csharp
using System.Linq.Async;
```

Now we can do some really powerful things:

```csharp
var results = GetEntireList((nextLink) => dataService.GetItemsPage<Member>(nextLink))
                .Where(member => member.IsPending)
                .Select(member => new { member.Id, member.Name });

await foreach (var member in results) {
    // Some business logic on the ID and Names
}
```

As you can see, we can use the standard `Where` and `Select` statements (brought in from `System.Linq.Async`) to process our `IAsyncEnumerable` just like an `IEnumerable`. 

Hopefully you found this post informative and as always if you see any errors or issues, please reach out to me on [twitter](https://twitter.com/iamdavidfrancis). 

## References

* https://docs.microsoft.com/en-us/archive/msdn-magazine/2019/november/csharp-iterating-with-async-enumerables-in-csharp-8
* https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1?view=dotnet-plat-ext-5.0
