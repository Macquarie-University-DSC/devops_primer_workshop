# DevOps Primer

This workshop is a primer to beginning system administration and devops. This is a relatively simple deployment of a
hobby project where we will learn to set up nginx with http 2 compression and tls encryption to serve a dynamic website.
This will also introduce concepts of DevOps such as continuous integration and continuous delivery using github actions.
This workshop is mean't to be a primer before we do the real enterprise stuff dealing with google cloud, managed
databases, horizontal scaling, load balancers and such.

In this application we are deploying we have a front end and a back end. Both are seperate applications but both will
be deployed on the same server. Additionally we will have a postgres instance running in the background.

## What is Dev Ops?

Dev ops is kind of like a way of life. It is the idea that we automate a lot of tedious tasks, so that we can focus on
code and automate tasks that would otherwise be tedious to do on it's own. When we talk about Dev Ops we are really
talking about creating a pipeline, or a continuous flow of code to production.

DevOps is often broken down into two stages, continuous integration and continuous delivery/deployment.

Continuous Integration typically looks like this.

1. We push our code from our repository to a repository on the internet (e.g. github)

2. We trigger our CI instance to run tests on the application to verify that it's functions work correctly (called unit
   testing).

3. We trigger a build, building our application into some runnable form such as a binary or a bytecode.

4. We take the output of our build step (called artifacts) and save them, deleting everything else. (Really considered
   a part of the CD phase)

Continuous Delivery and Continuous Deployment are different things and they look very different.

Continuous Delivery is the more modern approach, in continuos delivery we would take our build artifacts from the CI
phase and build an image of a container or a service like a digital ocean droplet or a virtual server instance and deploy
it using an orchestration tool (like kubernetes or terraform). We then perform some tests against our production like
environment (called integration tests, there are also other types of tests), then we deploy to a high availability
environment (something that lots of users can access 24/7). This is the recommend approach for Dev Ops, it is easy to
scale both vertically (adding more powerful servers) and horizontally (adding more servers in general). On the other hand
it costs a lot of $$$ to maintain and use this kind of infrastructure. Another use of continuous delivery is just
releasing the binaries as is on things like github releases or publishing to the app store. This is more for people like
games developers who continually build actual video games, and have no server production environment they need to manage.

Continuous Deployment on the other hand is taking the artifacts from the CI phase and automating the deployment directly
on the server using the artifacts. This approach only scales vertically, on the other hand it is great for static
websites using services like github pages, netlify or gitlab pages. It is also a great option for instances where you
would want all your services running on the same server (if you had a business that used their own servers).

Essentially CD looks like this.

Get Artifacts -> Test against production environment -> Deploy

## Prerequisites

In terms of knowledge it would be helpful but not necessarily required to knowledge

- Basic unix/linux file structures and directory structures (not essential)
- Basic unix/linux commands (just look them up if you struggle)
- Vim or Vi text editing (can also use the nano text editor, but vscode is not gonna help)

In terms of what software would be required to follow along in case you want to follow along which is not required, can
just listen.

1. A github account connected to your university or instituition (see 2)

2. Would need the student developer pack from github, should take about three seconds and a university email to apply
   for one.

3. Namecheap domain (free with student developer pack and a .me extension) Look up instructions on github student pack
   and google.

4. Some vps (digital ocean droplet recommended) just the account needed setting them up will be covered, also doing
   these things are provider agnostic, i'll  be using digital ocean but these apply to all platforms, since they all
   provide similar experiences.

5. You NEED some way of getting around, some sort of unix shell. Instructions
   below.

6. You also need to have vagrant and ansible installed on your local computer for testing purposes.

### Windows Unix Shell

In windows there are many options for getting unix shells. Putty is one, but not recommended for system administration
work. The two most recommended options are to either use a Virtual Machine, but they can be quite tedious to set up, or
you can use wsl. It is recommended that you use wsl.

WSL stands for windows subsytem for linux, and is essentially a linux kernel embedded in windows allowing windows users
access to the linux cli.

In order to get it follow this guide https://docs.microsoft.com/en-us/windows/wsl/install-win10

I would personally recommend downloading either Debian, Fedora or openSUSE for WSL.

Debian has the most available packages, but you will only be a few here. OpenSUSE Leap, is well regarded as one of the
best linux distros with good default tools which would probably be my own personal choice. Then Fedora is very similar
to CentOS which is what we will be using as the deployment environment, generally it is a good idea to have your
development environment be as close to your deployment environment as possible.

The default packages installed with wsl should be fine.

### MacOS

Make sure ssh is installed and enabled, just the client, not the server.

It is recommended to install vim or a cli text editor.

It is also recommended to install rsync for sysadmin work even if it isn't goint to be user here.

### Linux

See the macos for packages, but everything else should be fine.

## Setting up CI pipelines for testing

### Building a unit testing pipeline in github actions for elixir api

1. Our first step is to fork the tasks api project from the Macquarie DSC's github. Navigate to
   https://github.com/Macquarie-University-DSC/tasks-site and click the fork button.

2. Our next step should be to clone the repository into our local computer. Navigate to the repository that you have
   forked. Click on Code, and copy and paste the link (or the ssh link if you have set up ssh on your computer), and
   run the command `git clone link_you_copied.git`

3. In your new directory, create the folder `.github/workflows` with `mkdir .github && mkdir .github/workflows`

4. Create and open a new file, I will call mine `elixir-actions.yml` in the folder `.github/workflows/elixir-actions.yml`

5. We now have to create our first pipeline

Our first pipeline we need to do a couple of things, usually the first step of a CI pipeline is running unit tests, this
just tests that each individual function of our web app works correctly. We need to test that functions such as creating
new id's and then deleting them from a database works as expected, to do this we need to have a database instance running
in our pipeline.

```yaml
name: Elixir Actions
on: [push]
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:13.3-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: tasks_api_test
        ports:
          - 5432:5432
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
    env:
      MIX_ENV: test
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - uses: actions/checkout@v2
      - uses: erlef/setup-beam@v1
        with:
          otp-version: '24.0.2'
          elixir-version: '1.12.0'
      - run: mix deps.get
      - run: mix test
```

Let us break this down.

The name is what we want to call our pipeline, for large projects we could have multiple pipelines doing different things
for most people they would only have one.

jobs is the steps in the pipeline that we need to take, so far we only will have one step, running unit tests.

remember our steps

checkout from source control -> run unit tests -> build application

So first we create a job, called unit-tests, which we run our unit tests in, in this job we run on a linux environment,
github actions only supports running on ubuntu. So we pick `ubuntu-latest`

We need to have a test database setup and configured, first we need to configure a docker image that will run in the
background.

Our docker image uses alpine, a lightweight linux distribution for containers, and is hard locked to use 13.3 for
consistancy. We set a few environment variables on our postgres image, setting the username and password to postgres and
setting the database name to be tasks_api_test. We also need to forward the 5432 port so that our test can access the
database, and the last option checks if the database is ready before continuing on to the next step.

We now need to slightly modify the configuration of our test build in our elixir app.

Navigate to `config/test.exs`

Modify the config to look like this

```elixir
use Mix.Config

# Configure your database
#
# The MIX_TEST_PARTITION environment variable can be used
# to provide built-in test partitioning in CI environment.
# Run `mix help test` for more information.
config :tasks_api, TasksApi.Repo,
  username: "postgres",
  password: "postgres",
  database: "tasks_api_test#{System.get_env("MIX_TEST_PARTITION")}",
  hostname: "localhost",
  pool: Ecto.Adapters.SQL.Sandbox

if System.get_env("GITHUB_ACTIONS") do
  config :tasks_api, TasksApi.Repo,
    username: "postgres",
    password: "postgres"
end

# We don't run a server during test. If one is required,
# you can enable the server option below.
config :tasks_api, TasksApiWeb.Endpoint,
  http: [port: 4002],
  server: false

# Print only warnings and errors during test
config :logger, level: :warn
```

Notice that all we do is add an if statement. What this does is that it checks if the `GITHUB_ACTIONS` environment
variable is set by our devops server, and if it is, it will run the tests on a postgres database using the username and
password we set.

Our next step is to create environment variables used for testing and deployment. The first environment variable tells
our app that we will be using a testing environment for this step. I am not sure what the other environment variable
does not gonna lie but the examples I have seen all have this environment variable set.

Now we need to define the steps in order to actually test the application. If you notice the steps, some are `run`
commands and some are `use` commands. Github has a collection of github actions for doing certain steps. Like we need to
copy our project into our testing environment, github already has a pipeline for this called `actions/checkout@v2`. We
also need to install elixir and the erlang virtual machine for testing purposes. So we use the `erlef/setup-beam@v1`
action by the erlang team. using `with` sets a predefined consistant setup for testing against the same build
environment, and we can update this at a later date. We now only have two commands left to run, `mix deps.get` getting
and installing all the dependencies used to build the application. Our final step is to actually run the unit tests with
`mix test`.

### Building a build pipeline in github actions for elixir app

Our next step in building our testing pipeline is to create a build step. Most would consider this part of the CI phase
since we are not actually deploying anything yet. We are just testing if our application builds successfully. Our first
step is to create a build configuration in our elixir application.

For this step we will be building using elixir releases. This generates a sandboxed environment with all the
dependencies required to build the application. This means that on our actual server, we do not need to install a
erlang runtime.

In order to setup our project follow these steps.

1. First we need to think about what our database will look like in production. I will call the database `tasks_db`,
   and my user will be my default username, I will also set a custom password for it. Our database url we provide to the
   api will now look like `ecto://USERNAME:PASSWORD/tasks_db`

2. Next we need to get a secret, this secret we will probably not reuse so no need to remember it, with
   `mix phx.gen.secret`.

3. We will need to add these to our runtime so execute these commands

```sh
$ export SECRET_KEY_BASE=the_key_you_generated
$ export DATABASE_URL=your_database_url
```

4. We now need to uncomment the line in `config/prod.secret.exs` that describes using releases.

```elixir
# ## Using releases (Elixir v1.9+)
#
# If you are doing OTP releases, you need to instruct Phoenix
# to start each relevant endpoint:

config :tasks_api, TasksApiWeb.Endpoint, server: true

# Then you can assemble a release by calling `mix release`.
# See `mix help release` for more information.
```

and we need to change `use Mix.Config` to `import Config` at the top of the `config.releases.exs` file.

5. Now we need to move this file to `config/releases.exs` with `mv config/prod.secret.exs config/releases.exs`

6. We then need to remove the command that calls the production secrets from the bottom of `config/prod.exs`
   `import_config "prod.secret.exs"`

7. We are ready to test our build, first install production dependencies and compile then create a releases build.

```sh
$ mix deps.get --only prod
$ MIX_ENV=prod mix compile
$ MIX_ENV=prod mix release
```

8. Now if this works correctly we have one final step to do before we can start building our build pipeline, we have to
   set up releases to have a migrate command, and a rollback command for our initial server setup. To do this we create
   a new file `lib/tasks_api/release.ex` that looks like.

```elixir
defmodule TasksApi.Release do
  @app :tasks_api

  def migrate do
    load_app()

    for repo <- repos() do
      {:ok, _, _} = Ecto.Migrator.with_repo(repo, &Ecto.Migrator.run(&1, :up, all: true))
    end
  end

  def rollback(repo, version) do
    load_app()
    {:ok, _, _} = Ecto.Migrator.with_repo(repo, &Ecto.Migrator.run(&1, :down, to: version))
  end

  defp repos do
    Application.fetch_env!(@app, :ecto_repos)
  end

  defp load_app do
    Application.load(@app)
  end
end
```

All this does is adds the ability to run ecto migrations in our elixir release. We can test our build again, to see if
it works using the commands previously described.

Note that we do not need to set the environment variables in production since they are compiled into the application.

#### Writing the build pipeline

Our actual build pipeline would be pretty simple. Our first step is to add encrypted secrets to our github actions CI.

Go to your repository where you have your CI setup, go to Settings -> Secrets and click New repository secret.

Remember the command to generate a secret. Run that again and copy, call this secret `SECRET_KEY_BASE` and it's contents
should be the random string you have generated.

Now remember your database URL, create a new repository secret and it's name should be `DATABASE_URL`, and add your
database url to the contents.

Let us now update our actions workflow to look like this.

```yaml
name: Elixir Actions
on: [push]
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:13.3-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: tasks_api_test
        ports:
          - 5432:5432
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
    env:
      MIX_ENV: test
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - uses: actions/checkout@v2
      - uses: erlef/setup-beam@v1
        with:
          otp-version: '24.0.2'
          elixir-version: '1.12.0'
      - run: mix deps.get 
      - run: mix test

  build:
    needs: unit-tests
    runs-on: ubuntu-latest
    env:
      MIX_ENV: prod
      SECRET_KEY_BASE: ${{ secrets.SECRET_KEY_BASE }}
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
    steps:
      - uses: erlef/setup-beam@v1
        with:
          otp-version: '24.0.2'
          elixir-version: '1.12.0'
      - run: mix compile
      - run: mix release
```

All we have done is added a build step. Our build step is just the commands we have previously done, although the
`needs` tells us to use the files from when we ran the `unit-tests` pipeline. We also set our database url and
secret key base to the secrets we have created. So we are pretty much finished for the API CI pipeline right now.

### Building unit testing pipeline in github actions for elm app

### Building a build pipeline in github actions for elm app

## Initial server setup for deployment

## Building and Testing our ansible configuration and final server setup

## Setting up a Continuous Deployment Pipeline

