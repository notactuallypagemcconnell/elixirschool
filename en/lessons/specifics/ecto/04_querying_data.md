---
version: 1.0.0
title: Querying Data
---

## Querying

- set 










Before we can query our repository we need to import the Query API.  For now we only need to import `from/2`:

```elixir
import Ecto.Query, only: [from: 2]
```

The official documentation can be found at [Ecto.Query](http://hexdocs.pm/ecto/Ecto.Query.html).

### Basics

Ecto provides an excellent Query DSL that allows us to express queries clearly.  To find the usernames of all confirmed accounts we could use something like this:

```elixir
alias ExampleApp.{Repo, User}

query =
  from(
    u in User,
    where: u.confirmed == true,
    select: u.username
  )

Repo.all(query)
```

In addition to `all/2`, Repo provides a number of callbacks including `one/2`, `get/3`, `insert/2`, and `delete/2`.  A complete list of callbacks can be found at [Ecto.Repo#callbacks](http://hexdocs.pm/ecto/Ecto.Repo.html#callbacks).

### Count

If we want to count the number of users that have confirmed account we could use `count/1`:

```elixir
query =
  from(
    u in User,
    where: u.confirmed == true,
    select: count(u.id)
  )
```

There is `count/2` function that counts the distinct values in given entry:

```elixir
query =
  from(
    u in User,
    where: u.confirmed == true,
    select: count(u.id, :distinct)
  )
```

### Group By

To group users by their confirmation status we can include the `group_by` option:

```elixir
query =
  from(
    u in User,
    group_by: u.confirmed,
    select: [u.confirmed, count(u.id)]
  )

Repo.all(query)
```

### Order By

Ordering users by their creation date:

```elixir
query =
  from(
    u in User,
    order_by: u.inserted_at,
    select: [u.username, u.inserted_at]
  )

Repo.all(query)
```

To order by `DESC`:

```elixir
query =
  from(
    u in User,
    order_by: [desc: u.inserted_at],
    select: [u.username, u.inserted_at]
  )
```



