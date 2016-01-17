---
layout: post
title: Postgres UUID Primary Keys in Phoenix
---

I'm at the begining stages of converting a Rails IoT project to Phoenix. Initially I plan to replicate my temperature dashboard, which has the below graph on it.

![Temperature Dashboard](/assets/images/homeatron9k-dashboard-temps.png)

The green line is the air temperature in my ceiling space, I have a metal roof which is why it heats up to over 50C on some days. Something needs to be done about the heat as it has a massive flow on effect throughout the house in summer. That's another post though.

So far I've created a blank Phoenix app and begun to setup a few models to represent my existing Rails tables so that I can pull the data which powers the dashboard. I immediately encountered an issue as I'm using UUID primay keys everywhere.

In Rails I would create my tables this way:

{% highlight ruby %}
class CreateReadings < ActiveRecord::Migration
  def change
    create_table :readings, id: :uuid do |t|
      t.json :raw_data
      t.float :timestamp
    end
  end
end
{% endhighlight %}

As far as I'm aware there is no standard way to default the primary key to being `:uuid` in Rails, and as a result I'd sometimes forget. The same seems to be the case for Ecto migrations too.

{% highlight elixir %}
defmodule MyApp.Repo.Migrations.CreateReading do
  use Ecto.Migration

  def change do
    create table(:readings, primary_key: false) do
      add :id, :uuid, primary_key: true, default: fragment("uuid_generate_v4()")
      add :raw_data, :map
      add :timestamp, :float

      timestamps
    end

  end
end
{% endhighlight %}

If you get the following error when running the above migration with `mix ecto.migrate`

{% highlight bash %}
ERROR (undefined_function): function uuid_generate_v4() does not exist
{% endhighlight %}

Then you need to add the UUID extension to your database from psql like this:

{% highlight postgresql %}
CREATE EXTENSION IF NOT EXISTS "uuid-ossp" WITH SCHEMA public;
{% endhighlight %}

Or you can create an extension to do it for you, just make sure your Postgres user has the appropriate rights, and that this migration runs first.

{% highlight elixir %}
defmodule MyApp.Repo.Migrations.AddUuidExtension do
  use Ecto.Migration

  def up do
    execute "CREATE EXTENSION IF NOT EXISTS \"uuid-ossp\" WITH SCHEMA public;"
  end

  def down do
    execute "DROP EXTENSION \"uuid-ossp\";"
  end
end
{% endhighlight %}

To then get Ecto in Phoenix using database generated UUIDs isn't straight forward as it doesn't support this, only client side generation is. After a bit of research I've found you can get database generated primary keys used, but it's a bit of a hack.

Reading the [Ecto.Schema](http://hexdocs.pm/ecto/Ecto.Schema.html#content) docs shows how straight forward is it to switch the default primary and foreign key types to `:uuid`. The problem is that this line in `MyApp.Web.Model` will cause Ecto to generate the primary_key client side and send that in the `insert` statements:

{% highlight elixir %}
@primary_key {:id, :binary_id, autogenerate: true}
{% endhighlight %}

If I then run `Repo.insert %Reading{}` from `iex -S mix` I get the following debug output, note the sql generated includes the UUID being set.

{% highlight bash %}
{% raw %}
[debug] INSERT INTO "readings" ("id", "inserted_at", "updated_at", "raw_data", "timestamp")
  VALUES ($1, $2, $3, $4, $5) [<<200, 142, 251, 97, 204, 162, 64, 238, 175, 67, 139, 113, 6, 50, 86, 153>>, {{2016, 1, 16}, {23, 40, 42, 342846}}, {{2016, 1, 16}, {23, 40, 42, 342841}}, nil, nil] OK query=95.5ms queue=7.5ms
{% endraw %}
{% endhighlight %}

In particular it's the `autogenerate` option causing this, if this was of type `:id` instead of `:binary_id` Ecto would let the database generate the ID. You can see that [stated in the code](https://github.com/elixir-lang/ecto/blob/v1.1.1/lib/ecto/adapters/sql.ex#L80) Ecto is generating the UUID. So what to do about this, you can remove the `autogenerate: true`, but that doesn't stop Ecto sending the `id` field, it's just now `nil` which causes a constraint error:

{% highlight bash %}
{% raw %}
[debug] INSERT INTO "readings" ("inserted_at", "updated_at", "id", "raw_data", "timestamp") VALUES ($1, $2, $3, $4, $5) [{{2016, 1, 16}, {23, 54, 9, 340068}}, {{2016, 1, 16}, {23, 54, 9, 340063}}, nil, nil, nil] ERROR query=94.5ms queue=7.4ms
** (Postgrex.Error) ERROR (not_null_violation): null value in column "id" violates not-null constraint
    (ecto) lib/ecto/adapters/sql.ex:497: Ecto.Adapters.SQL.model/6
    (ecto) lib/ecto/repo/schema.ex:297: Ecto.Repo.Schema.apply/5
    (ecto) lib/ecto/repo/schema.ex:81: anonymous fn/11 in Ecto.Repo.Schema.do_insert/4
{% endraw %}
{% endhighlight %}

The trick is to use [before_insert](http://hexdocs.pm/ecto/Ecto.Model.Callbacks.html#before_insert/2) from Ecto.Model.CallBacks to remove the `:id` field from the changeset before the query statement is generated. `MyApp.Web.Model` can be updated like this:

{% highlight elixir %}
@primary_key {:id, :binary_id, read_after_writes: true}
@foreign_key_type :binary_id

use Ecto.Model.Callbacks
before_insert MyApp.DontSetId, :remove_id
{% endhighlight %}

{% highlight elixir %}
defmodule MyApp.DontSetId do
  def remove_id(changeset) do
    Ecto.Changeset.delete_change(changeset, :id)
  end
end
{% endhighlight %}

The `read_after_writes` option will not trigger a select but simply means the database generated id is read from the return response. Now when I insert a new record the `:id` field is not being set, a `RETURNING` statement is included and the returned model has a primary key.

{% highlight bash %}
{% raw %}
[debug] BEGIN [] OK query=88.3ms queue=6.7ms
[debug] INSERT INTO "readings" ("inserted_at", "updated_at", "raw_data", "timestamp") VALUES ($1, $2, $3, $4) RETURNING "id" [{{2016, 1, 17}, {0, 13, 28, 146557}}, {{2016, 1, 17}, {0, 13, 28, 146553}}, nil, nil] OK query=2.1ms
[debug] COMMIT [] OK query=0.9ms
{:ok,
 %UuidPrimaryKeys.Reading{__meta__: #Ecto.Schema.Metadata<:loaded>,
  id: "ece5530a-4268-4b28-9815-a548ef3383e8",
  inserted_at: #Ecto.DateTime<2016-01-17T00:13:28.146557Z>, raw_data: nil,
  timestamp: nil, updated_at: #Ecto.DateTime<2016-01-17T00:13:28.146553Z>}}
{% endraw %}
{% endhighlight %}

The convention is to let the client generate the UUID as by their very nature it will be unique, but at this stage I'm going to stick with how the system I'm replacing works. It's not a great idea to immediately go against convention when learning something new, but this hack is quite self contained, and I sure did learn heaps trying to figure all this out.