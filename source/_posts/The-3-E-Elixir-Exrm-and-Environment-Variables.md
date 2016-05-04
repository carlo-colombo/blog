---
title: 'The 3 E: Elixir, Exrm, and Environment variables'
tags:
  - elixir
  - conform
  - exrm
date: 2016-05-04 21:05:43
---


### Intro
I’m building a bot for Telegram, once make a build with exrm I found myself some problem configuring the telegram api key using environment variables. I decided to share what I found because my google foo was not helpful at all.

### TL;DR
To configure en elixir application built with [exrm](https://github.com/bitwalker/exrm) use [conform](https://github.com/bitwalker/conform) and load the environment variable in the trasforms section of the conform schema.

## config.exs

config.exs is where the configuration of elixir project are added. The file is interpreted when a project is ran with `ies -S mix` or `mix run`.

The values are retrieved during the lifetime of the application with the functions([get_env/3](http://elixir-lang.org/docs/stable/elixir/Application.html#get_env/3), [fetch_env/2](http://elixir-lang.org/docs/stable/elixir/Application.html#fetch_env/2), [fetch_env!/2](http://elixir-lang.org/docs/stable/elixir/Application.html#fetch_env!/2)) available in the module Application. To include values from the environment where the project is running [System.get_env/1](http://elixir-lang.org/docs/stable/elixir/System#get_env/1) is used.

``` elixir
use Mix.Config

config :plug,
  key1: "value1",
  key2: System.get_env("KEY")

import_config "#{Mix.env}.exs"
```

## exrm
Surprisingly (if you did not have an erlang background as me) when you make a release of your project with exrm the `config.exs` file is executed at build time and the environment variables are crystallized in the build output.

The output from exrm contains a file named `sys.config` that is the output of executing the `config.exs` file and is defined as erlang terms. Once released editing this file is the only way to dynamically configure the application once built.

## conform
[conform](https://github.com/bitwalker/conform) is a library from the same author of exrm and is been made to ease the configuration of elixir application. The library validate a property like file (`configuration/your_app.conf`) against a configuration schema (`configuration/your_app.schema.exs`) and generate the `sys.config` file. The schema file contains descriptions, defaults, and types of the parameters. A property file is a lot easier and common to configure than a file containing erlang terms, additionaly conform add flexibility and more control over configurations.

A couple of task are made available to transition to a conform based configuration.

``` elixir
#generate the schema from the actual configuration
mix conform.new

#generate a .conf file from the configuration
mix conform.configure
```


The `sys.config` is generated at the start of the application, and using the plugin `exrm_conform` at the start of a packaged application. This behaviour allow to load configuration parameter from environment variables defined when the application is started.

After running `mix conform.new` you will find a `yourapp.schema.exs` in your `conf` folder, this file has 3 main sections: mapping, transforms, and validators. The mapping section is where the parameters are defined and where you can set up defaults, descriptions, and type. The transform section allow to add transformation function to change or derive a configured parameter. In the end the validators sections allow to reject invalid configuration errors and stop the application.

At first I tried to add the reading from the environment variable to the default of the parameters, but this lend to an uncommon situation where a static parameter (the one in `yourapp.conf`) will override a parameter derived from an environment variable.

Eventually I found that adding a function to transforms is probably a better way to do it.

``` elixir
[
  extends: [],
  import: [],
  mappings: [...],
  transforms: [
    "nadia.token": fn conf ->
      [{_, val}] = Conform.Conf.get(conf, "nadia.token")
      System.get_env("TELEGRAM_BOT_TOKEN") || val
    end],
  validators: []
]
```

In this case we are loading with the conform API the the configured value and return it only if the environment variable is empty. Generating the parameter in this way disallow to have a default in the mapping sections but a workaround would be to chain `||` to add a default value. I think that this approach is not bad for api key and similar values that you don't want to checkin with your code (even if are keys of development environments).
