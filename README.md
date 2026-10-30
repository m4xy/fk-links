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

Each part says what it does in its type:

- `Shortening` is pure: what a link may be, and the names made for links. It touches nothing.
- `LinkStore`'s functions are `@Database`: they run SQL, and nothing else.
- `Api.routes` is a `List<Route<@Database>>`: routes that use the database, and do nothing else
  besides answering.
- `main` opens the database and serves the routes, which is all that touches the network or a file.

So the tests in `test/service.fk` run the whole service, every route, over HTTP as a client would
ask, but in memory, with `Server.answering` in place of sockets and `":memory:"` in place of the
file: changing nothing in the service.

It needs the OCaml package `sqlite3` (`opam install sqlite3`), and fckt's branch `ranges`, which has
ranges and, under it, the `printError` fk-cli uses.
