# Lotus::View

A View layer for [Lotus](http://lotusrb.org).

It's based on a **separation between views and templates**.

A _view_ is an object that encapsulates the presentation logic of a page.
A _template_ is a file that defines the semantic and visual elements of a page.
In order to show a result to an user, a template must be _rendered_ by a view.

Keeping things separated, helps to declutter templates and models from presentation logic.
Also, since views are objects they are easily testable.
If you ever used [Mustache](http://mustache.github.io/), you are already aware of the advantages.

Like all the other Lotus compontents it can be used as a standalone framework, or within a full Lotus application.

## Status

[![Gem Version](https://badge.fury.io/rb/lotus-view.png)](http://badge.fury.io/rb/lotus-view)
[![Build Status](https://secure.travis-ci.org/lotus/view.png?branch=master)](http://travis-ci.org/lotus/view?branch=master)
[![Coverage](https://coveralls.io/repos/lotus/view/badge.png?branch=master)](https://coveralls.io/r/lotus/view)
[![Code Climate](https://codeclimate.com/github/lotus/view.png)](https://codeclimate.com/github/lotus/view)
[![Dependencies](https://gemnasium.com/lotus/view.png)](https://gemnasium.com/lotus/view)
[![Inline docs](http://inch-pages.github.io/github/lotus/view.png)](http://inch-pages.github.io/github/lotus/view)

## Contact

* Home page: http://lotusrb.org
* Mailing List: http://lotusrb.org/mailing-list
* API Doc: http://rdoc.info/gems/lotus-view
* Bugs/Issues: https://github.com/lotus/view/issues
* Support: http://stackoverflow.com/questions/tagged/lotus-ruby

## Rubies

__Lotus::View__ supports Ruby (MRI) 2+

## Installation

Add this line to your application's Gemfile:

    gem 'lotus-view'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install lotus-view

## Usage

### Conventions

  * Templates are searched under `Lotus::View.root`, set this value according to your app structure (eg. `"app/templates"`).
  * A view will look for a template with a file name that is composed by its full class name (eg. `"articles/index"`).
  * A template must have two concatenated extensions: one for the format one for the engine (eg. `".html.erb"`).
  * The framework must be loaded before to render for the first time: `Lotus::View.load!`.

### Views

A simple view looks like this:

```ruby
require 'lotus/view'

module Articles
  class Index
    include Lotus::View
  end
end
```

Suppose that we want to render a list of `articles`:

```ruby
require 'lotus/view'

module Articles
  class Index
    include Lotus::View
  end
end

Lotus::View.root = 'app/templates'
Lotus::View.load!

path     = Lotus::View.root.join('articles/index.html.erb')
template = Lotus::View::Template.new(path)
articles = ArticleRepository.all

Articles::Index.new(template, articles: articles).render
```

While this code is working fine, it's inefficient and verbose, because we are loading a template from the filesystem for each rendering attempt.
Also, this is strictly related to the HTML format, what if we want to manage other formats?

```ruby
require 'lotus/view'

module Articles
  class Index
    include Lotus::View
  end

  class AtomIndex < Index
    format :atom
  end
end

Lotus::View.root = 'app/templates'
Lotus::View.load!

articles = ArticleRepository.all

Articles::Index.render(format: :html, articles: articles)
  # => This will use Articles::Index
  #    and "articles/index.html.erb"

Articles::Index.render(format: :atom, articles: articles)
  # => This will use Articles::AtomIndex
  #    and "articles/index.atom.erb"

Articles::Index.render(format: :xml, articles: articles)
  # => This will raise a Lotus::View::MissingTemplateError
```

### Locals

All the objects passed in the context are called _locals_, they are available both in the view and in the template:

```ruby
require 'lotus/view'

module Articles
  class Show
    include Lotus::View

    def authors
      article.map(&:author).join ', '
    end
  end
end
```

```erb
<h1><%= article.title %></h1>
<article>
  <%= article.content %>
</article>
```

All the methods defined in the view are accessible in the template:

```erb
<h2><%= authors %></h2>
```

For convenience, they are also available as an Hash `locals` in the view.

```ruby
require 'lotus/view'

module Articles
  class Show
    include Lotus::View

    # This view, already responds to `#article` because there is an element in
    # the locals with the same key.
    #
    # In order to allow developers to override those methods, and decorate a
    # single locals object, a view has an Hash with the same values.
    #
    # If we had implemented this method like this:
    #
    #   def article
    #     ArticlePresenter.new(article)
    #   end
    #
    # We would have generated a `SystemStackError` (stack level too deep).
    def article
      ArticlePresenter.new(locals[:article])
    end
  end
end
```

### Custom rendering

Since a view is an object, you can override `#render` and provide your own rendering policy:

```ruby
require 'lotus/view'

module Articles
  class Show
    include Lotus::View
    format :json

    def render
      ArticleSerializer.new(article).to_json
    end
  end
end

Articles::Show.render({format: :json, article: article})
  # => This will render from ArticleSerializer,
  #    without the need of a template
```

### Format

The `.format` DSL is used to declare one or more mime types that a view is able to render.
These values are **arbitrary**, just **be sure to create a corresponding template**.

```ruby
require 'lotus/view'

module Articles
  class Show
    include Lotus::View
    format :custom
  end
end

Articles::Show.render({format: :custom, article: article})
  # => This will render "articles/show.custom.erb"
```

### Engines

The builtin rendering engine is [ERb](http://en.wikipedia.org/wiki/ERuby).
However, Lotus::View supports countless rendering engines out of the box.
Require your library of choice **before** of requiring `'lotus/view'`, and it will just work.

```ruby
require 'haml'
require 'lotus/view'

module Articles
  class Show
    include Lotus::View
  end
end

Articles::Show.render({format: :html, article: article})
  # => This will render "articles/show.html.haml"
```

This is the list of the supported engines.
They are listed in order of **higher precedence**, for a given extension.
For instance, if [ERubis](http://www.kuwata-lab.com/erubis/) is loaded, it will be preferred over ERb to render `.erb` templates.

<table>
  <tr>
    <th>Engine</th>
    <th>Extensions</th>
  </tr>
  <tr>
    <td>Erubis</td>
    <td>erb, rhtml, erubis</td>
  </tr>
  <tr>
    <td>ERb</td>
    <td>erb, rhtml</td>
  </tr>
  <tr>
    <td>Redcarpet</td>
    <td>markdown, mkd, md</td>
  </tr>
  <tr>
    <td>RDiscount</td>
    <td>markdown, mkd, md</td>
  </tr>
  <tr>
    <td>Kramdown</td>
    <td>markdown, mkd, md</td>
  </tr>
  <tr>
    <td>Maruku</td>
    <td>markdown, mkd, md</td>
  </tr>
  <tr>
    <td>BlueCloth</td>
    <td>markdown, mkd, md</td>
  </tr>
  <tr>
    <td>Asciidoctor</td>
    <td>ad, adoc, asciidoc</td>
  </tr>
  <tr>
    <td>Builder</td>
    <td>builder</td>
  </tr>
  <tr>
    <td>CSV</td>
    <td>rcsv</td>
  </tr>
  <tr>
    <td>CoffeeScript</td>
    <td>coffee</td>
  </tr>
  <tr>
    <td>WikiCloth</td>
    <td>wiki, mediawiki, mw</td>
  </tr>
  <tr>
    <td>Creole</td>
    <td>wiki, creole</td>
  </tr>
  <tr>
    <td>Etanni</td>
    <td>etn, etanni</td>
  </tr>
  <tr>
    <td>Haml</td>
    <td>haml</td>
  </tr>
  <tr>
    <td>Less</td>
    <td>less</td>
  </tr>
  <tr>
    <td>Liquid</td>
    <td>liquid</td>
  </tr>
  <tr>
    <td>Markaby</td>
    <td>mab</td>
  </tr>
  <tr>
    <td>Nokogiri</td>
    <td>nokogiri</td>
  </tr>
  <tr>
    <td>Plain</td>
    <td>html</td>
  </tr>
  <tr>
    <td>RDoc</td>
    <td>rdoc</td>
  </tr>
  <tr>
    <td>Radius</td>
    <td>radius</td>
  </tr>
  <tr>
    <td>RedCloth</td>
    <td>textile</td>
  </tr>
  <tr>
    <td>Sass</td>
    <td>sass</td>
  </tr>
  <tr>
    <td>Scss</td>
    <td>scss</td>
  </tr>
  <tr>
    <td>String</td>
    <td>str</td>
  </tr>
  <tr>
    <td>Yajl</td>
    <td>yajl</td>
  </tr>
</table>

### Root

Templates lookup is performed under the `Lotus::View.root` directory. Single views can specify a different path:

```ruby
class ViewWithDifferentRoot
  include Lotus::View

  root 'path/to/root'
end
```

### Template

The template file must be located under the relevant `root` and must match the class name:

```ruby
puts Lotus::View.root     # => #<Pathname:app/templates>
Articles::Index.template  # => "articles/index"
```

Each view can specify a different template:

```ruby
module Articles
  class Create
    include Lotus::View

    template 'articles/new'
  end
end

Articles::Index.template  # => "articles/new"
```

### Partials

Partials can be rendered within a template:

```erb
<%= render partial: 'articles/form', locals: { secret: 23 } %>
```

It will look for a template `articles/_form.html.erb` and it will make available both the view's and partial's locals (eg. `article` and `secret`).

### Templates

Templates can be rendered within another template:

```erb
<%= render template: 'articles/new', locals: { errors: {} } %>
```

It will render `articles/new.html.erb` and it will make available both the view's and templates's locals (eg. `article` and `errors`).

### Layouts

Layouts are wrappers for views, they can be a way to reuse common markup.

```ruby
class ApplicationLayout
  include Lotus::Layout

  def page_title
    'Title:'
  end
end

module Articles
  class Index
    include Lotus::View
    layout :application

    def page_title
      "#{ layout.page_title } articles"
    end
  end

  class RssIndex < Index
    format :rss
    layout nil
  end
end

Articles::Index.render(format: :html) # => Will use ApplicationLayout
Articles::Index.render(format: :rss)  # => Will use nothing
```

As per convention, layouts' templates are located under `Lotus::View.root` or `ApplicationLayout.root` and they uses the underscored name (eg. `ApplicationLayout => application.html.erb`).

### Presenters

They are a way to wrap and reuse presentational logic for an object.

```ruby
class Map
  attr_reader :locations

  def initialize(locations)
    @locations = locations
  end

  def location_names
    @locations.join(', ')
  end
end

class MapPresenter
  include Lotus::Presenter

  def count
    locations.count
  end

  def location_names
    super.upcase
  end

  def inspect_object
    @object.inspect
  end
end

map = Map.new(['Rome', 'Boston'])
presenter = MapPresenter.new(map)

# access a map method
puts presenter.locations # => ['Rome', 'Boston']

# access presenter concrete methods
puts presenter.count # => 1

# uses super to access original object implementation
puts presenter.location_names # => 'ROME, BOSTON'

# it has private access to the original object
puts presenter.inspect_object # => #<Map:0x007fdeada0b2f0 @locations=["Rome", "Boston"]>
```

### Thread safety

**Lotus::View**'s is thread safe during the runtime, but it isn't during the loading process.
Please load the framework as the last thing before your application starts.
Also, be sure that your app provides a thread safe context while it's loaded.


```ruby
Mutex.new.synchronize do
  Lotus::View.load!
end
```

After this operation, all the class variables are frozen, in order to prevent accidental modifications at the run time.

**This is not necessary, when Lotus::View is used within a Lotus application.**

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

## Copyright

Copyright 2014 Luca Guidi – Released under MIT License
