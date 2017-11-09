# Smash the State
A useful utility for transforming state that provides step sequencing, middleware, and validation.

# Example

``` ruby
class CreateUserOperation < SmashTheState::Operation
  # the schema defines how state is initialized. state begins life as a light-weight
  # struct with the keys defined below and values type-cast using
  # https://github.com/Azdaroth/active_model_attributes
  schema do
    attribute :email, :string
    attribute :name,  :string
    attribute :age,   :integer
  end

  # if a validate block is provided, a validation step will be inserted at its position in
  # the class relative to the other steps. failing validation will cause the operation to
  # exit, returning the current state
  validate do
    validates_presence_of :name, :email
  end

  # steps are executed in the order in which they are defined when the operation is
  # run. each step is expected to return the new state of the operation, which is handed
  # to the subsequent step
  step :normalize_data do |state|
    state.tap do |state|
      state.email = state.email.downcase
    end
  end

  custom_validation do |state|
    state.errors.add(:email, "no funny bizness") if state.email.end_with? ".biz"
  end

  # this step receives the normalized state from :normalize_data and creates a user,
  # which, because it is returned from the block, becomes the new operation state
  step :create_user do |state|
    User.create(state.as_json)
  end

  # this step receives the created user as its state and sends the user an email
  step :send_email do |user|
    email_job = Email.send(user.email, "hi there!")
    error!(user) if email_job.failed?
    user
  end

  # this error handler is attached to the :send_email step. if `error!`` is called
  # in a step, the error handler attached to the step is executed with the state as its
  # argument. an optional, second error argument may be passed in when error! is called.
  # if an error handler is run, operation execution halts, subsequent steps are not
  # executed, and the return value of the error handler block is returned to the caller
  error :send_email do |user|
    Logger.error "sending an email to #{user.email} failed"
    # do some error handling here
    user
  end
end

```

# Advanced stuff

## Middleware

You can define middleware classes to which the operation can delegate steps. The middleware class names can be arbitrarily composed by information pulled from the state.

Let's say you have two database types: `WhiskeyDB` and `AbsintheDB`, each of which have different behaviors in the `:create_environment` step. You can delegate those behaviors using middleware.

``` ruby
class WhiskeyDBCreateMiddleware
  def create_environment(state)
    # do whiskey things
  end
end

```

``` ruby
class AbsintheDBCreateMiddleware
  def create_environment(state)
    # do absinthe things
  end
end
```

``` ruby
class CreateDatabase < SmashTheState::Operation
  schema do
    attribute :name, :string
  end

  middleware_class do |state|
    "#{state.name}CreateMiddleware"
  end

  middleware_step :create_environment

  step :more_things do |state_from_middleware_step|
    # ... and so on
  end
end
```

## Representation

Let's say you want to represent the state of the operation, wrapped in a class that defines some `as_*` and `to_*` methods. You can do this with a `represent` step.

``` ruby
class DatabaseRepresenter
  def initialize(state)
    @state = state
  end

  def as_json
    {name: @state.name, foo: "bar"} # and so on
  end

  def as_xml
    XML.dump(@state)
  end
end
```

``` ruby
class CreateDatabaseOperation < SmashTheState::Operation
  schema do
    attribute :name, :string
  end

  # ... steps

  represent DatabaseRepresenter
end
```

``` ruby
> CreateDatabaseOperation.call(name: "AbsintheDB").as_json
=> {name: "AbsintheDB", foo: "bar"}
```

## Swagger

If you use [Swagger Blocks](https://github.com/fotinakis/swagger-blocks/), you can generate a parameter blocks from your operation schema by extending the state with the Swagger mixin. The `attribute` method is extended to support Swagger options like `description`, `required`, etc. The ActiveModel attribute types will be converted to the closest matching Swagger types for you (example: `:big_decimal` translates into Swagger type `:integer` with format `:int64`.)

``` ruby
class CreateDatabaseOperation < SmashTheState::Operation
  schema do
    extend SmashTheState::Operation::Swagger
    attribute :name, :string
  end
end
```

``` ruby
class DatabaseController < AbstractController
  swagger_path '/databases' do
    operation :post do
      key :summary, 'Create a database'
      CreateDatabaseOperation.eval_swagger_params(self)
    end
  end
end
```

### Overrides

Because operations are meant to be portable, sometimes you need to override the Swagger block defaults. Say, in this controller `name` is in the path instead of the body.

``` ruby
class DatabaseController < AbstractController
  swagger_path '/databases/{name}' do
    operation :post do
      key :summary, 'Create a database'

      # the :name param now will be `in` :query rather than :body
      CreateDatabaseOperation.override_swagger_param(:name) do
        key :in, :path
      end

      CreateDatabaseOperation.eval_swagger_params(self)
    end
  end
end
```

Or if you need to change some keys for all the params, you can use `override_swagger_params`:

``` ruby
class DatabaseController < AbstractController
  swagger_path '/databases' do
    operation :post do
      key :summary, 'Create a database'

      # all swagger params now will be `in` :query rather than :body
      CreateDatabaseOperation.override_swagger_params do
        key :in, :query
      end

      CreateDatabaseOperation.eval_swagger_params(self)
    end
  end
end
```
