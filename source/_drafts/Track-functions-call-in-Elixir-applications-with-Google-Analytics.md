---
title: Decorate functions using macros in Elixir
tags:
  - elixir
  - macro
published: true
---
Recently I decided to make public a telegram bot I did to monitor bus time in Dublin ([@dublin_bus_bot](https://telegram.me/dublin_bus_bot)). Before the release I became curious to see how many people will use it (spoiler: just an handful) and I thought that would be nice to track the use on google analytics.

### Overview

Google analytics provide a measurement protocol that can be used to track things that are different from websites (mobile apps, IOT). At the moment no elixir client exists for this protocol (and it would not be anything more than an api wrapper). My plan is to make call to the Ga TK endpoint with httpoison but I'd prefer to not have to call the tracking function for every single bot command.

One of the feature that I prefer of the elixir are macros, macros allow to generate code at compile time. My plan is to define a macro that looking like a function definition would define a function with the same body and with an additional call to the track function. I decided this approach because seems more idiomatics than try to use decorator syntax typical of other languages (@decorator at least in python and javascript)
``` elixir
defmeter sample_function(arg1, arg2) do
    IO.inspect([arg1, arg2])
end

# would generate something similar to

def sample_function(arg1, arg2) do
    track(:sample_function, [arg1: arg1, arg2: arg2])
    IO.inspect([arg1, arg2])
end

```

### Implementation

The first thing to do is to define a macro and extract the function definition (function name, and function arguments names).

``` elixir
  # A macro definitio can use pattern matching to destructure the arguments as
  # is possible with a normal function
  defmacro defmeter({function,_,args} = fundef, [do: body]) do
    # arguments are defined in 3 elements tuples
    # this extract the arguments names in a list
    names = Enum.map(args, &elem(&1, 0))

    # meter will contain the body of the function that will be defined by the macro
    metered = quote do
      # quote and unquote allow to switch context,
      # simplyfing a lot quoted code will run when the function is called
      # unquoted code run at compile time (when the macro is called)
      values = unquote(
        args
        |> Enum.map(fn arg ->  quote do
            # allow to access a value ad runtime knowing the name
            # elixir macros are hygienic so it's necessary to mark it
            # explicitly
            var!(unquote(arg))
          end
        end)
      )

      # Match argument names with their own values at call time
      map = Enum.zip(unquote(names), values)

      # wrap the original function call with a try to track errors too
      try do
        to_return = unquote(body)
        track(unquote(function), map)
        to_return
      rescue
        e ->
          track_error(unquote(function), map, e)
          raise e
      end
    end

    # define a function with the same name and arguments and with the augmented body
    quote do
      def(unquote(fundef),unquote([do: metered]))
    end
  end
```
