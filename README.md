# SyntaxSuggest

An error in your code forces you to stop. SyntaxSuggest helps you find those errors to get you back on your way faster.

```
Unmatched `end', missing keyword (`do', `def`, `if`, etc.) ?

  1  class Dog
> 2    defbark
> 4    end
  5  end
```

> This project was previously named `dead_end` as it attempted to find dangling `end` keywords. The name was changed to be more descriptive and welcoming as part of the effort to merge it into [Ruby 3.2](https://bugs.ruby-lang.org/issues/18159#note-29).

## Installation in your codebase

To automatically annotate errors when they happen, add this to your Gemfile:

```ruby
gem 'syntax_suggest'
```

And then execute:

    $ bundle install

If your application is not calling `Bundler.require` then you must manually add a require:

```ruby
require "syntax_suggest"
```

If you're using rspec add this to your `.rspec` file:

```
--require syntax_suggest
```

> This is needed because people can execute a single test file via `bundle exec rspec path/to/file_spec.rb` and if that file has a syntax error, it won't load `spec_helper.rb` to trigger any requires.

## Install the CLI

To get the CLI and manually search for syntax errors (but not automatically annotate them), you can manually install the gem:

    $ gem install syntax_suggest

This gives you the CLI command `$ syntax_suggest` for more info run `$ syntax_suggest --help`.

## Editor integration

An extension is available for VSCode:

- Extension: https://marketplace.visualstudio.com/items?itemName=Zombocom.dead-end-vscode
- GitHub: https://github.com/zombocom/dead_end-vscode

## What syntax errors does it handle?

Syntax suggest will fire against all syntax errors and can isolate any syntax error. In addition, syntax_suggest attempts to produce human readable descriptions of what needs to be done to resolve the issue. For example:

- Missing `end`:

<!--
```ruby
class Dog
  def bark
    puts "bark"
end
```
-->

```
Unmatched keyword, missing `end' ?

> 1  class Dog
> 2    def bark
> 4  end
```

- Missing keyword
<!--
```ruby
class Dog
  def speak
    @sounds.each |sound|
      puts sound
    end
  end
end
```
-->

```
Unmatched `end', missing keyword (`do', `def`, `if`, etc.) ?

  1  class Dog
  2    def speak
> 3      @sounds.each |sound|
> 5      end
  6    end
  7  end
```

- Missing pair characters (like `{}`, `[]`, `()` , or `|<var>|`)
<!--

```ruby
class Dog
  def speak(sound
    puts sound
  end
end
```
-->

```
Unmatched `(', missing `)' ?

  1  class Dog
> 2    def speak(sound
> 4    end
  5  end
```

- Any ambiguous or unknown errors will be annotated by the original ripper error output:

<!--
class Dog
  def meals_last_month
    puts 3 *
  end
end
-->

```
syntax error, unexpected end-of-input

  1  class Dog
  2    def meals_last_month
> 3      puts 3 *
  4    end
  5  end
```

## How is it better than `ruby -wc`?

Ruby allows you to syntax check a file with warnings using `ruby -wc`. This emits a parser error instead of a human focused error. Ruby's parse errors attempt to narrow down the location and can tell you if there is a glaring indentation error involving `end`.

The `syntax_suggest` algorithm doesn't just guess at the location of syntax errors, it re-parses the document to prove that it captured them.

This library focuses on the human side of syntax errors. It cares less about why the document could not be parsed (computer problem) and more on what the programmer needs (human problem) to fix the problem.

## Sounds cool, but why isn't this baked into Ruby directly?

We are now talking about it https://bugs.ruby-lang.org/issues/18159#change-93682.

## Artificial Inteligence?

This library uses a goal-seeking algorithm for syntax error detection similar to that of a path-finding search. For more information [read the blog post about how it works under the hood](https://schneems.com/2020/12/01/squash-unexpectedend-errors-with-syntaxsearch/).

## How does it detect syntax error locations?

We know that source code that does not contain a syntax error can be parsed. We also know that code with a syntax error contains both valid code and invalid code. If you remove the invalid code, then we can programatically determine that the code we removed contained a syntax error. We can do this detection by generating small code blocks and searching for which blocks need to be removed to generate valid source code.

Since there can be multiple syntax errors in a document it's not good enough to check individual code blocks, we've got to check multiple at the same time. We will keep creating and adding new blocks to our search until we detect that our "frontier" (which contains all of our blocks) contains the syntax error. After this, we can stop our search and instead focus on filtering to find the smallest subset of blocks that contain the syntax error.

Here's an example:

![](assets/syntax_search.gif)

## Use internals

To use the `syntax_suggest` gem without monkeypatching you can  `require 'syntax_suggest/api'`. This will allow you to load `syntax_suggest` and use its internals without mutating `require`.

Stable internal interface(s):

- `SyntaxSuggest.handle_error(e)`

Any other entrypoints are subject to change without warning. If you want to use an internal interface from `syntax_suggest` not on this list, open an issue to explain your use case.

## Development

### Handling conflicts with the default gem

Because `syntax_suggest` is a default gem you can get conflicts when working on this project with Ruby 3.2+. To fix conflicts you can disable loading `syntax_suggest` as a default gem by using then environment variable `RUBYOPT` with the value `--disable=syntax_suggest`. The `RUBYOPT` environment variable works the same as if we had entered those flags directly in the ruby cli (i.e. `ruby --disable=syntax_suggest` is the same as `RUBYOPT="--disable=syntax_suggest" ruby`). It's needed because we don't always directly execute Ruby and RUBYOPT will be picked up when other commands load ruby (`rspec`, `rake`, or `bundle` etc.).

There are some binstubs that already have this done for you. Instead of running `bundle exec rake` you can run `bin/rake`. Binstubs provided:

- `bin/bundle`
- `bin/rake`
- `bin/rspec`

### Installation

After checking out the repo, run `bin/setup` to install dependencies. Then, run `bin/rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bin/rake install`. To release a new version, update the version number in `version.rb`, and then run `bin/rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

### How to debug changes to output display

You can see changes to output against a variety of invalid code by running specs and using the `DEBUG_DISPLAY=1` environment variable. For example:

```
$ DEBUG_DISPLAY=1 bundle exec rspec spec/ --format=failures
```

### Run profiler

You can output profiler data to the `tmp` directory by running:

```
$ DEBUG_PERF=1 bundle exec rspec spec/integration/syntax_suggest_spec.rb
```

Some outputs are in text format, some are html, the raw marshaled data is available in `raw.rb.marshal`. See https://ruby-prof.github.io/#reports for more info. One interesting one, is the "kcachegrind" interface. To view this on mac:

```
$ brew install qcachegrind
```

Open:

```
$ qcachegrind tmp/last/profile.callgrind.out.<numbers>
```

## Environment variables

- `SYNTAX_SUGGEST_DEBUG` - Enables debug output to STDOUT/STDERR and/or disk at `./tmp`. The contents of debugging output are not stable and may change. If you would like stability, please open an issue to explain your use case.
- `SYNTAX_SUGGEST_TIMEOUT` - Changes the default timeout value to the number set (in seconds).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/zombocom/syntax_suggest. This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [code of conduct](https://github.com/zombocom/syntax_suggest/blob/main/CODE_OF_CONDUCT.md).


## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).

## Code of Conduct

Everyone interacting in the SyntaxSuggest project's codebases, issue trackers, chat rooms and mailing lists is expected to follow the [code of conduct](https://github.com/zombocom/syntax_suggest/blob/main/CODE_OF_CONDUCT.md).
