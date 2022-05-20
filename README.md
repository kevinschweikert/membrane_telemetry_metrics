# Membrane Telmetry Metrics

[![Hex.pm](https://img.shields.io/hexpm/v/membrane_telemetry_metrics.svg)](https://hex.pm/packages/membrane_telemetry_metrics)
[![API Docs](https://img.shields.io/badge/api-docs-yellow.svg?style=flat)](https://hexdocs.pm/membrane_telemetry_metrics)
[![CircleCI](https://circleci.com/gh/membraneframework/membrane_telemetry_metrics.svg?style=svg)](https://circleci.com/gh/membraneframework/membrane_telemetry_metrics)

This repository contains a tool for generating metrics.

It is part of [Membrane Multimedia Framework](https://membraneframework.org).

## Installation

The package can be installed by adding `membrane_telemetry_metrics` to your list of dependencies in `mix.exs`:

```elixir
def deps do
  [
    {:membrane_telemetry_metrics, github: "membraneframework/membrane_telemetry_metrics"}
  ]
end
```

## Usage

To benefit from of full functionality offered by this tool, you have to use both `Membrane.TelemetryMetrics` and `Mebrane.TelemetryMetrics.Reporter`. 

First, put 
```elixir
config :membrane_telemetry_metrics, emit_events: :all
```
in your `config.exs`. Alternatively, instead of `:all`, you can also pass there a list of emitted events, in this case, it will be `[[:shopping]]`


Let's assume, that you want to track three metrics: 
 * `cash_spent`
 * `number_of_payments`
 * `last_visited_shop`
So, the definition of our list of metrics will look like 
```elixir 
metrics = 
  [
    Telemetry.Metrics.sum(
      "cash_spent",
      event_name: [:shopping],
      measurement: :payment
    ),
    Telemetry.Metrics.counter(
      "number_of_payments",
      event_name: [:shopping]
    ),
    Telemetry.Metrics.last_value(
      "last_visited_shop",
      event_name: [:shopping],
      measurement: :shop
    )
  ]
```

Then, you have to start `Mebrane.TelemetryMetrics.Reporter`. It can be made by calling 
```elixir
  Membrane.TelemetryMetrics.Reporter.start_link([metrics: metrics], name: ShoppingReporter)
```
but the suggested way is to do it under `Supervisor` tree.

Now, `ShoppingReporter` is ready to aggregate metrics from emitted events. But before that, we have to register our event in every process, that will emit it. Let's assume, that we want to aggregate data about three people: `James Smith`, `Mary Smith` and `Patricia Johnson`. Events with data about each of these people will be emitted in a different process.

In `John Smith`'s process we will execute code
```elixir
require Membrane.TelemetryMetrics

Membrane.TelemetryMetrics.register_event_with_telemetry_metadata(
  [:shopping],
  [name: "James", surname: "Smith"]
)

Membrane.TelemetryMetrics.execute(
  [:shopping],
  %{shop: "Grocery", payment: 10},
  %{telemetry_metadata: [name: "James", surname: "Smith"]}
)

Membrane.TelemetryMetrics.execute(
  [:shopping],
  %{shop: "Bakery", payment: 5},
  %{telemetry_metadata: [name: "James", surname: "Smith"]}
)

Membrane.TelemetryMetrics.execute(
  [:shopping],
  %{shop: "Jeweller", payment: 100},
  %{telemetry_metadata: [name: "James", surname: "Smith"]}
)
```

In `Mary Smith`'s process we will execute code 
```elixir
require Membrane.TelemetryMetrics

Membrane.TelemetryMetrics.register_event_with_telemetry_metadata(
  [:shopping],
  [name: "Mary", surname: "Smith"]
)

Membrane.TelemetryMetrics.execute(
  [:shopping],
  %{shop: "Bookshop", payment: 25},
  %{telemetry_metadata: [name: "Mary", surname: "Smith"]}
)

Membrane.TelemetryMetrics.execute(
  [:shopping],
  %{shop: "Butcher", payment: 15},
  %{telemetry_metadata: [name: "Mary", surname: "Smith"]}
)
```

In `Patricia Johnson`'s process we will execute code 
```elixir
require Membrane.TelemetryMetrics

Membrane.TelemetryMetrics.register_event_with_telemetry_metadata(
  [:shopping],
  [name: "Patricia", surname: "Johnson"]
)

Membrane.TelemetryMetrics.execute(
  [:shopping],
  %{shop: "Newsagent", payment: 5},
  %{telemetry_metadata: [name: "Patricia", surname: "Johnson"]}
)
```

Then, calling 
```elixir
Membrane.TelemetryMetrics.Reporter.scrape(ShoppingReporter)
```
from anywhere on this same node, as noticed processes, will cause getting the following result:
```elixir 
%{
  {:surname, "Johnson"} => %{
    {:name, "Patricia"} => %{
      "cash_spent" => 5,
      "last_visited_shop" => "Newsagent",
      "number_of_payments" => 1
    }
  },
  {:surname, "Smith"} => %{
    {:name, "James"} => %{
      "cash_spent" => 115,
      "last_visited_shop" => "Jeweller",
      "number_of_payments" => 3
    },
    {:name, "Mary"} => %{
      "cash_spent" => 40,
      "last_visited_shop" => "Butcher",
      "number_of_payments" => 2
    }
  }
}
```

Then, for example, if `Mary Smith`'s process exits, you will get a report like
```elixir 
%{
  {:surname, "Johnson"} => %{
    {:name, "Patricia"} => %{
      "cash_spent" => 5,
      "last_visited_shop" => "Newsagent",
      "number_of_payments" => 1
    }
  },
  {:surname, "Smith"} => %{
    {:name, "James"} => %{
      "cash_spent" => 115,
      "last_visited_shop" => "Jeweller",
      "number_of_payments" => 3
    }
  }
}
```

If a process registers an event by calling
```elixir
Membrane.TelemetryMetrics.register_event_with_telemetry_metadata(event_name, telmetry_metadata)
```
and exits after it, every metric that aggregated measurements from an event with `event_name` will drop its values held for a specific value of `telemetry_metadata`, as in the example above `Mary Smith` has disappeared from the report, after the end of her's process.


## Copyright and License

Copyright 2022, [Software Mansion](https://swmansion.com/?utm_source=git&utm_medium=readme&utm_campaign=membrane_template_plugin)

[![Software Mansion](https://logo.swmansion.com/logo?color=white&variant=desktop&width=200&tag=membrane-github)](https://swmansion.com/?utm_source=git&utm_medium=readme&utm_campaign=membrane_template_plugin)

Licensed under the [Apache License, Version 2.0](LICENSE)
