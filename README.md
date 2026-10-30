# fk-links

A service that shortens links, written in [fckt](https://github.com/m4xy/fckt): JSON over HTTP, kept
in SQLite. It's an example of a whole service in fckt, made of the ecosystem's libraries:
[fk-http](../fk-http) serves it, [fk-sqlite](../fk-sqlite) keeps its links,
[fk-serialization](../fk-serialization) writes its JSON, [fk-regex](../fk-regex) checks what it's
given, and [fk-cli](../fk-cli) reads its command line.

```
$ kiln build
$ build/bin/fk-links --port 8080 --database links.db
$ curl -XPOST localhost:8080/links -d '{"url": "https://ocaml.org", "slug": "ocaml"}'
{"slug":"ocaml","url":"https://ocaml.org","visits":0}
$ curl -i localhost:8080/ocaml
HTTP/1.1 302 Found
Location: https://ocaml.org
```

| | |
|---|---|
| `POST /links` `{"url": ..., "slug": ...}` | makes a link, named as it asks or in base 62 of its row: 201; 409 if the name's taken; 422 if it may not be made |
| `GET /links` | every link, newest first |
| `GET /links/{slug}` | the link, with how many times it's been followed |
| `DELETE /links/{slug}` | deletes it: 204 |
| `GET /{slug}` | 302 to where it leads, counting the visit |
| `GET /health` | 200, if it's up |

## How it's made

Four layers, each a package, each using only those before it:

| Package | What's in it |
|---|---|
| `uk.m4xy.links.domain` | What links are, and their rules, pure: `Slug` and `TargetUrl`, which only `Slugs` and `TargetUrls` can make, so each is always valid; `Link`; `LinkProblem`, everything that can go wrong; and `LinkRepository`, the effect that says what the domain needs of a store, without saying how. |
| `uk.m4xy.links.application` | `LinkService`, the use cases: shorten, follow, show, list, delete. Each takes what it's asked as it's asked, checks it by the domain's rules, and is `@LinkRepository`: it keeps links, and does nothing else. |
| `uk.m4xy.links.infrastructure` | Handlers of `LinkRepository`: `SqliteLinks`, through SQL, which is `@Database`; and `InMemoryLinks`, in a list. |
| `uk.m4xy.links.presentation` | The service over HTTP: `LinkRoutes`, a table of fk-http endpoints, each its method, its path, what it reads from a call, typed, and the handler it's given to, then the handlers; the JSON it reads and writes (`ShortenRequest`, `LinkView`, ...), apart from the domain's types; `Problems`, each `LinkProblem` as a status and a message. |

`main.fk` joins them: it opens the database, keeps the links in it with `SqliteLinks.keeping`, and
serves `LinkRoutes.routes`. That's all that touches the network or a file.

Each layer is tested on its own: `test/domain.fk` the rules, with nothing to handle;
`test/application.fk` the use cases, with `InMemoryLinks`; and `test/service.fk` the whole service,
every route, over HTTP as a client would ask, but in memory, with `Server.answering` in place of
sockets and SQLite's `":memory:"` in place of the file.
