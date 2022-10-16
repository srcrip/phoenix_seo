<!-- badges -->

[![Hex.pm Version](http://img.shields.io/hexpm/v/phoenix_seo.svg)](https://hex.pm/packages/phoenix_seo)
[![Hex docs](http://img.shields.io/badge/hex.pm-docs-blue.svg?style=flat)](https://hexdocs.pm/phoenix_seo)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE.md)

# SEO

![logo](./priv/logo.png)

<!-- MDOC !-->

**WORK IN PROGRESS, DO NOT USE**

```
/ˈin(t)ərˌnet jo͞os/
noun: internet juice
```

SEO (Search Engine Optimization) provides a framework for Phoenix applications
to more-easily optimize your site for search engines and displaying rich results
when your URLs are shared across the internet. The better visibility your pages
have in search results, the more likely you are to have visitors.

## Installation

```elixir
def deps do
  [
    {:phoenix_seo, "~> 0.1.4"}
  ]
end
```

## Usage

1. Define an SEO module for your web application and defaults

```elixir
defmodule MyAppWeb.SEO do
  use SEO, [
    json_library: Jason,
    site: SEO.Site.build(
      default_title: "Default Title",
      description: "A blog about development",
      title_suffix: " · My App"
    ),
    open_graph: SEO.OpenGraph.build(
      description: "A blog about development",
      site_name: "My Blog",
      type: :website,
      locale: "en_US"
    ),
    twitter: SEO.Twitter.build(
      site: "@example",
      site_id: "27704724",
      creator: "@example",
      creator_id: "27704724",
      card: :summary
    )
  ]
end
```

2. Implement functions to build SEO information about your entities

```elixir
defmodule MyApp.Article do
  # This might be an Ecto schema or a plain struct
  defstruct [:id, :title, :description, :author, :reading, :published_at]
end

defimpl SEO.OpenGraph.Build, for: MyApp.Article do
  alias MyAppWeb.Router.Helpers, as: Routes
  @endpoint MyAppWeb.Endpoint

  def build(article) do
    SEO.OpenGraph.build(
      type: :article,
      type_detail:
        SEO.OpenGraph.Article.build(
          published_time: article.published_at |> DateTime.to_date() |> Date.to_iso8601(),
          author: article.author,
          section: "Tech"
        ),
      image: image(article),
      title: article.title,
      description: article.description
    )
  end

  defp image(article) do
    file = "/images/article/#{article.id}.png"

    exists? =
      [Application.app_dir(:my_app), "/priv/static", file]
      |> Path.join()
      |> File.exists?()

    if exists? do
      SEO.OpenGraph.Image.build(
        url: Routes.static_url(@endpoint, file),
        secure_url: Routes.static_url(@endpoint, file),
        alt: article.title
      )
    end
  end
end

defimpl SEO.Site.Build, for: MyApp.Article do
  alias MyAppWeb.Router.Helpers, as: Routes
  @endpoint MyAppWeb.Endpoint

  def build(article) do
    SEO.Site.build(
      url: Routes.article_url(@endpoint, :show, article.id),
      title: article.title,
      description: article.description
    )
  end
end

defimpl SEO.Facebook.Build, for: MyApp.Article do
  def build(_article) do
    SEO.Facebook.build(app_id: "123")
  end
end

defimpl SEO.Twitter.Build, for: MyApp.Article do
  def build(article) do
    SEO.Twitter.build(description: article.description, title: article.title)
  end
end

defimpl SEO.Unfurl.Build, for: MyApp.Article do
  def build(article) do
    SEO.Unfurl.build(
      label1: "Reading Time",
      data1: "5 minutes",
      label2: "Published",
      data2: DateTime.to_iso8601(article.published_at)
    )
  end
end

defimpl SEO.Breadcrumb.Build, for: MyApp.Article do
  alias MyAppWeb.Router.Helpers, as: Routes
  @endpoint MyAppWeb.Endpoint

  def build(article) do
    SEO.Breadcrumb.List.build([
      %{name: "Articles", item: Routes.article_url(@endpoint, :index),
      %{name: article.title, item: Routes.article_url(@endpoint, :show, article.id)
    ])
  end
end
```

3. Assign the item to your conns and/or sockets

```elixir
# In a plain Phoenix Controller
def show(conn, params) do
  article = load_article(params)

  conn
  |> SEO.assign(article)
  |> render("show.html")
end

def index(conn, params) do
  conn
  |> SEO.assign(%{title: "Listing Best Hugs"})
  |> render("show.html")
end

# In a Phoenix LiveView, make sure you handle with
# mount/3 or handle_params/3 so it's present on
# first static render.
def mount(_params, _session, socket) do
  # You may mark it as temporary since it's only needed on the first render.
  {:ok, socket, temporary_assigns: [{SEO.key(), nil}]}
end

def handle_params(params, _uri, socket) do
  {:noreply, SEO.assign(socket, load_article(params))}
end
```

4. Juice up your root layout:

```heex
<head>
  <%# remove the Phoenix-generated <.live_title> component %>
  <%# and replace with SEO.juice component %>
  <SEO.juice
    config={MyAppWeb.SEO.config()}
    item={SEO.item(@conn)}
    page_title={assigns[:page_title]}
  />
</head>
```

Alternatively, you may selectively render components. For example:

```heex
<head>
  <%# With your SEO module's configuration %>
  <SEO.OpenGraph.meta
    config={MyAppWeb.SEO.config(:open_graph)}
    item={SEO.OpenGraph.Build.build(SEO.item(assigns))}
  />

  <%# Or with some other default configuration %>
  <SEO.OpenGraph.meta
    config={[default_title: "Foo Fighters"]}
    item={SEO.OpenGraph.Build.build(SEO.item(assigns))}
  />

  <%# Or without defaults %>
  <SEO.OpenGraph.meta item={SEO.OpenGraph.Build.build(SEO.item(assigns))} />
</head>
```
