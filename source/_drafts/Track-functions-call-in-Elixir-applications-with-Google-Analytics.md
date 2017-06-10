---
title: Decorate functions using macros in Elixir
tags:
  - elixir
  - macro
  - google analytics
published: true
---
After I decided to make public a telegram bot to monitor bus time in Dublin ([@dublin_bus_bot](https://telegram.me/dublin_bus_bot)). Before the release I became curious to see how many people will use it (spoiler: just an handful) and I thought that would be a good idea to track the use on google analytics.

### Overview

Google analytics provide a measurement protocol that can be used to track things that are different from websites (mobile apps, IOT). At the moment no elixir client exists for this protocol (and it would not be anything more than an api wrapper). My plan is to make call to the Google Analytics TK endpoint with [HTTPOison](https://github.com/edgurgel/httpoison) but I'd prefer to not have to call the tracking function for every single bot command.

One of the feature that I prefer of the elixir are macros, macros allow to generate code at compile time. I decided to define a macro that looking like a function definition would define a function with the same body and with an additional call to the track function. I decided this approach because seems more idiomatics than using the [decorator syntax](https://github.com/arjan/decorator) typical of other languages (`@decorator` at least in python and javascript).

```elixir
defmetered sample_function(arg1, arg2) do
    IO.inspect([arg1, arg2])
end

# would generate something similar to

def sample_function(arg1, arg2) do
    track(:sample_function, [arg1: arg1, arg2: arg2])
    IO.inspect([arg1, arg2])
end

```

### Implementation

I implemented this approach in [meter](https://hex.pm/packages/meter) to use in the telegram bot I wrote.

```elixir
  @doc """
  Replace a function definition, automatically tracking every call to the function
  on google analytics. It also track exception with the function track_error.
  This macro intended use is with a set of uniform functions that can be concettualy
  mapped to pageviews (eg: messaging bot commands).
  
  Example:
  
      defmetered function(arg1, arg2), do: IO.inspect({arg1,arg2})
  
      function(1,2)
      
  will call track with this parameters
      
      track(:function, [arg1: 1, arg2: 2])
  
  Additional parameters will be loaded from the configurationd
  """
  # A macro definition can use pattern matching to destructure the arguments
  defmacro defmetered({function,_,args} = fundef, [do: body]) do
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
            # allow to access a value at runtime knowing the name
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

### Conclusion

Elixir macros are a powerful tool to abstract away some functionality or to write DSLs. They require a bit of time to wrap head around, in particular with the context swith, but it totally worth the hassle if you can reduce the clutter in your code base.

