# Sourcegraph cookbook

Fancy tips and tricks from the DevX team when using Sourcegraph!

## Finding _when_ a file got deleted

Let's say you find a dead link in the handbook and you can't find where the new content is. What you can do is to search using the `diff` type in conjunction with `file` to find the commit that deleted it.

```sourcegraph
repo:^github\.com/sourcegraph/handbook$ file:^content/departments/product-engineering/process/planning-process\.md type:diff
```

To ensure this example stays correct, the query here includes the full path toward the file, but using simply `file:planning-process.md` would have worked as well.

Here we can see that the first result, mentions a commit from _Quinn Slack_ that deleted the file we're looking for. Inspecting that commit, we can see that the file has been fully deleted and there isn't a direct equivalent for our original dead link.

## Finding with which go version a given module is being used 

Let's say we're bumping the minimum version of a module, because now we're using generics. We probably want to see what go version the modules importing our library are using in the wild:

```sourcegraph
context:@sourcegraph file:go.mod go 1.:[~\d+] :[_] require (:[_]github.com/sourcegraph/log :[_]) patternType:structural
```
