---
version: 1.0.0
title: Querying Data
---

# Setup content taken from [@SophieDeBenedetto's PR](https://github.com/elixirschool/elixirschool/pull/1553)
# Notes to future us:
- Where do we draw the line between Associations content + Querying content?
- How do we handle duplicated repo setup across multiple lessons? Do we keep a "base" project in the ElixirSchool repo?

##########################################################################
##########################################################################
################### COPIED FROM SOPHIE'S LESSON ##########################
##########################################################################
##########################################################################

## Querying Data

In this section we'll learn how to use Ecto to define and work with associations between our schemas.

## Table of Contents

* Set up
* Types of associations
  * Belongs to/has many
  * Belongs to/has one
  * Many to Many
* Saving associated data
  * Belongs to
  * Many to many

## Set Up

First, we'll set up a simple app that uses Ecto. These steps are taken from the [Getting Started](https://hexdocs.pm/ecto/getting-started.html#adding-ecto-to-an-application) guide of the official Ecto documentation.

Create a new app with a supervision tree:

```shell
mix new movie_db --sup
cd movie_db
```

Open the `mix.exs` file and add the Ecto and Postgrex dependencies:

```elixir
# mix.exs
defp deps do
  [{:ecto, "~> 2.0"},
   {:postgrex, "~> 0.11"}]
end
```

Install your dependencies with `mix deps.get`.

Next, generate the Repo for the application.

```shell
mix ecto.gen.repo -r MovieDb.Repo
```

Now we're ready to configure our newly created Repo.

First, check out the database configuration and make sure the details and credentials are correct:

```elixir
# config/config.exs

config :movie_db, MovieDb.Repo,
  adapter: Ecto.Adapters.Postgres,
  database: "movie_db_repo",
  username: "postgres",
  password: "postgres",
  hostname: "localhost"
```

Next up, add the repo to your application config so that Ecto knows what repo to execute tasks against:

```elixir
# config/config.exs
...
config :movie_db,
  ecto_repos: [MovieDb.Repo]
```

Lastly, we need to tell our app to start up the repo under a supervisor when the app starts:

```elixir
# application.ex

def start(_type, _args) do
  import Supervisor.Spec

  children = [
    supervisor(MovieDb.Repo, [])
  ]

  opts = [strategy: :one_for_one, name: MovieDb.Supervisor]
  Supervisor.start_link(children, opts)
end
```

Now we're ready to create the database:

```shell
mix ecto.create
```

## Types of Associations

There are three types of associations we can define between our schemas. We'll look at what they are and how to implement each type of relationship.

### Belongs To/Has Many

We're building out a simple app, MovieDb, that catalogues our favorite films. We'll start with two schemas: `Movie` and `Character`. We'll implement a "has many/belongs to" relationship between these two schemas: A movie has many characters and a character belongs to a movie.

#### The Has Many Migration

Let's generate a migration for `Movie`:

```shell
mix ecto.gen.migration create_movie
```

Open up the newly generated migration file and define your `change` function to create the `movies` table with a few attributes:

```elixir
# priv/migrations/*_create_movie.exs
defmodule MovieDb.Repo.Migrations.CreateMovie do
  use Ecto.Migration

  def change do
    create table(:movies) do
      add :title, :string
      add :tagline, :string
    end
  end
end
```

#### The Has Many Schema

We'll add a schema that specifies the "has many" relationship between a movie and its characters.

```elixir
# lib/movie.ex
defmodule MovieDb.Movie do
  use Ecto.Schema

  schema "movies" do
    field :title, :string
    field :tagline, :string
    has_many :characters, MovieDb.Character
  end
end
```

The `has_many` macro doesn't add anything to the database itself. What it does is use the foreign key on the associated schema, `characters`, to make a movie's associated characters available. This is what will allow us to call `movie.characters`.

#### The Belongs To Migration

Now we're ready to build our `Character` migration and schema. A character belongs to a movie, so we'll define a migration and schema that specifies this relationship.

First, generate the migration:

```shell
mix ecto.gen.migration create_character
```

To declare that a character belongs to a movie, we need the `characters` table to have a `movie_id` column. We want this column to function as a foreign key. We can accomplish this with the following line in our `create_table` function:

```elixir
add :movie_id, references(:movies)
```
So our migration should look like this:

```elixir
# priv/migrations/*_create_chaaracter.exs
defmodule MovieDb.Repo.Migrations.CreateCharacter do
  use Ecto.Migration

  def change do
    create table(:characters) do
      add :name, :string
      add :movie_id, references(:movies)
    end
  end
end
```

#### The Belongs To Schema

Our `Character` schema likewise needs to define the "belongs to" relationship between a character and its movie.

```elixir
# lib/character.ex

defmodule MovieDb.Character do
  use Ecto.Schema

  schema "characters" do
    field :name, :string
    belongs_to :movie, MovieDb.Movie
  end
end
```

Let's take a closer look at what the `belongs_to` macro does for us. Unlike adding the `movie_id` column to our `characters` table, this macro _doesn't_ add anything to the database. It _does_ give us the ability to access our associated `movies` schema _through_ `characters`. It uses the the foreign key of `movie_id` on the `characters` table to make a character's associated movie available when we query for characters. This is what will allow us to call `character.movie`.

Now we're ready to run our migrations:

```shell
mix ecto.migrate
```

### Belongs To/Has One

Let's say that a movie has one distributor, for example Netflix is the distributor of their original film "Bright".

We'll define the `Distributor` migration and schema with the `belongs_to` relationship. First, let's generate the migration:

```shell
mix ecto.gen.migration create_distributor
```

Our migration should add a foreign key of `movie_id` to the `distributors` table:

```elixir
# priv/migrations/*_create_distributor.ex

defmodule MovieDb.Repo.Migrations.CreateDistributor do
  use Ecto.Migration

  def change do
    create table(:distributor) do
      add :name, :string
      add :movie_id, references(:movies)
    end
  end
end
```

And the `Distributor` schema should use the `belongs_to` macro to allow us to call `distributor.movie` and look up a distributor's associated movie using this foreign key.

```elixir
# lib/distributor.ex

defmodule MovieDb.Distributor do
  use Ecto.Schema

  schema "distributors" do
    field :name, :string
    belongs_to :movie, MovieDb.Movie
  end
end
```

Next up, we'll add the "has one" relationship to the `Movie` schema:

```elixir
# lib/movie.ex

defmodule MovieDb.Movie do
  use Ecto.Schema

  schema "movies" do
    field :title, :string
    field :tagline, :string
    has_many :characters, MovieDb.Character
    has_one :distributor, MovieDb.Distributor # I'm new!
  end
end
```

The `has_one` macro functions just like the `has_many` macro. It doesn't add anything to the database but it _does_ use the associated schema's foreign key to look up and expose the movie's distributor. This will allow us to call `movie.distributor`.

We're ready to run our migrations:

```shell
mix ecto.migrate
```

### Many To Many

Let's say that a movie has many actors and that an actor can belong to more than one movie. We'll build a join table that references _both_ movies _and_ actors to implement this relationship.

First, let's generate the `Actor` migration:

Generate the migration:

```shell
mix ecto.gen.migration create_actor
```

Define the migration:

```Elixir
# priv/migrations/*_create_actor.ex

defmodule MovieDb.Repo.Migrations.CreateActor do
  use Ecto.Migration

  def change do
    create table(:actors) do
      add :name, :string
    end
  end
end
```

Let's generate our join table migration:

```shell
mix ecto.gen.migration create_movies_actors
```

We'll define our migration such that the table has two foreign keys. We'll also add a unique index to enforce unique pairings of actors and movies:

```elixir
# priv/migrations/*_create_movies_actors.ex

defmodule MovieDb.Repo.Migrations.CreateMoviesActors do
  use Ecto.Migration

  def change do
    create table(:movies_actors) do
      add :movie_id, references(:movies)
      add :actor_id, references(:actors)
    end

    create unique_index(:movies_actors, [:movie_id, :actor_id])
  end
end
```

Next up, let's add the `many_to_many` macro to our `Movies` schema:

```elixir
# lib/movie.ex

defmodule MovieDb.Movie do
  use Ecto.Schema

  schema "movies" do
    field :title, :string
    field :tagline, :string
    has_many :characters, MovieDb.Character
    has_one :distributor, MovieDb.Distributor
    many_to_many :actors, MovieDb.Actor, join_through: "movies_actors" # I'm new!
  end
end
```

Finally, we'll define our `Actor` schema with the same `many_to_many` macro.

```elixir
# lib/actor.ex

defmodule MovieDb.Actor do
  use Ecto.Schema

  schema "actors" do
    field :name, :string
    many_to_many :movies, MovieDb.Movie, join_through: "movies_actors"
  end
end
```

We're ready to run our migrations:

```shell
mix ecto.migrate
```

## Saving Associated Data

The manner in which we save records along with their associated data depends on the nature of the relationship between the records. Let's start with the "belongs to/has many" relationship.

### Belongs To

#### Saving With `Ecto.build_assoc/3`

With a `belongs_to` relationship, we can leverage Ecto's `build_assoc` function.

[`build_assoc`](https://hexdocs.pm/ecto/Ecto.html#build_assoc/3) takes in three arguments:

* The struct of the record we want to save.
* The name of the association.
* Any attributes we want to assign to the associated record we are saving.

Let's save a movie and and associated character:

First, we'll create a movie record:

```elixir
iex> alias MovieDb.{Movie, Character, Repo}
iex> movie = %Movie{title: "Ready Player One", tagline: "Something about video games"}

%MovieDb.Movie{
  __meta__: #Ecto.Schema.Metadata<:built, "movies">,
  actors: #Ecto.Association.NotLoaded<association :actors is not loaded>,
  characters: #Ecto.Association.NotLoaded<association :characters is not loaded>,
  distributor: #Ecto.Association.NotLoaded<association :distributor is not loaded>,
  id: nil,
  tagline: "Something about video games",
  title: "Ready Player One"
}

iex> movie = Repo.insert!(movie)
# NOTE: We should probably put the output here (to standardize)
```

NOTE: maybe worth discussing diff between Repo.insert and Repo.insert!

Now we'll build our associated character and insert it into the database:

```shell
character = Ecto.build_assoc(movie, :characters, %{name: "Wade Watts"})
%MovieDb.Character{
  __meta__: #Ecto.Schema.Metadata<:built, "characters">,
  id: nil,
  movie: #Ecto.Association.NotLoaded<association :movie is not loaded>,
  movie_id: 1,
  name: "Wade Watts"
}
Repo.insert!(character)
%MovieDb.Character{
  __meta__: #Ecto.Schema.Metadata<:loaded, "characters">,
  id: 1,
  movie: #Ecto.Association.NotLoaded<association :movie is not loaded>,
  movie_id: 1,
  name: "Wade Watts"
}
```

Notice that since the `Movie` schema's `has_many` macro specifies that a movie has many `:characters`, the name of the association that we pass as a second argument to `build_assoc` is exactly that: `:characters`. We can see that we've created a character that has its `movie_id` properly set to the ID of the associated movie.

In order to use `build_assoc` to save a movie's associated distributor, we take the same approach of passing the _name_ of the movie's relationship to distributor as the second argument to `build_assoc`:

```elixir
iex> movie = Ecto.build_assoc(movie, :distributor, %{name: "Netflix"})
%MovieDb.Distributor{
  __meta__: #Ecto.Schema.Metadata<:built, "distributors">,
  id: nil,
  movie: #Ecto.Association.NotLoaded<association :movie is not loaded>,
  movie_id: 1,
  name: "Netflix"
}
iex> Repo.insert!(movie)
%MovieDb.Distributor{
  __meta__: #Ecto.Schema.Metadata<:loaded, "distributors">,
  id: 1,
  movie: #Ecto.Association.NotLoaded<association :movie is not loaded>,
  movie_id: 1,
  name: "Netflix"
}
```

# NOTE: distributor(s) table

### Many to Many
#### Saving With `Ecto.Changeset.put_assoc/3`

The `build_assoc` approach won't work for our many-to-many relationship. That is because neither the movie nor actor tables contain a foreign key. Instead, we need to leverage Ecto Changesets and the `put_assoc` function.

Assuming we already have the movie record we created above, let's create an actor record:

```elixir
iex> actor = %Actor{name: "Tyler Sheridan"}
%MovieDb.Actor{
  __meta__: #Ecto.Schema.Metadata<:built, "actors">,
  id: nil,
  movies: #Ecto.Association.NotLoaded<association :movies is not loaded>,
  name: "Tyler Sheridan"
}
iex> actor = Repo.insert!(actor)
%MovieDb.Actor{
  __meta__: #Ecto.Schema.Metadata<:loaded, "actors">,
  id: 1,
  movies: #Ecto.Association.NotLoaded<association :movies is not loaded>,
  name: "Tyler Sheridan"
}
```

Now we're ready to associate our movie to our actor via the join table.

First, note that in order to work with Changesets, we need to make sure that our `movie` record has preloaded its associated schemas. We'll talk more about preloading data in a bit. For now, its enough to understand that we can preload our associations like this:

```elixir
iex> movie = Repo.preload(movie, [:distributor, :characters, :actors])
%MovieDb.Movie{
  __meta__: #Ecto.Schema.Metadata<:loaded, "movies">,
  actors: [],
  characters: [],
  distributor: nil,
  id: 1,
  tagline: "Something about video games",
  title: "Ready Player One"
}
```

Next up, we'll create a changeset for our movie record:

```elixir
iex> movie_changeset = Ecto.Changeset.change(movie)
#Ecto.Changeset<action: nil, changes: %{}, errors: [], data: #MovieDb.Movie<>,
 valid?: true>
```

Now we'll pass our changeset as the first argument to `Ecto.Changeset.put_assoc/3`:

```elixir
iex> movie_actors_changeset = movie_changeset |> Ecto.Changeset.put_assoc(:actors, [actor])
#Ecto.Changeset<
  action: nil,
  changes: %{
    actors: [
      #Ecto.Changeset<action: :update, changes: %{}, errors: [],
       data: #MovieDb.Actor<>, valid?: true>
    ]
  },
  errors: [],
  data: #MovieDb.Movie<>,
  valid?: true
>
```

This gives us a _new_ changeset that represents the following change: add the actors in this list of actors to the give movie record.

Lastly, we'll update the given movie and actor records using our latest changeset:

```elixir
iex> Repo.update!(movie_actors_changeset)
%MovieDb.Movie{
  __meta__: #Ecto.Schema.Metadata<:loaded, "movies">,
  actors: [
    %MovieDb.Actor{
      __meta__: #Ecto.Schema.Metadata<:loaded, "actors">,
      id: 1,
      movies: #Ecto.Association.NotLoaded<association :movies is not loaded>,
      name: "Bob"
    }
  ],
  characters: [],
  distributor: nil,
  id: 1,
  tagline: "Something about video games",
  title: "Ready Player One"
}
```

We can see that this gives us a movie record with the new actor properly associated and already preloaded for us under `movie.actors`

We can use this same approach to create a brand new actor that is associated with the given movie. Instead of passing a _saved_ actor struct into `put_assoc`, we simply pass in an actor struct describing a new actor that we want to create:

```elixir
iex> changeset = movie_changeset |> Ecto.Changeset.put_assoc(:actors, [%{name: "Gary"}])
#Ecto.Changeset<
  action: nil,
  changes: %{
    actors: [
      #Ecto.Changeset<
        action: :insert,
        changes: %{name: "Gary"},
        errors: [],
        data: #MovieDb.Actor<>,
        valid?: true
      >
    ]
  },
  errors: [],
  data: #MovieDb.Movie<>,
  valid?: true
>
iex>  Repo.update!(changeset)
%MovieDb.Movie{
  __meta__: #Ecto.Schema.Metadata<:loaded, "movies">,
  actors: [
    %MovieDb.Actor{
      __meta__: #Ecto.Schema.Metadata<:loaded, "actors">,
      id: 2,
      movies: #Ecto.Association.NotLoaded<association :movies is not loaded>,
      name: "Gary"
    }
  ],
  characters: [],
  distributor: nil,
  id: 1,
  tagline: "Something about video games",
  title: "Ready Player One"
}
```

We can see that a new actor was created with an ID of "2" and the attributes we assigned it.

## Querying For Associated data
### Preloading
In order to be able to access the associated records that the  `belongs_to`, `has_many` and `has_one` macros expose to us, we need to _preload_ the associated schemas.

Let's take a look a see what happens when we try to ask a movie for its associated actors:

```elixir
iex> movie = Repo.get!(Movie, 1)
iex> movie.actors
#Ecto.Association.NotLoaded<association :actors is not loaded>
```

We _can't_ access those associated characters unless we preload them. There are a few different way to preload records with Ecto.

#### Preloading With Two Queries

The following query will preload associated records in a _separate_ query.

```elixir
iex> import Ecto.Query
Ecto.Query
iex> Repo.all(from m in Movie, preload: [:actors])

08:53:01.078 [debug] QUERY OK source="movies" db=107.4ms decode=16.8ms
SELECT m0."id", m0."title", m0."tagline" FROM "movies" AS m0 []

08:53:01.120 [debug] QUERY OK source="actors" db=14.9ms
SELECT a0."id", a0."name", m1."id" FROM "actors" AS a0 INNER JOIN "movies" AS m1 ON m1."id" = ANY($1) INNER JOIN "movies_actors" AS m2 ON m2."movie_id" = m1."id" WHERE (m2."actor_id" = a0."id") ORDER BY m1."id" [[1]]
[
  %MovieDb.Movie{
    __meta__: #Ecto.Schema.Metadata<:loaded, "movies">,
    actors: [
      %MovieDb.Actor{
        __meta__: #Ecto.Schema.Metadata<:loaded, "actors">,
        id: 1,
        movies: #Ecto.Association.NotLoaded<association :movies is not loaded>,
        name: "Bob"
      },
      %MovieDb.Actor{
        __meta__: #Ecto.Schema.Metadata<:loaded, "actors">,
        id: 2,
        movies: #Ecto.Association.NotLoaded<association :movies is not loaded>,
        name: "Gary"
      }
    ],
    characters: #Ecto.Association.NotLoaded<association :characters is not loaded>,
    distributor: #Ecto.Association.NotLoaded<association :distributor is not loaded>,
    id: 1,
    tagline: "Something about video games",
    title: "Ready Player One"
  }
]
```

We can see that the above line of code ran _two_ database queries. One for all of the movies, and another for all of the actors with the given movie IDs.


#### Preloading With One Query
We can cut down on our database queries with the following:

```elixir
iex> Repo.all(from m in Movie, join: a in assoc(m, :actors), preload: [actors: a])

08:55:46.146 [debug] QUERY OK source="movies" db=4.9ms
SELECT m0."id", m0."title", m0."tagline", a1."id", a1."name" FROM "movies" AS m0 INNER JOIN "movies_actors" AS m2 ON m2."movie_id" = m0."id" INNER JOIN "actors" AS a1 ON m2."actor_id" = a1."id" []
[
  %MovieDb.Movie{
    __meta__: #Ecto.Schema.Metadata<:loaded, "movies">,
    actors: [
      %MovieDb.Actor{
        __meta__: #Ecto.Schema.Metadata<:loaded, "actors">,
        id: 1,
        movies: #Ecto.Association.NotLoaded<association :movies is not loaded>,
        name: "Bob"
      },
      %MovieDb.Actor{
        __meta__: #Ecto.Schema.Metadata<:loaded, "actors">,
        id: 2,
        movies: #Ecto.Association.NotLoaded<association :movies is not loaded>,
        name: "Gary"
      }
    ],
    characters: #Ecto.Association.NotLoaded<association :characters is not loaded>,
    distributor: #Ecto.Association.NotLoaded<association :distributor is not loaded>,
    id: 1,
    tagline: "Something about video games",
    title: "Ready Player One"
  }
]
```

This allows us to execute just one database call. It also has the added benefit of allowing us to select and filter both movies and associated actors in the same query. For example, this approach allows us to query for all movies where the associated actors meet certain conditions. Something like:

```elixir
Repo.all from m in Movie,
  join: a in assoc(m, :actors),
  where: a.name == "John Wayne"
  preload: [actors: a]
```

#### Preloading Fetched Records

We can also preload the associated schemas of records that have already been queried from the database.

```elixir
iex> movie = Repo.get!(Movie, 1)
%MovieDb.Movie{
  __meta__: #Ecto.Schema.Metadata<:loaded, "movies">,
  actors: #Ecto.Association.NotLoaded<association :actors is not loaded>, # actors are NOT LOADED!!
  characters: #Ecto.Association.NotLoaded<association :characters is not loaded>,
  distributor: #Ecto.Association.NotLoaded<association :distributor is not loaded>,
  id: 1,
  tagline: "Something about video games",
  title: "Ready Player One"
}
iex> movie = Repo.preload(movie, :actors)
%MovieDb.Movie{
  __meta__: #Ecto.Schema.Metadata<:loaded, "movies">,
  actors: [
    %MovieDb.Actor{
      __meta__: #Ecto.Schema.Metadata<:loaded, "actors">,
      id: 1,
      movies: #Ecto.Association.NotLoaded<association :movies is not loaded>,
      name: "Bob"
    },
    %MovieDb.Actor{
      __meta__: #Ecto.Schema.Metadata<:loaded, "actors">,
      id: 2,
      movies: #Ecto.Association.NotLoaded<association :movies is not loaded>,
      name: "Gary"
    }
  ], # actors are LOADED!!
  characters: [],
  distributor: #Ecto.Association.NotLoaded<association :distributor is not loaded>,
  id: 1,
  tagline: "Something about video games",
  title: "Ready Player One"
}
```

Now we can ask a movie for its actors:

```elixir
iex> movie.actors
[
  %MovieDb.Actor{
    __meta__: #Ecto.Schema.Metadata<:loaded, "actors">,
    id: 1,
    movies: #Ecto.Association.NotLoaded<association :movies is not loaded>,
    name: "Bob"
  },
  %MovieDb.Actor{
    __meta__: #Ecto.Schema.Metadata<:loaded, "actors">,
    id: 2,
    movies: #Ecto.Association.NotLoaded<association :movies is not loaded>,
    name: "Gary"
  }
]
```

##########################################################################
##########################################################################
################### EXISTING ELIXIR SCHOOL LESSON ########################
##########################################################################
##########################################################################

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



