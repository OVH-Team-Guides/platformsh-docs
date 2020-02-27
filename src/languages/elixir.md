# Elixir

Platform.sh supports building and deploying applications written in Elixir.  They are automatically compiled during the build phase, and support both committed dependencies and download-on-demand. The underlying Erlang version is 22.0.7.

## Supported versions

* 1.9

To specify an Elixir container, use the `type` property in your `.platform.app.yaml`.

{% codesnippet "/registry/images/examples/full/elixir.app.yaml", language="yaml" %}{% endcodesnippet %}

## Assumptions

Platform.sh is making assumptions about your application to provide a more streamlined experience. These assumptions are the following:

- Your `mix.exs` file is at the root of your application.

Platform.sh will then run `mix do deps.get, deps.compile, compile` on your application to build a binary.

If you don't want these assumptions, you can disable this behavior by specifying in your `.platform.app.yaml`:

{% codesnippet "/registry/images/examples/full/elixir.build.app.yaml", language="yaml" %}{% endcodesnippet %}

## Dependencies

The recommended way to handle Elixir dependencies on Platform.sh is using Hex. Commit a `mix.exs` file in your repository and the system will automatically download the dependencies in you `deps` section.

```elixir
  defp deps do
    [
	  {:platformshconfig, "~> 0.1.0"}
    ]
  end
```


## Platform.sh variables

Platform.sh exposes relationships and other configuration as [environment variables](/development/variables.md). Most notably, it allows a program to determine at runtime what HTTP port it should listen on and what the credentials are to access [other services](/configuration/services.md).

To get the `PORT` environment variable (the port on which your web application is supposed to listen) you would:

```elixir
String.to_integer(System.get_env("PORT") || "8888")
```

Some of the environment variables are in JSON format and are base64 encoded. You would need to import a JSON parsing library such as Jason or Poison to read those.

> CAVEAT: Remember `config/prod.exs` is evaluated at **build time** and will not have access to runtime configuration. Use `config/releases.exs` to configure your runtime environment.

## Building and running the application

Assuming `mix.exs` is present in your repository, the application will be automatically built on push.  You can then start it from the `web.commands.start` directive.  Note that the start command _must_ run in the foreground, so you should set the `--no-halt` flag when calling `mix run`.

The following basic `.platform.app.yaml` file is sufficient to run most Elixir applications.

```yaml
name: app
type: elixir:1.9
web:
    commands:
        start: mix run --no-halt
    locations:
        /:
            allow: false
            passthru: true
variables:
    env:
        MIX_ENV: 'prod'
disk: 512
```

Note that there will still be a proxy server in front of your application.  If desired, certain paths may be served directly by our router without hitting your application (for static files, primarily) or you may route all requests to the Elixir application unconditionally, as in the example above.

# Accessing Services

The simplest possible way to go around this is to use the [Platform.sh Config Reader](https://hex.pm/packages/platformshconfig) library from hex.

If you are building a Phoenix app for example, it would suffice to add a database to `.platform/services.yaml` and a relationship in `.platform.app.yaml`. Put the lib in your `deps` and, assuming you renamed the `proc.secrets.exs` to `releases.exs` per the [Phoenix guide](https://hexdocs.pm/phoenix/releases.html), change:

```
System.get_env("DATABASE_URL")
```

to

```
Platformsh.Config.ecto_dsn_formatter("database")
```

See [Platform.sh Config Reader Documentation](https://hexdocs.pm/platformshconfig/Platformsh.Config.html) for the full API.

# Accessing Services Manually

The services configuration is available in the environment variable `PLATFORM_RELATIONSHIPS`.

Given a relationship defined in `.platform.app.yaml`:

{% codesnippet "/registry/images/examples/full/postgresql.app.yaml", language="yaml" %}{% endcodesnippet %}

Assuming you have in `mix.exs` the Poison library to parse JSON:

```elixir
  defp deps do
    [
      {:poison, "~> 3.0"}
    ]
  end
```

And assuming you use `ecto` you could put in `config/config.exs`:

```elixir
relationships = Poison.decode!(Base.decode64!(System.get_env("PLATFORM_RELATIONSHIPS")))
[postgresql_config | _tail] = relationships["postgresdatabase"]

config :my_app, Repo,
  database: postgresql_config["path"],
  username: postgresql_config["username"],
  password: postgresql_config["password"],
  hostname: postgresql_config["host"]
```

## Project templates

Platform.sh offers [a project template for Elixir applications](https://github.com/platformsh/template-elixir) using the structure described above. It can be used as a starting point or reference for building your own website or web application.