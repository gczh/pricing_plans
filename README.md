# 💵 `pricing_plans` - Define and enforce pricing plan limits in your Rails app

[![Gem Version](https://badge.fury.io/rb/pricing_plans.svg)](https://badge.fury.io/rb/pricing_plans)

`pricing_plans` is the single source of truth for pricing plans and plan limits in your Rails apps. It provides methods you can use across your app to consistently check whether users can perform an action based on the plan they're currently subscribed to.

Define plans and their quotas and limits:
```ruby
plan :pro do
  limits :projects, to: 5.max
  allows :api_access
end
```

Then, gate features in your controllers:
```ruby
before_action :enforce_api_access!, only: [:create]
```

Enforce limits in your models too:
```ruby
class User < ApplicationRecord
  has_many :projects, limited_by_pricing_plans: true
end
```

Or anywhere in your app:
```ruby
@user.projects_remaining
# => 2
```

> [!TIP]
> The `pricing_plans` gem works seamlessly out of the box with [`pay`](https://github.com/pay-rails/pay) and [`usage_credits`](https://github.com/rameerez/usage_credits/). More info [here](#using-with-pay-andor-usage_credits).

## Quickstart

Add this to your Gemfile:

```ruby
gem "pricing_plans"
```

Then install the gem:

```bash
bundle install
rails g pricing_plans:install
rails db:migrate
```

This will generate and migrate [the necessary models](#why-the-models) to make the gem work. It will also create a `config/initializers/pricing_plans.rb` file where you need to define your pricing plans.

Important:
- You must set a default plan (either mark one with `default!` in the plan DSL or set `config.default_plan = :your_plan_key`).
- In the initializer, we enable integer DSL sugar like `5.max` via `using PricingPlans::IntegerRefinements`.

Once installed, just add the model mixin to the actual `billable` model on which limits should be enforced (like: `User`, `Organization`, etc.):

```ruby
class User < ApplicationRecord
  include PricingPlans::Billable
end
```

And that automatically gives your model all plan-limiting helpers and methods, so you can proceed to enforce plan limits in your models like this:
```ruby
class User < ApplicationRecord
  include PricingPlans::Billable

  has_many :projects, limited_by_pricing_plans: { error_after_limit: "Too many projects!" }, dependent: :destroy
end
```

There's many helper methods to help you enforce limits and feature gating in controller, methods, and everywhere in your app: read the [full API reference](#available-methods--full-api-reference).


## Why this gem exists

`pricing_plans` helps you avoid reimplementing feature gating over and over again across your project.

If you've ever had to implement pricing plan limits, you probably found yourself writing code like this everywhere in your app:

```ruby
if user_signed_in? && current_user.payment_processor&.subscription&.processor_plan == "pro" && current_user.projects.count <= 5
  # ...
elsif user_signed_in? && current_user.payment_processor&.subscription&.processor_plan == "premium" && current_user.projects.count <= 10
  # ...
end
```

You end up duplicating this kind of snippet for every plan and feature, and for every view and controller.

This code is brittle, tends to be full of magical numbers and nested convoluted logic; and plan enforcement tends to be scattered across the entire codebase. If you change something in your pricing table, it's highly likely you'll have to change the same magical number or logic in many different places, leading to bugs, inconsistencies, customer support tickets, and maintenance hell.

`pricing_plans` aims to offer a centralized, single-source-of-truth way of defining & handling pricing plans, so you can enforce plan limits with reusable helpers that read like plain English.

### Some features

Enforcing pricing plans is one of those boring plumbing problems that look easy from a distance but get complex when you try to engineer them for production usage. The poor man's implementation of nested ifs shown in the example above only get you so far, you soon start finding edge cases to consider. Here's some of what we've covered in this gem:

- Safe under load: we use row locks and retries when setting grace/blocked/warning state, and we avoid firing the same event twice. See [grace_manager.rb](lib/pricing_plans/grace_manager.rb).

- Accurate counting: persistent limits count live current rows (using `COUNT(*)`, make sure to index your foreign keys to make it fast at scale); per‑period limits record usage for the current window only. You can filter what counts with `count_scope` (Symbol/Hash/Proc/Array), and plan settings override model defaults. See [limitable.rb](lib/pricing_plans/limitable.rb) and [limit_checker.rb](lib/pricing_plans/limit_checker.rb).

- Clear rules: default is to block when you hit the cap; grace periods are opt‑in. In status/UI, 0 of 0 isn’t shown as blocked. See [plan.rb](lib/pricing_plans/plan.rb), [grace_manager.rb](lib/pricing_plans/grace_manager.rb), and [view_helpers.rb](lib/pricing_plans/view_helpers.rb).

- Simple controllers: one‑liners to guard actions, predictable redirect order (per‑call → per‑controller → global → pricing_path), and an optional central handler. See [controller_guards.rb](lib/pricing_plans/controller_guards.rb).

- Billing‑aware periods: supports billing cycle (when Pay is present), calendar month/week/day, custom time windows, and durations. See [period_calculator.rb](lib/pricing_plans/period_calculator.rb).

## Define pricing plans

You define plans and their limits and features in the `pricing_plans.rb` initializer. To define a free plan, for example, you would do:

```ruby
PricingPlans.configure do |config|
  # Enable integer sugar like 5.max (the installer adds this for you):
  # using PricingPlans::IntegerRefinements

  plan :free do
    price 0
    default!
  end
end
```

That's the basics! Let's dive in.

### Define plan limits (quotas) and features

At a high level, a plan needs to do **two** things:
  1) Gate features
  2) Enforce limits (quotas)

#### 1) Gate features in a plan

Let's start by giving access to certain features. For example, our free plan could give users API access:

```ruby
PricingPlans.configure do |config|
  plan :free do
    price 0

    allows :api_access
  end
end
```

We're just **defining** what the plan does now. Later, we'll see [all the methods we can use to enforce these limits and gate these features](#gate-features-in-controllers) very easily.


All features are disabled by default unless explicitly made available with the `allows` keyword. However, for clarity we can explicitly say what the plan disallows:

```ruby
PricingPlans.configure do |config|
  plan :free do
    price 0

    allows :api_access
    disallows :premium_features
  end
end
```

This wouldn't do anything, though, because all features are disabled by default; but it makes it obvious what the plan does and doesn't.

#### 2) Enforce limits (quotas) in a plan

The other thing plans can do is enforce a limit. We can define limits like this:

```ruby
PricingPlans.configure do |config|
  plan :free do
    price 0

    allows :api_access

    limits :projects, to: 3
  end
end
```

The `limits :projects, to: 3` does exactly that: whoever has this plan can only have three projects at most. We'll see later [how to tie this limit to the actual model relationship](#models), but for now, we're just **defining** the limit.

##### `after_limit`: Define what happens after a limit is reached

What happens after a limit is reached is controlled by `after_limit`. The default is `:block_usage`. You can customize per limit. Examples:

```ruby
# Just warn (never block):
PricingPlans.configure do |config|
  plan :free do
    price 0
    allows :api_access
    limits :projects, to: 3, after_limit: :just_warn
  end
end
```

If we want to prevent more resources being created after the limit has been reached, we can `:block_usage`:

```ruby
# Block immediately:
PricingPlans.configure do |config|
  plan :free do
    price 0
    allows :api_access
    limits :projects, to: 3, after_limit: :block_usage
  end
end
```

However, we can be nicer and give users a bit of a grace period after the limit has been reached. To do that, we use `:grace_then_block`:

```ruby
# Opt into grace, then block:
PricingPlans.configure do |config|
  plan :free do
    price 0
    allows :api_access
    limits :projects, to: 3, after_limit: :grace_then_block
  end
end
```

We can also specify how long the grace period is:

```ruby
PricingPlans.configure do |config|
  plan :free do
    price 0
    allows :api_access
    limits :projects, to: 3, after_limit: :grace_then_block, grace: 7.days
  end
end
```

In summary: persistent caps count live rows (per billable model). When over the cap:
  - `:just_warn` → validation passes; use controller guard to warn.
  - `:block_usage` → validation fails immediately (uses `error_after_limit` if set).
  - `:grace_then_block` → validation fails once grace is considered “blocked” (we track and switch from grace to blocked).

Note: `grace` is only valid with blocking behaviors. We’ll raise at boot if you set `grace` with `:just_warn`.

##### Per‑period allowances

Besides persistent caps, a limit can be defined as a per‑period allowance that resets each window. Example:

```ruby
plan :pro do
  # Allow up to 3 custom models per calendar month
  limits :custom_models, to: 3, per: :calendar_month
end
```

Accepted `per:` values:
- `:billing_cycle` (default globally; respects Pay subscription anchors if available, else falls back to calendar month)
- `:calendar_month`, `:calendar_week`, `:calendar_day`
- A callable: `->(billable) { [start_time, end_time] }`
- An ActiveSupport duration: `2.weeks` (window starts at beginning of day)

Per‑period usage is tracked in [the `PricingPlans::Usage` model (`pricing_plans_usages` table)](#why-the-models) and read live. Persistent caps do not use this table.

###### How period windows are calculated

- **Default period**: Controlled by `config.period_cycle` (defaults to `:billing_cycle`). You can override per limit with `per:`.
- **Billing cycle**: When `pay` is available, we use the subscription’s anchors (`current_period_start`/`current_period_end`). If not available, we fall back to a monthly window anchored at the subscription’s `created_at`. If there is no subscription, we fall back to calendar month.
- **Calendar windows**: `:calendar_month`, `:calendar_week`, `:calendar_day` map to `beginning_of_* … end_of_*` for the current time.
- **Duration windows**: For `ActiveSupport::Duration` (e.g., `2.weeks`), the window starts at `beginning_of_day` and ends at `start + duration`.
- **Custom callable**: You can pass `->(billable) { [start_time, end_time] }`. We validate that both are present and `end > start`.

###### Automatic usage tracking (race‑safe)

- Include `limited_by_pricing_plans` on the model that represents the metered object. On `after_create`, we atomically upsert/increment the current period’s usage row for that `billable` and `limit_key`.
- Concurrency: we de‑duplicate with a uniqueness constraint and retry on `RecordNotUnique` to increment safely.
- Reads are live: `LimitChecker.current_usage_for(billable, :key)` returns the current window’s `used` (or 0 if none).

Callback timing:
- We increment usage in an `after_create` callback (not `after_commit`). This runs inside the same database transaction as the record creation, so if the outer transaction rolls back, the usage increment rolls back as well.

###### Grace/warnings and period rollover (explicit semantics)

- State lives in `pricing_plans_enforcement_states` per billable+limit.
- Per‑period limits:
  - We stamp the active window on the state; when the window changes, stale state is discarded automatically (warnings re‑arm and grace resets at each new window).
  - Warnings: thresholds re‑arm every window; the same threshold can emit again in the next window.
  - Grace: if `:grace_then_block`, grace is per window. A new window clears prior grace/blocked state.
- Persistent caps:
  - Warnings are monotonic: once a higher `warn_at` threshold has been emitted, we do not re‑emit lower or equal thresholds again unless you clear state via `PricingPlans::GraceManager.reset_state!(billable, :limit_key)`.
  - Grace is absolute: if `:grace_then_block`, we start grace once the limit is exceeded. It expires after the configured duration. There is no automatic reset tied to time windows. Enforcement for creates is still driven by “would this action exceed the cap now?”. If usage drops below the cap, create checks will pass again even if a prior state exists.
  - You may clear any existing warning/grace/blocked state manually with `reset_state!`.

###### Example: usage resets next period

```ruby
# pro allows 3 custom models per month
PricingPlans::Assignment.assign_plan_to(org, :pro)

travel_to(Time.parse("2025-01-15 12:00:00 UTC")) do
  3.times { org.custom_models.create!(name: "Model") }
  PricingPlans::LimitChecker.plan_limit_remaining(org, :custom_models)
  # => 0
  result = PricingPlans::ControllerGuards.require_plan_limit!(:custom_models, billable: org)
  result.grace? # => true when after_limit: :grace_then_block
end

travel_to(Time.parse("2025-02-01 12:00:00 UTC")) do
  # New window — counters reset automatically
  PricingPlans::LimitChecker.plan_limit_remaining(org, :custom_models)
  # => 3
end
```

##### Warn users when they cross a limit threshold

We can also set thresholds to warn our users when they're halfway through their limit, approaching the limit, etc. To do that, we first set up trigger thresholds with `warn_at:`

```ruby
PricingPlans.configure do |config|
  plan :free do
    price 0

    allows :api_access

    limits :projects, to: 3, after_limit: :grace_then_block, grace: 7.days, warn_at: [0.5, 0.8, 0.95]
  end
end
```

And then, for each threshold and for each limit, an event gets triggered, and we can configure its callback in the `pricing_plans.rb` initializer:

```ruby
config.on_warning(:projects) do |billable, threshold|
  # send a mail or a notification
  # this fires when :projects crosses 50%, 80% and 95% of its limit
end

# Also available:
config.on_grace_start(:projects) do |billable, grace_ends_at|
  # notify grace started; ends at `grace_ends_at`
end
config.on_block(:projects) do |billable|
  # notify usage is now blocked for :projects
end
```

If you only want a scope, like active projects, to count towards plan limits, you can do:

```ruby
PricingPlans.configure do |config|
  plan :free do
    price 0

    allows :api_access

    limits :projects, to: 3.max, count_scope: :active
  end
end
```

(Assuming, of course, that your `Project` model has an `active` scope)

You can also make something unlimited (again, just syntactic sugar to be explicit, everything is unlimited unless there's an actual limit):

```ruby
PricingPlans.configure do |config|
  plan :free do
    price 0

    allows :api_access

    unlimited :projects
  end
end
```

##### Limits API reference

To summarize, here's what persistent caps (plan limits) are:
  - Counting is live: `SELECT COUNT(*)` scoped to the billable association, no counter caches.
  - Validation on create: blocks immediately on `:block_usage`, or blocks when grace is considered “blocked” on `:grace_then_block`. `:just_warn` passes.
  - Deletes automatically lower the count. Backfills simply reflect current rows.

  - Filtered counting via count_scope: scope persistent caps to active-only rows.
    - Idiomatic options:
      - Plan DSL with AR Hash: `limits :licenses, to: 25, count_scope: { status: 'active' }`
      - Plan DSL with named scope: `limits :activations, to: 50, count_scope: :active`
      - Plan DSL with multiple: `limits :seats, to: 10, count_scope: [:active, { kind: 'paid' }]`
  - Macro form on the child model: `limited_by_pricing_plans :licenses, billable: :organization, count_scope: :active`
  - Billable‑side convenience: `has_many :licenses, limited_by_pricing_plans: { limit_key: :licenses, count_scope: :active }`
  - Full freedom: `->(rel) { rel.where(status: 'active') }` or `->(rel, billable) { rel.where(organization_id: billable.id) }`
    - Accepted types: Symbol (named scope), Hash (where), Proc (arity 1 or 2), or Array of these (applied left-to-right).
    - Precedence: plan-level `count_scope` overrides macro-level `count_scope`.
    - Restriction: `count_scope` only applies to persistent caps (not allowed on per-period limits).
    - Performance: add indexes for your filters (e.g., `status`, `deactivated_at`).


### Define user-facing plan attributes

Since this is our single source of truth for plans, we can define plan information we can later use to show pricing tables, like plan name, description, and bullet points. We can also override the price for a string, and we can set a CTA button text and URL to link to:

```ruby
PricingPlans.configure do |config|
  plan :free do
    price_string "Free!"

    name "Free Plan" # optional, would default to "Free" as inferred from the :free key
    description "A plan to get you started"
    bullets "Basic features", "Community support"

    cta_text "Subscribe"
    # In initializers, prefer a string path/URL or set a global default CTA in config.
    # Route helpers are not available here.
    cta_url  "/pricing"

    allows :api_access

    limits :projects, to: 3
  end
end
```

You can also make a plan `default!`; and you can make a plan `highlighted!` to help you when building a pricing table.

### Link plans to Stripe prices

If we're defining a paid plan, it's better to omit the explicit price, and just let the gem read the actual price from Stripe (assuming the `pay` gem is set up correctly):

```ruby
PricingPlans.configure do |config|
  plan :pro do
    stripe_price "price_123abc"

    description "For growing teams and businesses"
    bullets "Advanced features", "Priority support", "API access"

    allows :api_access, :premium_features
    limits :projects, to: 10
    unlimited :team_members
    highlighted!
  end
end
```

If you have monthly and yearly prices for the same plan, you can define them like:

```ruby
PricingPlans.configure do |config|
  plan :pro do
    stripe_price month: "price_123abc", year: "price_456def"
  end
end
```

`stripe_price` accepts String or Hash (e.g., `{ month:, year:, id: }`) and the `pricing_plans` PlanResolver maps against Pay's `subscription.processor_plan`.

## Usage: available methods & full API reference

Assuming you've correctly installed the gem and configured your pricing plans in `pricing_plans.rb`, here's everything you can do:

### Models

#### Define your `Billable` class and add limits to your model

Your `Billable` class is the class on which limits are enforced. It's usually the same class that gets charged for a subscription, the class which "owns" the plan, the class with the `pay_customer` if you're using Pay, etc. It's usually `User`, `Organization`, `Team`, etc.

To define your `Billable` class, just add the model mixin:

```ruby
class User < ApplicationRecord
  include PricingPlans::Billable
end
```

Now you can link `has_many` relationships in this model to `limits` defined in your `pricing_plans.rb`

For example, if you defined a `:projects` limit in your `pricing_plans.rb` like this:

```ruby
plan :pro do
  limits :projects, to: 5
end
```

then you can link `:projects` to any `has_many` relationship on the `Billable` model (`User`, in this example):

```ruby
class User < ApplicationRecord
  include PricingPlans::Billable

  has_many :projects, limited_by_pricing_plans: true
end
```

The `:limited_by_pricing_plans` infers that the association name (:projects) is the same as the limit key you defined on `pricing_plans.rb`. If that's not the case, you can make it explicit:

```ruby
class User < ApplicationRecord
  include PricingPlans::Billable

  has_many :custom_projects, limited_by_pricing_plans: { limit_key: :projects }
end
```

In general, you can omit the limit key when it can be inferred from the model (e.g., `Project` → `:projects`).

`limited_by_pricing_plans` plays nicely with every other ActiveRecord validation you may have in your relationship:
```ruby
class User < ApplicationRecord
  include PricingPlans::Billable

  has_many :projects, limited_by_pricing_plans: true, dependent: :destroy
end
```

You can also customize the validation error message by passing `error_after_limit`. This error message behaves like other ActiveRecord validation, and will get attached to the record upon failed creation because of limits:
```ruby
class User < ApplicationRecord
  include PricingPlans::Billable

  has_many :projects, limited_by_pricing_plans: { error_after_limit: "Too many projects!" }, dependent: :destroy
end
```


#### Enforce limits in your `Billable` class

The `Billable` class (the class to which you add the `include PricingPlans::Billable` mixin) automatically gains these helpers to check limits:

```ruby
# Check limits for a relationship
user.plan_limit_remaining(:projects)         # => integer or :unlimited
user.plan_limit_percent_used(:projects)      # => Float percent
user.within_plan_limits?(:projects, by: 1)   # => true/false

# Grace helpers
user.grace_active_for?(:projects)            # => true/false
user.grace_ends_at_for(:projects)            # => Time or nil
user.grace_remaining_seconds_for(:projects)  # => Integer seconds
user.grace_remaining_days_for(:projects)     # => Integer days (ceil)
user.plan_blocked_for?(:projects)            # => true/false (considering after_limit policy)
```

We also add syntactic sugar methods. For example, if your plan defines a limit for `:projects` and you have a `has_many :projects` relationship, you also get these methods:
```ruby
# Check limits (per `limits` key)
user.projects_remaining
user.projects_percent_used
user.projects_within_plan_limits?

# Grace helpers (per `limits` key)
user.projects_grace_active?
user.projects_grace_ends_at
user.projects_blocked?
```

These methods are dynamically generated for every `has_many :<limit_key>`, like this:
- `<limit_key>_remaining`
- `<limit_key>_percent_used`
- `<limit_key>_within_plan_limits?` (optionally: `<limit_key>_within_plan_limits?(by: 1)`)
- `<limit_key>_grace_active?`
- `<limit_key>_grace_ends_at`
- `<limit_key>_blocked?`

You can also check for feature flags like this:
```ruby
user.plan_allows?(:api_access)               # => true/false
```

And, if you want to get aggregates across all keys instead of checking them individually:
```ruby
# Aggregates across keys
user.any_grace_active_for?(:products, :activations)
user.earliest_grace_ends_at_for(:products, :activations)
```

You can also check and override the current pricing plan for any user:
```ruby
user.current_pricing_plan                    # => PricingPlans::Plan
user.assign_pricing_plan!(:pro)              # manual assignment override
user.remove_pricing_plan!                    # remove manual override (fallback to default)
```

And finally, you get very thin convenient wrappers if you're using `pay`:
```ruby
# Pay (Stripe) convenience (returns false/nil when Pay is absent)
# Note: this is billing-facing state, distinct from our in-app
# enforcement grace which is tracked per-limit.
user.pay_subscription_active?                # => true/false
user.pay_on_trial?                           # => true/false
user.pay_on_grace_period?                    # => true/false
```

### Controllers

#### Setting things up for controllers

First of all, the gem needs a way to know what the current billable object is (the current user, current organization, etc.)

`pricing_plans` will [auto-try common conventions](/lib/pricing_plans/controller_guards.rb): `current_organization`, `current_account`, `current_user`, `current_team`, `current_company`, `current_workspace`, `current_tenant`. If you set `billable_class`, we’ll also try `current_<billable_class>`.

If these methods already defined in your controller(s), there's nothing you need to do! For example: `pricing_plans` works out of the box with Devise.

If none of those methods are defined, or you want custom logic, we recommend defining a current billable helper in your `ApplicationController`:
```ruby
class ApplicationController < ActionController::Base
  # Adapt to your auth/session logic
  def current_organization
    # Your lookup here (e.g., current_user.organization)
  end
end
```

You can explicitly configure the billable resolution per controller:
```ruby
class ApplicationController < ActionController::Base
  pricing_plans_billable :current_organization

  # Or pass a block:
  # pricing_plans_billable { current_user&.organization }
end
```

Optionally, you can set a global resolver in the initializer (per-controller still wins):
```ruby
# config/initializers/pricing_plans.rb
PricingPlans.configure do |config|
  # Either:
  config.controller_billable :current_organization
  # Or:
  # config.controller_billable { current_account }
end
```

You can also override the `current_<billable_class>` per helper (with `on: :current_organization` or `on: -> { find_org }`), as we'll see in the next sections.

Once all of this is configured, you can gate features and enforce limits easily in your controllers.

#### Gate features in controllers

Feature-gate any controller action with:
```ruby
before_action :enforce_api_access!, only: [:create]
```

These `enforce_<feature_key>!` controller helper methods are dynamically generated for each of the features `<feature_key>` you defined in your plans. So, for the helper above to work, you would have to have defined a `allows :api_access` in your `pricing_plans.rb` file.

When the feature is disallowed, the controller will raise a `FeatureDenied` (we rescue it for you by default). You can customize the response by overriding `handle_pricing_plans_feature_denied(error)` in your `ApplicationController`:

```ruby
class ApplicationController < ActionController::Base
  private

  # Override the default 403 handler (optional)
  def handle_pricing_plans_feature_denied(error)
    # Custom HTML handling
    redirect_to upgrade_path, alert: error.message, status: :see_other
  end
end
```

You can also specify which the current billable object is **per action** by passing it to the `enforce_` callback via the `on:` param:
```ruby
before_action { enforce_api_access!(on: :current_organization) }
```

Or if you need a lambda:
```ruby
before_action { enforce_api_access!(on: -> { find_org }) }
```

Of course, this is all syntactic sugar for the primitive method, which you can also use:
```ruby
before_action { require_feature!(:api_access, billable: current_organization) }
```

#### Enforce plan limits in controllers

You can enforce limits for any action:
```ruby
before_action :enforce_projects_limit!, only: :create
```

As in feature gating, this is syntactic sugar (`enforce_<limit_key>_limit!`) that gets generated for every `limits` key in `pricing_plans.rb`. You can also use the primitive method:
```ruby
before_action { enforce_plan_limit!(:projects, billable: current_organization) }
```

And you can also pass a custom billable:
```ruby
before_action { enforce_projects_limit!(on: :current_organization) }
# or
before_action { enforce_plan_limit!(:projects, billable: current_organization) }
```

You can also specify a custom redirect path that will override the global config:
```ruby
before_action { enforce_plan_limit!(:projects, billable: current_organization, redirect_to: pricing_path) }
```

In the example aboves, the gem assumes the action to call will only create one extra project. So, if the plan limit is 5, and you're currently at 4 projects, you can still create one extra one, and the action will get called. If your action creates more than one object per call (creating multiple objects at once, importing objects in bulk etc.) you can enforce it will stay within plan limits by passing the `by:` parameter like this:
```ruby
before_action { enforce_projects_limit!(by: 10) }
```

You can also check limits inside a controller action by using `require_plan_limit!` and reading its `result`:
```ruby
def create
  result = require_plan_limit!(:products, billable: current_organization, by: 1)

  if result.blocked? # ok?, warning?, grace?, blocked?, success?
    # result.message is available:
    redirect_to pricing_path, alert: result.message, status: :see_other and return
  end

  # ...
  Product.create!(...)
  redirect_to products_path
end
```

You can define how your application responds when a limit check blocks an action by defining `handle_pricing_plans_limit_blocked` in your controller:

```ruby
class ApplicationController < ActionController::Base
  private

  def handle_pricing_plans_limit_blocked(result)
    # Default behavior (HTML): flash + redirect_to(pricing_path) if defined; else render 403
    # You can customize globally here. The Result carries rich context:
    # - result.limit_key, result.billable, result.message, result.metadata
    redirect_to(pricing_path, status: :see_other, alert: result.message)
  end
end
```

`enforce_plan_limit!` invokes this handler when `result.blocked?`, passing a `Result` enriched with `metadata[:redirect_to]` resolved via:
  1. explicit `redirect_to:` option
  2. per-controller default `self.pricing_plans_redirect_on_blocked_limit`
  3. global `config.redirect_on_blocked_limit`
  4. `pricing_path` helper if available


#### Set up a redirect when a limit is reached

You can configure a global default redirect (optional):

```ruby
# config/initializers/pricing_plans.rb
PricingPlans.configure do |config|
  config.redirect_on_blocked_limit = :pricing_path # or "/pricing" or ->(result) { pricing_path }
end
```

Or a per-controller default (optional):

```ruby
class ApplicationController < ActionController::Base
  self.pricing_plans_redirect_on_blocked_limit = :pricing_path
end
```

Redirect resolution priority:
1) `redirect_to:` option on the call
2) Per-controller `self.pricing_plans_redirect_on_blocked_limit`
3) Global `config.redirect_on_blocked_limit`
4) `pricing_path` helper (if present)
5) Fallback: render 403 (HTML or JSON)

Per-controller default accepts:
- Symbol: helper method name (e.g., `:pricing_path`)
- String: path or URL (e.g., `"/pricing"`)
- Proc: `->(result) { pricing_path }` (instance-exec'd in the controller)

Global default accepts the same types. The Proc receives the `Result` so you can branch on `limit_key`, etc.

Recommended patterns:
- Set a single global default in your initializer.
- Override per controller only if UX differs for a section.
- Use the dynamic helpers as symbols in before_action for clarity:
```ruby
before_action :enforce_projects_limit!, only: :create
before_action :enforce_api_access!
```

### Downgrades and overages

When a customer moves to a lower plan (via Stripe/Pay or manual assignment), the new plan’s limits start applying immediately. Existing resources are never auto‑deleted by the gem; instead:

- **Persistent caps** (e.g., `:projects, to: 3`): We count live rows. If the account is now over the new cap, creations will be blocked (or put into grace/warn depending on `after_limit`). Users must remediate by deleting/archiving until under cap.
- **Per‑period allowances** (e.g., `:custom_models, to: 3, per: :month`): The current window’s usage remains as is. Further creations in the same window respect the downgraded allowance and `after_limit` policy. At the next window, the allowance resets.

Use `OverageReporter` to present a clear remediation UX before or after applying a downgrade:

```ruby
report = PricingPlans::OverageReporter.report_with_message(org, :free)
if report.items.any?
  flash[:alert] = report.message
  # report.items -> [#<OverageItem limit_key:, kind: :persistent|:per_period, current_usage:, allowed:, overage:, grace_active:, grace_ends_at:>]
end
```

Example human message:
- "Over target plan on: projects: 12 > 3 (reduce by 9), custom_models: 5 > 0 (reduce by 5). Grace active — projects grace ends at 2025-01-06T12:00:00Z."

Notes:
- If you provide a `config.message_builder`, it’s used to customize copy for the `:overage_report` context.
- This reporter works regardless of whether any controller/model action has been hit; it reads live counts and current period usage.

#### Override checks

Some times you'll want to override plan limits / feature gating checks. A common use case is if you're responding to a webhook (like Stripe), you'll want to process the webhook correctly (bypassing the check) and maybe later handle the limit manually.

To do that, you can use `require_plan_limit!`. An example to proceed but mark downstream:

```ruby
def webhook_create
  result = require_plan_limit!(:projects, billable: current_organization, allow_system_override: true)

  # Your custom logic here.
  # You could proceed to create; inspect result.grace?/warning? and result.metadata[:system_override]
  Project.create!(metadata: { created_during_grace: result.grace? || result.warning?, system_override: result.metadata[:system_override] })

  head :ok
end
```

Note: model validations will still block creation even with `allow_system_override` -- it's just intended to bypass the block on controllers.

### Views (UI-neutral helpers)

We provide a small, consolidated set of data helpers that make it dead simple to build your own pricing and usage UIs.

- Pricing data for tables/cards:
  - `PricingPlans.plans` → Array of `PricingPlans::Plan` objects
  - `plan.price_label` → "Free", "$29/mo", or "Contact". If `stripe_price` is set and the Stripe gem is available, it auto-fetches the live price from Stripe. You can override or disable this (see below).
  - `PricingPlans.suggest_next_plan_for(billable, keys: [:projects, ...])`
  - `plan.free?` and `current_organization.on_free_plan?` (syntactic sugar for quick checks)
  - `plan.popular?` (alias of `highlighted?`)
  - Global helpers: `PricingPlans.highlighted_plan`, `PricingPlans.highlighted_plan_key`, `PricingPlans.popular_plan`, `PricingPlans.popular_plan_key`

- Usage/status for settings dashboards:
  - `org.limit(:projects)` → one status struct for a limit (responds to `key`, `current`, `allowed`, `percent_used`, `grace_active`, `grace_ends_at`, `blocked`, `per`)
    - Example:
      ```erb
      <% s = current_organization.limit(:projects) %>
      <div><%= s.key.to_s.humanize %>: <%= s.current %> / <%= s.allowed %> (<%= s.percent_used.round(1) %>%)</div>
      <% if s.blocked %>
        <div class="notice notice--error">Creation blocked due to plan limits</div>
      <% elsif s.grace_active %>
        <div class="notice notice--warning">Over limit — grace active until <%= s.grace_ends_at %></div>
      <% end %>
      ```
  - `org.limits(:projects, :custom_models)` → Hash of statuses (with no args, defaults to all limits on the current plan)
  - `org.limits_summary(:projects, :custom_models)` → Array of simple structs (key/current/allowed/percent_used/grace/blocked)
  - `org.limits_severity(:projects, :custom_models)` → `:ok | :warning | :grace | :blocked`
  - `org.limits_message(:projects, :custom_models)` → combined human message string (or `nil`)

  - Pure-data, English-like helpers for views (no UI components):
    - Single-limit intents on the billable:
      - `org.limit_severity(:projects)` → `:ok | :warning | :grace | :blocked`
      - `org.limit_message(:projects)` → `String | nil`
      - `org.limit_overage(:projects)` → `Integer` (0 if within)
      - `org.attention_required_for_limit?(:projects)` → `true | false` (alias for any of warning/grace/blocked)
      - `org.approaching_limit?(:projects, at: 0.9)` → `true | false` (uses highest `warn_at` if `at` omitted)
      - `org.plan_cta` → `{ text:, url: }` from current plan or global defaults
    - Top-level equivalents if you prefer: `PricingPlans.severity_for(billable, :projects)`, `message_for`,
      `overage_for`, `attention_required?(billable, :projects)`, `approaching_limit?(billable, :projects, at: 0.9)`, `cta_for(billable)`

    - One-call alert view-model (pure data, no HTML):
      - `org.limit_alert(:products)` or `PricingPlans.alert_for(org, :products)` returns:
        `{ visible?: true/false, severity:, title:, message:, overage:, cta_text:, cta_url: }`

#### Simple ERB examples

- Gate creation by ability to add one more (recommended for create buttons):

```erb
<% if current_organization.within_plan_limits?(:products) %>
  <!-- Show enabled create button -->
<% else %>
  <!-- Disabled button + hint -->
<% end %>
```

- Strict block check:

```erb
<% if current_organization.plan_blocked_for?(:products) %>
  <!-- Creation blocked by plan -->
<% end %>
```

- One-line alert decision + render:

```erb
<% if current_organization.attention_required_for_limit?(:products) %>
  <%= render "shared/plan_limit_alert", billable: current_organization, key: :products %>
<% end %>
```

#### Titles, messages, and CTA defaults

- Severity order: `:blocked` > `:grace` > `:at_limit` > `:warning` > `:ok`.
- Titles (defaults):
  - `warning`: "Approaching Limit"
  - `at_limit`: "At Limit"
  - `grace`: "Limit Exceeded (Grace Active)"
  - `blocked`: "Cannot create more resources"
- Messages come from your `config.message_builder` when present; otherwise we provide sensible defaults, e.g.:
  - Blocked: "Cannot create more <key> on your current plan."
  - Grace: "Over the <key> limit, grace active until <date>."
  - At limit: "You are at <current>/<limit> <key>. The next will exceed your plan."
  - Warning: "You have used <current>/<limit> <key>."
- CTA: we resolve CTA as follows:
  - Plan-specific CTA if set (`plan.cta_url` / `cta_text`)
  - Global defaults (`config.default_cta_url` / `default_cta_text`)
  - Fallback: if `config.redirect_on_blocked_limit` is a String path/URL, we use it as CTA URL.

    Example (you craft the UI; we give you clean data):
    ```erb
    <% if current_organization.attention_required_for_limit?(:products) %>
      <% sev = current_organization.limit_severity(:products) %>
      <% msg = current_organization.limit_message(:products) %>
      <% cta = current_organization.plan_cta %>
      <!-- Render your own banner/button using sev/msg/cta -->
    <% end %>
    ```

    Recommended ERB usage patterns:

    - Gate create actions by ability to add one more (most ergonomic for buttons):
      ```erb
      <% if current_organization.within_plan_limits?(:products) %>
        <!-- Show enabled create button -->
      <% else %>
        <!-- Disabled button + hint -->
      <% end %>
      ```

    - Check whether creation is blocked (strict block semantics):
      ```erb
      <% if current_organization.plan_blocked_for?(:products) %>
        <!-- disabled UI; creation is blocked by the plan -->
      <% end %>
      ```

    - Show an attention banner (warning/grace/blocked):
      ```erb
      <% if current_organization.attention_required_for_limit?(:products) %>
        <% sev = current_organization.limit_severity(:products) %>
        <% msg = current_organization.limit_message(:products) %>
        <% cta = current_organization.plan_cta %>
        <!-- Your banner markup here, using sev/msg/cta -->
      <% end %>
      ```

- Feature toggles (billable-centric):
  - `current_user.plan_allows?(:api_access)`
  - `current_user.plan_limit_remaining(:projects)` and `current_user.plan_limit_percent_used(:projects)`

Message customization:

- You can override copy globally via `config.message_builder`, which is used across limit checks and features. Suggested signature: `(context:, **kwargs) -> string` with contexts `:over_limit`, `:grace`, `:feature_denied`, and `:overage_report`.

#### Example: pricing table in ERB

```erb
<% PricingPlans.plans.each do |plan| %>
  <article class="card <%= 'is-current' if plan == current_user.current_pricing_plan %> <%= 'is-popular' if plan.highlighted? %>">
    <h3><%= plan.name %></h3>
    <p><%= plan.description %></p>
    <ul>
      <% plan.bullets.each do |b| %>
        <li><%= b %></li>
      <% end %>
    </ul>
    <div class="price"><%= plan.price_label %></div>
    <% if (url = plan.cta_url) %>
      <%= link_to plan.cta_text, url, class: 'btn' %>
    <% else %>
      <%= button_tag plan.cta_text, class: 'btn', disabled: true %>
    <% end %>
  </article>
<% end %>
```

To wire the button URL automatically with Pay, use a single initializer hook that generates a Checkout/Billing URL. The proc can accept `(billable, plan)`, `(plan)`, or no args — pick what fits your app conventions. You can also set a global `default_billable_resolver` so you never pass billable explicitly:

```ruby
# config/initializers/pricing_plans.rb
PricingPlans.configure do |config|
  # Resolve the current billable globally (optional)
  config.default_billable_resolver = -> { Current.organization || Current.user }

  # Auto-generate CTA URLs with Pay (optional)
  config.auto_cta_with_pay = ->(billable, plan) do
    next unless plan.stripe_price
    billable ||= Current.organization || Current.user
    return unless billable
    billable.set_payment_processor(:stripe) unless billable.payment_processor
    session = billable.payment_processor.checkout(
      mode: "subscription",
      line_items: [{ price: plan.stripe_price[:id] || plan.stripe_price }],
      success_url: Rails.application.routes.url_helpers.root_url,
      cancel_url: Rails.application.routes.url_helpers.root_url
    )
    session.url
  end
end
```

#### Example: settings usage summary (billable-centric)

```erb
<% org = current_organization %>
<% org.limits_summary(:projects, :custom_models).each do |s| %>
  <div><%= s.key.to_s.humanize %>: <%= s.current %> / <%= s.allowed %> (<%= s.percent_used.round(1) %>%)</div>
<% end %>

<% sev = org.limits_severity(:projects, :custom_models) %>
<% if sev != :ok %>
  <div class="notice notice--<%= sev %>"><%= org.limits_message(:projects, :custom_models) %></div>
<% end %>
```

That’s it: you fully control the HTML/CSS, while `pricing_plans` gives you clear, composable data.

### Semantic pricing API

Building delightful pricing UIs usually needs structured price parts (currency, amount, interval) and both monthly/yearly data. `pricing_plans` ships a semantic, UI‑agnostic API so you never parse strings in your app.

#### Value object: `PricingPlans::PriceComponents`

Structure returned by the helpers below. Pure data, no HTML:

```ruby
PricingPlans::PriceComponents = Struct.new(
  :present?,                 # boolean: price is numeric?
  :currency,                 # string currency symbol, e.g. "$", "€"
  :amount,                   # string whole amount, e.g. "29"
  :amount_cents,             # integer cents, e.g. 2900
  :interval,                 # :month | :year
  :label,                    # friendly label, e.g. "$29/mo" or "Contact"
  :monthly_equivalent_cents, # integer; = amount for monthly, or yearly/12 rounded
  keyword_init: true
)
```

#### Plan helpers (semantic pricing)

```ruby
plan.price_components(interval: :month)    # => PriceComponents
plan.monthly_price_components              # sugar for :month
plan.yearly_price_components               # sugar for :year

plan.has_interval_prices?                  # true if configured/inferred
plan.has_numeric_price?                    # true if numeric (price or stripe_price)

plan.price_label_for(:month)               # "$29/mo" (uses PriceComponents)
plan.price_label_for(:year)                # "$290/yr" or Stripe-derived

plan.monthly_price_cents                   # integer or nil
plan.yearly_price_cents                    # integer or nil
plan.monthly_price_id                      # Stripe Price ID (when available)
plan.yearly_price_id
plan.currency_symbol                       # "$" or derived from Stripe
```

Notes:

- If `stripe_price` is configured, we derive cents, currency, and interval from the Stripe Price (and cache it).
- If `price 0` (free), we return components with `present? == true`, amount 0 and the configured default currency symbol.
- If only `price_string` is set (e.g., "Contact us"), components return `present? == false`, `label == price_string`.

#### Pure-data view models

- Per‑plan:

```ruby
plan.to_view_model
# => {
#   id:, key:, name:, description:, features:, highlighted:, default:, free:,
#   currency:, monthly_price_cents:, yearly_price_cents:,
#   monthly_price_id:, yearly_price_id:,
#   price_label:, price_string:, limits: { ... }
# }
```

- All plans (preserves `PricingPlans.plans` order):

```ruby
PricingPlans.view_models # => Array<Hash>
```

#### UI helpers (pure data; no HTML opinions)

We include data‑only helpers into ActionView.

```ruby
pricing_plan_ui_data(plan)
# => {
#   monthly_price:, yearly_price:,
#   monthly_price_cents:, yearly_price_cents:,
#   monthly_price_id:, yearly_price_id:,
#   free:, label:
# }

pricing_plan_cta(plan, billable: nil, context: :marketing, current_plan: nil)
# => { text:, url:, method: :get, disabled:, reason: }
```

`pricing_plan_cta` disables the button for the current plan (text: "Current Plan"). You can add a downgrade policy (see configuration) to surface `reason` in your UI.

#### Plan comparison ergonomics (for CTAs)

```ruby
plan.current_for?(current_plan)      # boolean
plan.upgrade_from?(current_plan)     # boolean
plan.downgrade_from?(current_plan)   # boolean
plan.downgrade_blocked_reason(from: current_plan, billable: org) # string | nil
```

#### Stripe lookups and caching

- We fetch Stripe Price objects when `stripe_price` is present.
- Caching is supported via `config.price_cache` (defaults to `Rails.cache` when available).
- TTL controlled by `config.price_cache_ttl` (default 10 minutes).

Example initializer snippet:

```ruby
PricingPlans.configure do |config|
  config.price_cache = Rails.cache
  config.price_cache_ttl = 10.minutes
end
```

#### Configuration for pricing semantics

```ruby
PricingPlans.configure do |config|
  # Currency symbol when Stripe is absent
  config.default_currency_symbol = "$"

  # Cache & TTL for Stripe Price lookups
  config.price_cache = Rails.cache
  config.price_cache_ttl = 10.minutes

  # Optional hook to fully customize components
  # Signature: ->(plan, interval) { PricingPlans::PriceComponents | nil }
  config.price_components_resolver = ->(plan, interval) { nil }

  # Optional free copy used by some data helpers
  config.free_price_caption = "Forever free"

  # Default UI interval for toggles
  config.interval_default_for_ui = :month # or :year

  # Downgrade policy used by CTA ergonomics
  # Signature: ->(from:, to:, billable:) { [allowed_boolean, reason_or_nil] }
  config.downgrade_policy = ->(from:, to:, billable:) { [true, nil] }
end
```

#### Stripe price labels in `plan.price_label`

By default, if a plan has `stripe_price` configured and the `stripe` gem is present, we auto-fetch the Stripe Price and render a friendly label (e.g., `$29/mo`).

- This mirrors Pay’s use of Stripe Prices.
- To disable auto-fetching globally:

```ruby
PricingPlans.configure do |config|
  config.auto_price_labels_from_processor = false
end
```

- To fully customize rendering (e.g., caching, locale):

```ruby
PricingPlans.configure do |config|
  config.price_label_resolver = ->(plan) do
    # Build and return a string like "$29/mo" based on your own logic
  end
end
```

## Using with `pay` and/or `usage_credits`

`pricing_plans` is designed to work seamlessly with other complementary popular gems like `pay` (to handle actual subscriptions and payments), and `usage_credits` (to handle credit-like spending and refills)

These gems are related but not overlapping. They're complementary. The boundaries are clear: billing is handled in Pay; metering (ledger-like) in usage_credits.

The integration with `pay` should be seamless and is documented throughout this entire README; however, here's a brief note about using `usage_credits` alongside `pricing_plans`.

### Using `pricing_plans` with the `usage_credits` gem

In the SaaS world, pricing plans and usage credits are related in so far credits are usually a part of a pricing plan. A plan would give you, say, 100 credits a month along other features, and users would find that information usually documented in the pricing table itself.

However, for the purposes of this gem, pricing plans and usage credits are two very distinct things.

If you want to add credits to your app, you should install and configure the [usage_credits](https://github.com/rameerez/usage_credits) gem separately. In the `usage_credits` configuration, you should specify how many credits your users get with each subscription.

#### The difference between usage credits and per-period plan limits

> [!WARNING]
> Usage credits are not the same as per-period limits.

**Usage credits behave like a currency**. Per-period limits are not a currency, and shouldn't be purchaseable.

- **Usage credits** are like: "100 image-generation credits a month"
- **Per-period limits** are like: "Create up to 3 new projects a month"

Usage credits can be refilled (buy credit packs, your balance goes up), can be spent (your balance goes down). Per-period limits do not. If you intend to sell credit packs, or if the balance needs to go both up and down, you should implement usage credits, NOT per-period limits.

Some other examples of per-period limits: “1 domain change per week”, “2 exports/day”. Those are discrete allowances, not metered workloads. For classic metered workloads (API calls, image generations, tokenized compute), use credits instead.

Here's a few rules for a clean separation to help you decide when to use either gem:

`pricing_plans` handles:
  - Booleans (feature flags).
  - Persistent caps (max concurrent resources: products, seats, projects at a time).
  - Discrete per-period allowances (e.g., “3 exports / month”), with no overage purchasing.

`usage_credits` handles:
  - Metered consumption (API calls, generations, storage GB*hrs, etc.).
  - Included monthly credits via subscription plans.
  - Top-ups and pay-as-you-go.
  - Rollover/expire semantics and the entire ledger.

If a dimension is metered and you want to sell overage/top-ups, use credits only. Don’t also define a periodic limit for the same dimension in `pricing_plans`. We’ll actively lint and refuse dual definitions at boot.

#### How to show `usage_credits` in `pricing_plans`

With all that being said, in SaaS users would typically find information about plan credits in the pricing plan table, and because of that, and since `pricing_plans` should be the single source of truth for pricing plans in your Rails app, you should include how many credits your plans give in `pricing_plans.rb`:

```ruby
PricingPlans.configure do |config|
  plan :pro do
    bullets "API access", "100 credits per month"
  end
end
```

`pricing_plans` ships some ergonomics to declare and render included credits, and guardrails to keep your configuration coherent when `usage_credits` is present.

##### Declare included credits in your plans (single currency)

Plans can advertise the total credits included. This is cosmetic for pricing UI; `usage_credits` remains the source of truth for fulfillment and spending:

```ruby
PricingPlans.configure do |config|
  config.plan :free do
    price 0
    includes_credits 100
  end

  config.plan :pro do
    price 29
    includes_credits 5_000
  end
end
```

When you’re composing your UI, you can read credits via `plan.credits_included`.

> [!IMPORTANT]
> You need to keep defining operations and subscription fulfillment in your `usage_credits` initializer, declaring it in pricing_plans is purely cosmetic and for ergonomics to render pricing tables.

##### Guardrails when `usage_credits` is installed

When the `usage_credits` gem is present, we lint your configuration at boot to prevent ambiguous setups:

Collisions between credits and per‑period plan limits are disallowed: you cannot define a per‑period limit for a key that is also a `usage_credits` operation (e.g., `limits :api_calls, to: 50, per: :month`). If a dimension is metered, use credits only.

This enforces a clean separation:

- Use `usage_credits` for metered workloads you may wish to top‑up or charge PAYG for.
- Use `pricing_plans` limits for discrete allowances and feature flags (things that don’t behave like a currency).

##### No runtime coupling; single source of truth

`pricing_plans` does not spend or refill credits — that’s owned by `usage_credits`.

- Keep using `@user.spend_credits_on(:operation, ...)`, subscription fulfillment, and credit packs in `usage_credits`.
- Treat `includes_credits` here as pricing UI copy only. The single source of truth for operations, costs, fulfillment cadence, rollover/expire, and balances lives in `usage_credits`.


## Why the models?

The `pricing_plans` gem needs three new models in the schema in order to work: `Assignment`, `EnforcementState`, and `Usage`. Why are they needed?

- `PricingPlans::Assignment` allow manual plan overrides independent of billing system (or before you wire up Stripe/Pay). Great for admin toggles, trials, demos.
  - What: The arbitrary `plan_key` and a `source` label (default "manual"). Unique per billable.
  - How it’s used: `PlanResolver` checks Pay → manual assignment → default plan. You can call `assign_pricing_plan!` and `remove_pricing_plan!` on the billable.

- `PricingPlans::EnforcementState` tracks per-billable per-limit enforcement state for persistent caps and per-period allowances (grace/warnings/block state) in a race-safe way.
  - What: `exceeded_at`, `blocked_at`, last warning info, and a small JSON `data` column where we persist plan-derived parameters like grace period seconds.
  - How it’s used: When you exceed a limit, we upsert/read this row under row-level locking to start grace, compute when it ends, flip to blocked, and to ensure idempotent event emission (`on_warning`, `on_grace_start`, `on_block`).

- `PricingPlans::Usage` tracks per-period allowances (e.g., “3 projects per month”). Persistent caps don’t need a table because they are live counts.
  - What: `period_start`, `period_end`, and a monotonic `used` counter with a last-used timestamp.
  - How it’s used: On create of the metered model, we increment or upsert the usage for the current window (based on `PeriodCalculator`). Reads power `remaining`, `percent_used`, and warning thresholds.

## Testing

We use Minitest for testing. Run the test suite with `bundle exec rake test`

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`.

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/rameerez/pricing_plans. Our code of conduct is: just be nice and make your mom proud of what you do and post online.

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).
