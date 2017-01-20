# Strongbolt

RBAC framework for model-level authorization gem with very granular control on permissions, using Capabilities, Roles and UserGroups.

Checkout our [minimal example app](https://github.com/AnalyticsMediaGroup/strongbolt_example) to try it out without the need to set it up yourself.

Only works with Rails 4 and 5.

## Installation

Add this line to your application's Gemfile:

    gem 'strongbolt'

And then execute:

    $ bundle

## Getting Started

To creates the required migrations, run:

    $ rails g strongbolt:install && rake db:migrate

To create a Role and User Group that has full access and assign the group to all users (this allow to get started using StrongBolt), run:

    $ rake strongbolt:seed

If you plan on using the built-in view to manage user groups, roles and permissions, also add to your application.js:

    //= require strongbolt

You will need to have jQuery or a similar library with Ajax to make it work

There is a minimal example app using Strongbolt made available [here](https://github.com/AnalyticsMediaGroup/strongbolt_example). If you experience any issues setting Strongbolt up or configuring it, this example app is already setup for you and might be able to resolve some issues.

## Usage

### Configuration

The initializer of strongbolt, `config/initializers/strongbolt.rb` has some included documentation to help you with configuring the little you need to for Strongbolt to work.

By default, the user class is `User` and there is no tenant set.

#### A note on the list of models

Strongbolt has been made to be used with the least configuration possible. If given a tenant, it will traverse the graph of dependencies on this tenant to try to configure all the tenant dependent models authorization check ups. However, some models may be totally independant or not belong to (directly or indirectly) to a tenant. You may also have no tenant at all. In these cases, some or all of your models won't be discovered by Strongbolt when running your app.
To avoid eager loading the whole application to automatically get the list of models, you can specify in the initializer the models of your application.
This list is prefilled when running the install generator.

### Controllers

Strongbolt perform high level authorization on controllers, to avoid testing more granular authorization and increase the performance. For instance, if an user cannot find any Movies, he certainly won't be able to find the movie with the specific id 5.

#### Custom controller actions

Strongbolt relies on the usual Rails restful actions to guess the corresponding model action (edit requires update authorization, new requires create authorization, etc.). However, you will sometimes create other custom actions. In that case, you must specify in the controller how to map these custom controller actions to one of the 4 model actions using:

```ruby
authorize_as_find :action1, :action2
authorize_as_create :action, :action2
authorize_as_update :action, :action2
authorize_as_destroy :action, :action2
```

#### Skipping authorization

You can disable the controller-level authorization checks by using in the controllers:

```ruby
skip_controller_authorization,
skip_controller_authorization, only: [:index]
skip_controller_authorization, except: [:update]
```

You can also specify a list of controllers in the initializer `config/initializers/strongbolt.rb`. It is useful for third-party controllers, like devise for instance. The syntax is:

```ruby
config.skip_controller_authorization_for "Devise::Sessions", "Devise::Registrations"
```

You can also skip ALL authorization checks (BAD IDEA) using:

```ruby
skip_all_authorization
skip_all_authorization, only: [:index]
skip_all_authorization, except: [:update]
```

Sometimes you may want to render the views without checking again every single instance you're displaying (for the list of movies for instance, if the user can find ALL movies no need to test separatly each of them). You can skip the authorization checks when rendering pages by using:

```ruby
render_without_authorization :index, :show
```

Be careful when using one of this skipping authorization check as it may result in leaked data.

#### Controller not derived from a Model

Usually most of your controllers, in a RestFUL design, are backed by a specific model, derived from the name of the controller. In that case Strongbolt will know what model authorization it should test against. Otherwise, it will raise an error unless you specify the model for authorization:

```ruby
self.model_for_authorization = "Movie"
```

### Models

Usually you will never need to configure anything on the models. In some cases though, to simplify the roles and permissions, you may want to authorize some models as if they were other models (in nested models, sometimes it is safe to assume that the lower level model should have the same permissions than its parent)>
To achieve this, use the following within your model:

```ruby
authorize_as "Movie"
```

All ActiveRecord models will get the following methods added by Strongbolt.

##### ::bolted?

Returns true if authorization checks are enabled.

##### ::unbolted?

Returns true if authorization checks are disabled.

##### ::owned?

Returns true when the model is owned by user (has a belongs_to association with the user class). Example: `Client.owned?`

##### ::owner_attribute

The attribute of the model which is used to determine the owner. Example: `Client.owner_attribute` would return `:user_id`, `User.owner_attribute` would `:id`.

##### ::tenant?

Returns true when the model is a tenant (see below for a description of tenants).

##### ::authorize_as

See above. Allows a different model to be used for authorization checks instead.

##### ::name_for_authorization

See above. Returns the name of the model to be used for authorization checks.

### Users

Your user class (configured in the initializer via `config.user_class`) will have the following methods added by Strongbolt.

##### #capabilities

Returns all capabilities assigned to the user, including inherited ones. The return type is an array of `Strongbolt::Capability` instances. Example: `user.capabilities`

##### #add_tenant(tenant_instance)

Give the user access to the given tenant (see below for a description of tenants). Example: `user.add_tenant Client.find_by(name: 'AMG')`

##### #owns?(instance)

Checks whether the user own the given instance and returns a boolean. `instance` can be any instance of an ActiveRecord model. If the model has an attribute `user_id` it is compared to the user's ID to decide if he owns it. A user owns his own user instance. Example: `user.owns? Client.find_by(name: 'AMG')`, `user.owns? user`

##### #can?(action, instance, attrs = :any, all_instance = false)

Return true/false and determines whether the user is authorized to perform `action` on `instance`. `action` has to be a symbol (`:find, :create, :update, :destroy`). `instance` has to be a class or instance of an ActiveRecord model. `attrs` has to be `:any` always for now (it could be used for attribute level authorization, but that's not supported yet). `all_instance` can be set to true when `instance` is a call, to check whether the user can perform `action` on all instances of that class.
Examples:
- `user.can?(:find, Client.find_by(name: 'AMG'))`
- `user.can?(:create, Plan)`
- `user.can?(:update, Client, :any, true)`

##### #cannot?(...)

Inverse of `user.can?(...)`

##### #user_groups

ActiveRecord has_may Association with `Strongbolt::UserGroup`

##### #roles

ActiveRecord has_may Association with `Strongbolt::Role` through `Strongbolt::UserGroup`

##### #{tenants}

ActiveRecord has_may Association with the tenant model. The name is the pluralized version of the tenant model name. Example: `User.clients`

##### #accessible_{tenants}

Method returning all tenants of the type the user has access to. Example: `User.accessible_clients`


### Tenants

Strongbolt allows the utilization of _tenants_. Tenants are vertical scopes within your application.

> Let's take an example. A project tracking application (like Pivotal) can have several companies as clients, but each company's users can see only what concern them. _Company_ is a tenant of the application.

The initializer let you define the model(s) that should be considered tenants.

#### How to set what tenants a user has access to

Strongbolt comes with a table, `strongbolt_users_tenants`, that will store what tenants users have access to.

When a tenant is declared, it will add some features to the _User class_ that has been defined in the initializer.

First, an association between the _User class_ and the _Tenant class_ will be created, named after the _Tenant class_ name. It is a `has_many :trough => :users_tenants_` association. You can grant or revoke access to tenants just by interacting with that association.

> For instance, a `Company` tenant will generate a `companies` association.

> To grant access to `companyA` to the user `myUser`, you just add it to the association `myUser.companies << companyA`. To revoke access to all companies the user might have, you would use `myUser.companies.clear`.

A convenient instance method will also be created on the _User class_ to directly access the list of _Tenant class_ a _User_ can access. It is name `accessible_{tenants}` where `{tenants}` is the pluralize version of the _Tenant class_ name.

> `Company` will create an `accessible_companies` instance method

#### Tenanted models

A tenanted model is a model that belongs indirectly to a _Tenant class_.

> For instance, `Project` is a tenanted model of `Company`. Now, let's consider a `belong_to` association that links `Project` to a `Country`. The list of countries is stored in the table `countries` and does not belong to `Company`: `Country` is not a tenanted model.

Strongbolt will traverse your schema and automatically determine what models in your application is linked to your _Tenant class_.

> In our example, where `Project` belongs to `Company` and `Country`, but `Country` does not belong to `Company`, Strongbolt will automatically determine that `Project` is a tenanted model and `Country` is not.

Strongbolt will then create a `has_one` association on every tenanted model, so you can access directly the dependent _Tenant_

> For instance, every `Task` of a `Project` will have a `has_one :company, :through => :project` association automatically created (if not already existing).

#### Restricting permissions to accessible tenanted models

Strongbolt's capabilites have a boolean attribute, `require_tenant_access`, that specify whether the user can access all _tenanted models_ or only the ones that belong to the _Tenants_ he has access to.

> Let's look back at the example. Each company has several _projects_. The normal user, belonging to a company, would only have access to his companies projects. You would then define a capability *requiring tenant access* for the normal user.

> An admin user, on the other hand, like an engineer of the application, could have access to all the companies' projects. An engineer's projects' permissions would then *not require tenant access*

It will then perform the right permission check based on this requirement and the relationship of the model being checked and the _Tenant class_. If the model being checked is not linked to any of the _Tenant classes_, it won't check any dependency.


### Troubleshooting

#### Strongbolt::ModelNotFound

This means Strongbolt is trying to perform some controller-level authorizations but cannot infer the model from the controller name. In that case you must use in this controller:

```ruby
self.model_for_authorization = "Movie"
```

Or skip controller authorizations when it cannot be related to a model or it is not useful (like for Devise controllers), using either one of the methods described in *Skipping Authorization*.

#### Strongbolt::ActionNotConfigured

This happens when a controller is using a custom action which is not one of the 6 usual restful Rails actions (index, new, create, sho, edit, update, destroy). Refer to *Custom controller actions* to fix this.


## Contributing

1. Fork it ( http://github.com/<my-github-username>/strongbolt/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
