TIL about [gron](https://github.com/tomnomnom/gron), a utility that transforms JSON into discrete
assignments that allow one to use familiar text utils (`grep`, `sed`, `awd`, etc.) to understand
the content of the JSON.

The documentation provides the following example.

```
▶ gron "https://api.github.com/repos/tomnomnom/gron/commits?per_page=1" | fgrep "commit.author"
json[0].commit.author = {};
json[0].commit.author.date = "2016-07-02T10:51:21Z";
json[0].commit.author.email = "mail@tomnomnom.com";
json[0].commit.author.name = "Tom Hudson";
```

I'm off to inspect response payloads.
