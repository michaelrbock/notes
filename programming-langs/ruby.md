# Ruby

`!` at the end of a method means it modifies the variable.

`?` means the method returns a boolean.

`attr_accessor` automatically makes setter and getter for a class attribute.

`||=`

`&&=`

`@` makes a variable an instance variable.

`:` makes a variable a "symbol" (basically a singleton string).

`&:` inside `each` calls that method on each item. It's actually the `&` operator being applied to a symbol.

Methods are referenced by symbols.

`&.` is like `try!` (from Rails) it lets you call methods on objects without worrying that the object may be `nil`.

`include` is for adding instance methods, `extend` is for adding class methods.

`require pry` then `binding.pry` to get a REPL in code.


## Rails

`before_action` runs a method before an action.

Getting a 500 error shows the request params.

Getting a 404 shows all the routes?

Use migrations to add/changes tables, attributes on models. `rails db:migrate`. `schema.rb` has db info.

`presence` is a Rails-specific thing, it's not in Ruby!

`blank?` if an object is false, empty, or a whitespace string. `present?` if it's not `blank?`.

From the console to use routes methods `include Rails.application.routes.url_helpers`

routes takes `"foo#bar"` and maps to `FooController#bar`

`try!` is better than `try`.

https://stackoverflow.com/questions/15746362/after-create-foo-vs-after-commit-bar-on-create

Forms automatically have validations. Model validations for objects underlying forms should really only apply to db-level stuff.

Helpers are automatically included in Views, use `helpers.` to access in Controllers, and the namespace must match the directory structure: `Discord::ChannelHelper` must be in `app/helpers/discord/channel_helper.rb`.

`belongs_to` automatically `validates` `presence: true` unless you add `optional: true` to the `belongs_to` line.



### Haml

If creating a `partial`, as a best practice, pass objects as `locals` (because they're shared across templates).

