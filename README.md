# Decanter

Decanter is a Ruby gem that makes it easy to transform incoming data before it hits the model.

```ruby
gem 'decanter', '~> 3'
```

## Usage

Declare a `Decanter` for a model:

```ruby
# app/decanters/trip_decanter.rb

class TripDecanter < Decanter::Base
  input :name, :string
  input :start_date, :date
  input :end_date, :date
end
```

Transform incoming params in your controller using `Decanter#decant`:

```rb
# app/controllers/trips_controller.rb

  def create
    trip_params = TripDecanter.decant(params[:trip])
    @trip = Trip.new(trip_params)

    # ...any response logic
  end

```

## Generators

Decanter comes with generators for creating `Decanter` and `Parser` files:

```
rails g decanter Trip name:string start_date:date end_date:date
```

```
rails g parser TruncatedString
```

## Nested resources

Decanters can declare relationships using `ActiveRecord`-style declarators:

```ruby
class TripDecanter < Decanter::Base
  has_many :destinations
end
```

## Default parsers

Decanter comes with the following parsers out of the box:

- `:boolean`
- `:date`
- `:date_time`
- `:float`
- `:integer`
- `:pass`
- `:phone`
- `:string`
- `:array`

Note: these parsers are designed to operate on a single value, except for `:array`. This parser expects an array, and will use the `parse_each` option to call a given parser on each of its elements:

```ruby
input :ids, :array, parse_each: :integer
```

## Parser options

Parsers can receive options that modify their behavior. These options are passed in as named arguments to `input`:

```ruby
input :start_date, :date, parse_format: '%Y-%m-%d'
```

This decanter will look up and apply the corresponding `DestinationDecanter` whenever necessary to transform nested resources.

## Exception handling

By default, `Decanter#decant` will raise will raise an exception when unexpected parameters are passed. To override this behavior, you can disable strict mode:

```ruby
class TripDecanter <  Decanter::Base
  strict false
  # ...
end
```

Or explicitly ignore a key:

```rb
class TripDecanter <  Decanter::Base
  ignore :created_at, :updated_at
  # ...
end
```

## Advanced Usage

### Custom Parsers

To add a custom parser, first create a parser class:

```rb
# app/parsers/truncate_string_parser.rb
class TruncateStringParser < Decanter::Parser::ValueParser

  parser do |value, options|
    length = options.fetch(:length, 100)
    value.truncate(length)
  end
end
```

Then, use the appropriate key to look up the parser:

```ruby
  input :name, :truncate_string #=> TruncateStringParser
```

#### Custom parser methods

- `#parse <block>`: (required) recieves a block for parsing a value. Block parameters are `|value, options|` for `ValueParser` and `|name, value, options|` for `HashParser`.
- `#allow [<class>]`: skips parse step if the incoming value `is_a?` instance of class(es).
- `#pre [<parser>]`: applies the given parser(s) before parsing the value.

#### Custom parser base classes

- `Decanter::Parser::ValueParser`: subclasses are expected to return a single value.
- `Decanter::Parser::HashParser`: subclasses are expected to return a hash of keys and values.

### Squashing inputs

Sometimes, you may want to take several inputs and combine them into one finished input prior to sending to your model. You can achieve this with a custom parser:

```ruby
class TripDecanter < Decanter::Base
  input [:day, :month, :year], :squash_date, key: :start_date
end
```

```ruby
class SquashDateParser < Decanter::Parser::ValueParser
  parser do |values, options|
    day, month, year = values.map(&:to_i)
    Date.new(year, month, day)
  end
end
```

### Chaining parsers

You can compose multiple parsers by using the `#pre` method:

```ruby
class FloatPercentParser < Decanter::Parser::ValueParser

  pre :float

  parser do |val, options|
    val / 100
  end
end
```

Or by declaring multiple parsers for a single input:

```ruby
class SomeDecanter < Decanter::Base
  input :some_percent, [:float, :percent]
end
```

### Requiring Params

If you provide the option `:required` for an input in your decanter, an exception will be thrown if the parameter is `nil` or an empty string.

```ruby
class TripDecanter <  Decanter::Base
  input :name, :string, required: true
end
```

_Note: we recommend using [Active Record validations](https://guides.rubyonrails.org/active_record_validations.html) to check for presence of an attribute, rather than using the `required` option. This method is intended for use in non-RESTful routes or cases where Active Record validations are not available._

### Global Configuration

You can generate a local copy of the default configuration with `rails generate decanter:install`. This will create an initializer where you can do global configuration:

```ruby
# ./config/initializers/decanter.rb

Decanter.config do |config|
  config.strict = false
end
```
