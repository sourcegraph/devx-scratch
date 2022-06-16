# Queries Examples

## Finding _when_ a file got deleted

Let's say you find a dead link in the handbook and you can't find where the new content is. What you can do is to search using the `diff` type in conjunction with `file` to find the commit that deleted it.

```sourcegraph
repo:^github\.com/sourcegraph/handbook$ file:^content/departments/product-engineering/process/planning-process\.md type:diff
```

To ensure this example stays correct, the query here includes the full path toward the file, but using simply `file:planning-process.md` would have worked as well.

Here we can see that the first result, mentions a commit from _Quinn Slack_ that deleted the file we're looking for. Inspecting that commit, we can see that the file has been fully deleted and there isn't a direct equivalent for our original dead link.
