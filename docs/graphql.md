# GraphQL integration

You can use Action Policy as an authorization library for you [GraphQL Ruby](https://graphql-ruby.org/) application via the [`action_policy-graphql` gem](https://github.com/palkan/action_policy-graphql).

This integration provides the following features:
- Fields & mutations authorization
- List and connections scoping
- [**Exposing permissions/authorization rules in the API**](https://dev.to/evilmartians/exposing-permissions-in-graphql-apis-with-action-policy-1mfh).

## Getting Started

First, add `action_policy-graphql` gem to your Gemfile (see [installation instructions](https://github.com/palkan/action_policy-graphql#installation)).

Then, include `ActionPolicy::GraphQL::Behaviour` to your base type (or any other type/mutation where you want to use authorization features):

```ruby
# For fields authorization, lists scoping and rules exposing
class Types::BaseObject < GraphQL::Schema::Object
  include ActionPolicy::GraphQL::Behaviour
end

# For using authorization helpers in mutations
class Types::BaseMutation < GraphQL::Schema::Mutation
  include ActionPolicy::GraphQL::Behaviour
end
```

## Authorization Context

By default, Action Policy use `context[:current_user]` as the `user` [authorization context](./authoriation_context.md).

**NOTE:** see below for more information on what's included into `ActionPolicy::GraphQL::Behaviour`.

## Authorizing Fields

You can add `authorize: true` option to any field (=underlying object) to protect the access (it's equal to calling `authorize! object, to: :show?`):

```ruby
# authorization could be useful for find-like methods,
# where the object is resolved from the provided params (e.g., ID)
field :home, Home, null: false, authorize: true do
  argument :id, ID, required: true
end

def home(id:)
  Home.find(id)
end

# Without `authorize: true` the code would look like this
def home(id:)
  Home.find(id).tap { |home| authorize! home, to: :show? }
end
```

You can use authorization options to customize the behaviour, e.g. `authorize: {to: :preview?, with: CustomPolicy}`.

By default, if a user is not authorized to access the field, an `ActionPolicy::Unauthorized` exception is raised.

If you want to return a `nil` instead, you should add `raise: false` to the options:

```ruby
# NOTE: don't forget to mark your field as nullable
field :home, Home, null: true, authorize: {raise: false}
```

You can make non-raising behaviour a default by setting a configuration option:

```ruby
ActionPolicy::GraphQL.authorize_raise_exception = false
```

You can also change the default `show?` rule globally:

```ruby
ActionPolicy::GraphQL.default_authorize_rule = :show_graphql_field?
```

### Class-level authorization

You can use Action Policy in the class-level [authorization hooks](https://graphql-ruby.org/authorization/authorization.html) (`self.authorized?`) like this:

```ruby
class Types::Friendship < Types::BaseObject
  def self.authorized?(object, context)
    super &&
      object.allowed_to?(
        :show?,
        object,
        context: {user: context[:current_user]}
      )
  end
end
```

## Authorizing Mutations

Mutation is just a Ruby class with a single API method. There is nothing specific in authorizing mutations: from the Action Policy point of view, they are just [_behaviours_](./behaviour.md).

If you want to authorize the mutation, you call `authorize!` method. For example:

```ruby
class Mutations::DestroyUser < Types::BaseMutation
  argument :id, ID, required: true

  def resolve(id:)
    user = User.find(id)

    # Raise an exception if the user has not enough permissions
    authorize! user, to: :destroy?
    # Or check without raising and do what you want
    #
    #     if allowed_to?(:destroy?, user)

    user.destroy!

    {deleted_id: user.id}
  end
end
```

## Handling exceptions

The query would fail with `ActionPolicy::Unauthorized` exception when using `authorize: true` (in raising mode) or calling `authorize!` explicitly.

That could be useful to handle this exception and send a more detailed error message to the client, for example:

```ruby
# in your schema file
rescue_from(ActionPolicy::Unauthorized) do |exp|
  raise GraphQL::ExecutionError.new(
    # use result.message (backed by i18n) as an error message
    exp.result.message,
    # use GraphQL error extensions to provide more context
    extensions: {
      code: :unauthorized,
      fullMessages: exp.result.reasons.full_messages,
      details: exp.result.reasons.details
    }
  )
end
```

## Scoping Data

You can add `authorized_scope: true` option to a field (list or [_connection_](https://graphql-ruby.org/relay/connections.html)) to apply the corresponding policy rules to the data:

```ruby
class CityType < ::Common::Graphql::Type
  # It would automatically apply the relation scope from the EventPolicy to
  # the relation (city.events)
  field :events, EventType.connection_type,
        null: false,
        authorized_scope: true

  # you can specify the policy explicitly
  field :events, EventType.connection_type,
        null: false,
        authorized_scope: {with: CustomEventPolicy}

  # without the option you would write the following code
  def events
    authorized_scope object.events
    # or if `with` option specified
    authorized_scope object.events, with: CustomEventPolicy
  end
end
```

See the documenation on [scoping](./scoping.md).

## Exposing Authorization Rules

With `action_policy-graphql` gem, you can easily expose your authorization logic to the client in a standardized way.

For example, if you want to "tell" the client which actions could be performed against the object you
can use the `expose_authorization_rules` macro to add authorization-related fields to your type:

```ruby
class ProfileType < Types::BaseType
  # Adds can_edit, can_destroy fields with
  # AuthorizationResult type.

  # NOTE: prefix "can_" is used by default, no need to specify it explicitly
  expose_authorization_rules :edit?, :destroy?, prefix: "can_"
end
```

**NOTE:** you can use [aliases](./aliases.md) here as well as defined rules.

 **NOTE:** This feature relies the [_failure reasons_](./reasons.md) and
the [i18n integration](./i18n.md) extensions. If your policies don't include any of these,
you won't be able to use it.

Then the client could perform the following query:

```gql
{
  post(id: $id) {
    canEdit {
      # (bool) true|false; not null
      value
      # top-level decline message ("Not authorized" by default); null if value is true
      message
      # detailed information about the decline reasons; null if value is true or you don't have "failure reasons" extension enabled
      reasons {
        details # JSON-encoded hash of the form { "event" => [:privacy_off?] }
        fullMessages # Array of human-readable reasons
      }
    }

    canDestroy {
      # ...
    }
  }
}
```

You can override a custom authorization field prefix (`can_`):

```ruby
ActionPolicy::GraphQL.default_authorization_field_prefix = "allowed_to_"
```

## Custom Behaviour

Including the default `ActionPolicy::GraphQL::Behaviour` is equal to adding the following to your base class:

```ruby
class Types::BaseObject < GraphQL::Schema::Object
  # include Action Policy behaviour and its extensions
  include ActionPolicy::Behaviour
  include ActionPolicy::Behaviours::ThreadMemoized
  include ActionPolicy::Behaviours::Memoized
  include ActionPolicy::Behaviours::Namespaced

  # define authorization context
  authorize :user, through: :current_user

  # add a method helper to get the current_user from the context
  def current_user
    context[:current_user]
  end

  # extend the field class to add `authorize` and `authorized_scope` options
  field_class.prepend(ActionPolicy::GraphQL::AuthorizedField)

  # add `expose_authorization_rules` macro
  include ActionPolicy::GraphQL::Fields
end
```

Feel free to create your own behaviour by adding only the functionality you need.
