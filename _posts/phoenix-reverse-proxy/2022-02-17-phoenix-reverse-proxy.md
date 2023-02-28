---
layout: post
title:  "A look into routing traffic into your umbrella and a deep-dive into the PhoenixReverseProxy"
date:   2022-02-17 01:23:45 +0500
categories: reverse-proxy umbrella
---

# Problem
When Phoenix projects grow, an umbrella structure is needed to separate different services. Once the application is refactored to fit inside an umbrella structure, and a new Phoenix app is added, the developer is immediately faced with the issue that the same port cannot be used for both services. A way of routing traffic is needed from one HTTP(S) port to the different Phoenix applications.

# Solutions
There are several solutions, each with different tradeoffs.

Here are some that I came up with:

1. Setup a reverse proxy with a popular HTTP daemon like NGINX or Apache2 and use different ports for each app.
1. Set up a load balancer to forward to different ports for each app.
1. Separate the applications into different repos, deployments and releases (this approach is suitable for large teams of 100+ developers).
1. Use an internal Phoenix reverse proxy.

Let's look at each option in more detail.

## Load Balancer or Reverse Proxy
Both these options require DevOps intervention or modifications to DevOps code or configs. Not everyone knows how to configure a load balancer or NGINX. Keeping the overhead of creating a new Phoenix application low is critical; otherwise, developers will avoid it and instead build up a monolith. There is also an issue with performance depending on the application because additional IO needs to be performed to proxy the requests.

## Separate Releases or Separate Repos
If a company has reached a level of scale where it can afford the overhead of deploying and maintaining multiple services, DevOps will need to be involved. This opens up the possibility of doing everything at the DevOps layer.

Eventually, this level of separation is what we should be striving for as an end goal when we divide up our applications in our umbrella. But it doesn't mean we need to do everything in the DevOps layer because individual services still, in many cases, need an umbrella structure involving more than one application; therefore, internal proxying is not ruled out for services that require that level of complexity. So a hybrid approach of separate repos and releases where some are umbrellas with internal proxying is likely. 

## Internal Reverse Proxy
There are currently three reverse proxy strategies that exist in the ecosystem:

1. [Roll your own](https://blog.appsignal.com/2019/04/16/elixir-alchemy-routing-phoenix-umbrella-apps.html)
1. [MasterProxy](https://github.com/jesseshieh/master_proxy)
1. [PhoenixReverseProxy](https://github.com/loopsocial/phoenix_reverse_proxy)

### Rolling your own Reverse Proxy
While this can be done, it is not recommended because it does not consider WebSockets. So unless it is reasonably certain that WebSockets will never be needed in your application, use one of the other solutions. When the details of the PhoenixReverseProxy internals are shown later, it will become clear why this is the case.

### MasterProxy
This was the first solution available that didn't require rolling our own. So it looked very promising. Unfortunately, it is not suitable for a couple of reasons. The biggest reason is that its implementation is tied to cowboy 2.x, which means if the HTTP server changes for Phoenix, a new implementation will be required. Having seen many HTTP server implementations change over the years, I wanted to avoid going down that path and doing the work twice. The other reason is that it used Regexes for relatively slow matching, especially as the number of Regexes grows since it has to match them sequentially!

Here is what a config looks like:
```elixir
config :master_proxy,
  http: [port: 80],
  backends: [
    %{
      host: ~r{^app-name\.gigalixirapp\.com$},
      phoenix_endpoint: MyAppWeb.Endpoint
    },
    %{
      host: ~r{^www\.example\.com$},
      phoenix_endpoint: MyAppWeb.Endpoint
    },
    %{
      host: ~r{^api\.example\.com$},
      phoenix_endpoint: MyAppApiWeb.Endpoint
    },
    %{
      host: ~r{^members\.example\.com$},
      phoenix_endpoint: MyAppMembersWeb.Endpoint
    }
  ]
```

### PhoenixReverseProxy
Going into this, the design goals were:
1. Keep everything as simple as possible.
1. Use Elixir pattern matching for host and path to have similar performance characteristics as the [`Phoenix.Router`](https://hexdocs.pm/phoenix/Phoenix.Router.html).
1. No coupling to the HTTP(S) implementation.
1. Allow routing of WebSockets.

Here is what the config looks like:
```elixir
defmodule ReverseProxyWeb.Endpoint do
  use PhoenixReverseProxy, otp_app: :reverse_proxy

  # IMPORTANT: All of these macros except for proxy_default/1
  #            and proxy_path/1 can take a path prefix, so
  #            they all have an arity of 2 and 3.

  # Maps to http(s)://api.example.com/v1
  proxy("api.example.com", "v1", ExampleApiV1.Endpoint)

  # Maps to http(s)://api.example.com/v2
  proxy("api.example.com", "v2", ExampleApiV2.Endpoint)

  # Matches the domain only and no subdomains
  proxy("example.com", ExampleWeb.Endpoint)
  # Matched any subdomain such as http(s)://images.example.com/
  # but not the domain itself http(s)://example.com/
  proxy_subdomains("example.com", ExampleSubs.Endpoint)

  # Matches all subdomains and the domain itself.
  # This is equivalent to combining these rules:
  #   proxy("foofoovalve.com", FoofooValve.Endpoint)
  #   proxy_subdomains("foofoovalve.com", FoofooValve.Endpoint)
  proxy_all("foofoovalve.com", FoofooValve.Endpoint)
  
  # Matches path /auth/ for any domain
  proxy_path("auth", Auth.Endpoint)
  
  # Matches anything not matched above
  proxy_default(ExampleWeb.Endpoint)
end
```

# Internals of PhoenixReverseProxy

## Plug init/1 and call/2 Routing
So how does all of this work under the hood? Let's start with the basics of how an [`Phoenix.Endpoint`](https://hexdocs.pm/phoenix/Phoenix.Endpoint.html) can forward to another in this example taken from [this](https://blog.appsignal.com/2019/04/16/elixir-alchemy-routing-phoenix-umbrella-apps.html) blog post.
```elixir
defmodule Proxy.Endpoint do
  use Phoenix.Endpoint, otp_app: :proxy
 
  @base_host_regex ~r|\.?mydomain.*$|
  @subdomains %{
    "admin" => Admin.Web.Endpoint,
    "client" => Client.Web.Endpoint
  }
  @default_host Client.Web.Endpoint
 
  def init(opts), do: opts
 
  def call(conn, _) do
    with subdomain <- String.replace(conn.host, @base_host_regex, ""),
         endpoint <- Map.get(@subdomains, subdomain, @default_host)
    do
      endpoint.call(conn, endpoint.init())
    end
  end
end
```

Here two callbacks have been implemented that looks suspiciously like a [`Plug`](https://hexdocs.pm/plug/readme.html) because it is! They use `conn.host` and `conn.path_info` to decide what `Phoenix.Endpoint` to route the request to. Writing this code every time is undesirable. `PhoenixReverseProxy` does it as a macro just like Phoenix does. It ends up like a slightly more complex version of this:

```elixir
  defmacro __using__(opts) do
    quote [{:location, :keep}, :generated] do
      Module.register_attribute(__MODULE__, :reverse_proxy_routes, accumulate: true)
      Module.register_attribute(__MODULE__, :default_reverse_proxy_endpoint, accumulate: false)

      @impl Plug
      def init(opts) do
        opts
      end

      @impl Plug
      def call(conn, opts) do
        matching_endpoint = match_endpoint(conn.host, conn.path_info)
        matching_endpoint.call(conn, matching_endpoint.init(opts))
      end

      import unquote(PhoenixReverseProxy)
      use Phoenix.Endpoint, unquote(opts)
    end
  end
  ```

Do not fear! Everything will be explained! 

## Accumulating Routes Phoenix Style
First off, what are module attributes? Well, most Elixir developers are already using them:
```elixir
defmodule M do
	@x 1
end
```
This sets the attribute `:x` to value `1`. This is syntactic sugar for:
```elixir
defmodule M do
	Module.register_attribute(__MODULE__, :x, accumulate: false)
	Module.put_attribute(__MODULE__, :x, 1)
end
```

There is an option called `accumulate` which means that an attribute can be set multiple times and it will accumulate in a list. Insertion at the head of a list being more efficient for performance means the list will be in the reverse order of the order in which the attributes were specified. Here is an example in `iex`:
```elixir
iex(1)> defmodule M do
...(1)>   Module.register_attribute(__MODULE__, :x, accumulate: true)
...(1)>   @x 1
...(1)>   @x 2
...(1)>   @x 3
...(1)>
...(1)>   def get_x, do: @x
...(1)> end
{:module, M,
 <<70, 79, 82, ...>>, {:get_x, 0}}
iex(2)> M.get_x
[3, 2, 1] # In the reverse order
```

If the attributes are desired in the order specified in the module, the list needs to be reversed. The attribute accumulator is used with macros to create a list of routing rules. Only one of the simplest macros is included here to make it easy to understand:
```elixir
  defmacro proxy(hostname, endpoint) do
    quote do
      @reverse_proxy_routes {
        unquote(endpoint),
        unquote(hostname)
      }
    end
  end
```
When this macro is called in our `PhoenixReverseProxy` which is a [`Phoenix.Endpoint`](https://hexdocs.pm/phoenix/Phoenix.Endpoint.html) module, it will add an item to the `PhoenixReverseProxy` routing table.

## Harnessing BEAM Pattern Matching Performance

To harness the BEAM pattern matching and get high performance, a generated function that efficiently uses pattern matching is utilised to route the correct [`Phoenix.Endpoint`](https://hexdocs.pm/phoenix/Phoenix.Endpoint.html).

For simplicity, here is a simplified example where the routes are statically defined and where matching functions are generated to match a domain (the real implementation uses accumulated attributes):
```elixir
defmodule DomainMatchingExample do
  @moduledoc """
  Proxy mappings from hostname to Phoenix.Endpoint
  """
  @default_route Default.Endpoint
  @routes [
	  {"a.com", A.Endpoint},
	  {"b.com", B.Endpoint},
	  {"c.com", C.Endpoint},
	  {"d.com", D.Endpoint},
  ]
  
  for {domain, endpoint} <- Enum.reverse(@routes) do
	def match_domain(unquote(domain)) do
		unquote(endpoint)
	end
  end
  def match_domain(_other) do
	@default_route
  end
end
```
The code above will generate a function `match_domain/1`, which can match faster than *O(n)* thanks for the magic inside the BEAM. So as we add rules, the cost is not linear.

Now an issue arises if we want to match subdomains because we want to match a prefix wildcard, and the BEAM does not permit prefix wildcard matching. How can we resolve this?

The best solution is to reverse the string before matching and then match it against the reversed string.

So say we want to match `*.firework.com`, and we want both domain and subdomains. We would need first to reverse the domain:

```elixir
iex(1)> "firework.com" |> String.reverse()
"moc.krowerif"
```
And match both `"moc.krowerif." <> _` and `"moc.krowerif"`.

```elixir
defmodule DomainMatchingExample do
  @moduledoc """
  Proxy mappings from hostname to Phoenix.Endpoint
  """
  @default_route Default.Endpoint
  @routes [
	  {"a.com", A.Endpoint},
	  {"b.com", B.Endpoint},
	  {"c.com", C.Endpoint},
	  {"d.com", D.Endpoint},
  ]
  for {domain, endpoint} <- Enum.reverse(@routes) do
	def match_reversed_domain(unquote(String.reverse(domain))) do
		unquote(endpoint)
	end
	def match_reversed_domain(unquote(String.reverse(domain) <> ".") <> _) do
		unquote(endpoint)
	end
  end
  def match_reversed_domain(_other) do
	@default_route
  end
end
```

Here are a few benchmarks to prove that BEAM pattern matching beats a list of regular expressions:
```elixir
defmodule PrefixPatternBench do
  @moduledoc """
  Documentation for `PrefixPatternBench`.
  """

  random_strings = 10_000..20_000 |> Enum.map(&Integer.to_string/1) |> Enum.shuffle()
  random_regexes = for random_string <- random_strings, do:  ~r/^#{random_string}/
  Module.register_attribute(__MODULE__, :random_regexes, accumulate: false)
  Module.put_attribute(__MODULE__, :random_regexes, random_regexes)

  for random_string <- random_strings do
    def beam_matching(unquote(random_string) <> _), do: unquote(random_string)
  end
  def beam_matching(_) do
    :not_found
  end

  def regex_matching(s) do
    @random_regexes |> Enum.find(:not_found, &Regex.match?(&1, s))
  end

end
```
And then we try to match from a shuffled list of 10000 prefixes and measure how long it took:
```elixir
iex(1)> defmodule TimeFrame do
...(1)>   defmacro execute(name, units \\ :microsecond, do: yield) do
...(1)>     quote do
...(1)>       start = System.monotonic_time(unquote(units))
...(1)>       result = unquote(yield)
...(1)>       time_spent = System.monotonic_time(unquote(units)) - start
...(1)>       IO.puts("Executed #{unquote(name)} in #{time_spent} #{unquote(units)}")
...(1)>       result
...(1)>     end
...(1)>   end
...(1)> end
{:module, TimeFrame,
 <<70, 79, 82,  ...>>, {:execute, 3}}
iex(3)> require TimeFrame
TimeFrame
iex(4)> TimeFrame.execute "regex 10000" do
...(4)>   PrefixPatternBench.regex_matching "12345"
...(4)> end
Executed regex 10000 in 13685 microsecond
~r/^12345/
iex(5)> TimeFrame.execute "regex 10000" do         
...(5)>   PrefixPatternBench.regex_matching "13333"
...(5)> end
Executed regex 10000 in 2216 microsecond
~r/^13333/
iex(6)> TimeFrame.execute "regex 10000" do         
...(6)>   PrefixPatternBench.regex_matching "14444"
...(6)> end
Executed regex 10000 in 8697 microsecond
~r/^14444/
iex(7)> TimeFrame.execute "beam 10000" do          
...(7)>   PrefixPatternBench.beam_matching "12345" 
...(7)> end
Executed beam 10000 in 412 microsecond
"12345"
iex(8)> TimeFrame.execute "beam 10000" do         
...(8)>   PrefixPatternBench.beam_matching "13333"
...(8)> end                                       
Executed beam 10000 in 18 microsecond
"13333"
iex(9)> TimeFrame.execute "beam 10000" do         
...(9)>   PrefixPatternBench.beam_matching "14444" 
...(9)> end
Executed beam 10000 in 17 microsecond
"14444"
```

BEAM averages 149 microseconds on three samples with 10000 routes, and a regular expression lists 8199 microseconds on three samples with 10000 routes. This is a speedup of ~55x for 10000 routes and around ~10x for 100 routes.

## Reversing Domains Fast

Now that the domain and subdomain can be matched in a way that scales to a large number of rules, the domain string needs to be reversed before matching. But one might ask, aren't operations like `String.reverse/1` pretty expensive due to Unicode? As it turns out, this is true. But this can be worked around because internationalised domain names use a particular encoding that translates directly to ASCII. For example, `點看.com` becomes `xn--c1yn36f.com`, so the bytes can be reversed instead to get much better performance. How are bytes reversed? Like this:
```elixir
defmodule ReverseDomainExample do
  @doc ~S"""
  Reverse a domain name string (ASCII). This is used internally for pattern
  matching of subdomains.
  ## Examples
      iex> ReverseDomainExample.reverse_domain("abc.com")
      "moc.cba"
  """
  def reverse_domain(domain) do
    domain |> :binary.decode_unsigned(:little) |> :binary.encode_unsigned(:big)
  end
end
```

How much faster is it?
```elixir
iex(5)> domain = "firework.com"
"firework.com"
iex(6)> TimeFrame.execute "String.reverse/1" do
...(6)>   String.reverse(domain)
...(6)> end
Executed String.reverse/1 in 63 microsecond
"moc.krowerif"
iex(7)> TimeFrame.execute "endian binary reverse" do
...(7)>   domain 
...(7)> 	|> :binary.decode_unsigned(:little)
...(7)>		|> :binary.encode_unsigned(:big)
...(7)> end
Executed endian binary reverse in 20 microsecond
"moc.krowerif"
```
About ~3x the performance.

## Routing WebSockets
How does [`Phoenix.Endpoint`](https://hexdocs.pm/phoenix/Phoenix.Endpoint.html) route sockets in the endpoint? The first hint is that it uses a macro [`socket/3`](https://hexdocs.pm/phoenix/Phoenix.Endpoint.html#socket/3) and as you've probably guessed already, it uses module attributes to accumulate them. Here is the actual [code](https://github.com/phoenixframework/phoenix/blob/aa9e708fec303f1114b9aa9c41a32a3f72c8a06c/lib/phoenix/endpoint.ex#L910):
```elixir
  defmacro socket(path, module, opts \\ []) do
    module = Macro.expand(module, %{__CALLER__ | function: {:__handler__, 2}})

    quote do
      @phoenix_sockets {unquote(path), unquote(module), unquote(opts)}
    end
  end
  ```
Higher up in the file:
```elixir
Module.register_attribute(__MODULE__,  :phoenix_sockets,  accumulate: true)
```

The socket configurations are then made available to other parts of Phoenix using this [function](https://github.com/phoenixframework/phoenix/blob/aa9e708fec303f1114b9aa9c41a32a3f72c8a06c/lib/phoenix/endpoint.ex#L669):

```elixir
sockets = Module.get_attribute(module, :phoenix_sockets)
...
def  __sockets__,  do: unquote(Macro.escape(sockets))
```

For the reverse proxy, the values of all sockets in the system must be collected and exposed via a `__sockets__/0` function. Failing to do so, we find that  WebSockets do not work.

We do this in the `__before_compile__/1` macro, and once all sockets are verified for collisions, we then set the module attribute for the proxy:
```elixir
Module.put_attribute(unquote(env.module), :phoenix_sockets, phoenix_socket)
 ```
 
 That's all, folks! There is more in there, but the primary concern is to handle path matching and validation.

# Conclusion
In conclusion, don't write your own. If you want the fastest solution today, both from a developer and from a runtime performance point of view, `PhoenixReverseProxy` is by far the fastest.
