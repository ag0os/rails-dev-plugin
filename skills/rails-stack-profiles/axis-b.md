# Axis B — Delivery

Whether the surface renders HTML or serves JSON. Two values: `html` and `api`. Each section below is self-contained — read only the one that matches the resolved axis value.

---

## html

Server-rendered views with Hotwire (Turbo + Stimulus). The app produces HTML; interactivity is progressive enhancement, not a separate frontend application.

### Beliefs

- **HTML over the wire.** Turbo Frames and Streams update the page without a client-side framework.
- **Views are first-class.** ERB templates, partials, layouts, and helpers carry the presentation.
- **Stimulus for sprinkles.** JavaScript augments server-rendered markup; it does not own the page.

### Characteristic patterns

Controller rendering HTML, with Turbo-aware responses:

```ruby
class OrdersController < ApplicationController
  def create
    @order = current_user.orders.build(order_params)

    if @order.save
      @order.confirm!
      redirect_to @order, notice: "Order placed!"
    else
      render :new, status: :unprocessable_entity
    end
  end
end
```

```erb
<%# app/views/orders/show.html.erb %>
<%= turbo_frame_tag dom_id(@order) do %>
  <h1>Order <%= @order.number %></h1>
  <p>Status: <%= @order.status %></p>
<% end %>
```

### Relevant skills

`rails-views-patterns` and `hotwire-patterns` apply on this axis value. They are inert when delivery is `api`.

### Detection signals

`app/views/**/*.erb` present; `gem "turbo-rails"`, `gem "stimulus-rails"`; `turbo_stream` responses in controllers.

---

## api

Rails as a JSON backend. No views, no Hotwire. The frontend is a separate application — a SPA, a mobile app, or third-party consumers.

### Beliefs

- **JSON in, JSON out.** No HTML rendering, no asset pipeline for app UI.
- **Versioned from day one.** Breaking changes are expensive once clients exist.
- **Explicit serialization.** Never expose ActiveRecord directly — always a serializer.
- **Contract-driven.** Response schemas and API docs are first-class artifacts.

### Characteristic patterns

API base controller:

```ruby
# app/controllers/api/v1/base_controller.rb
module Api
  module V1
    class BaseController < ActionController::API
      include ActionController::HttpAuthentication::Token::ControllerMethods

      before_action :authenticate_token!

      rescue_from ActiveRecord::RecordNotFound, with: :not_found
      rescue_from ActiveRecord::RecordInvalid, with: :unprocessable

      private

      def authenticate_token!
        authenticate_or_request_with_http_token do |token, _options|
          @current_user = User.find_by(api_token: token)
        end
      end

      def not_found
        render json: { error: "Not found" }, status: :not_found
      end

      def unprocessable(exception)
        render json: { errors: exception.record.errors.full_messages },
               status: :unprocessable_entity
      end
    end
  end
end
```

Response envelope with serializer:

```ruby
module Api
  module V1
    class OrdersController < BaseController
      def index
        orders = pagy(current_user.orders.order(created_at: :desc))
        render json: {
          data: OrderSerializer.new(orders).serialize,
          meta: pagy_metadata(orders)
        }
      end
    end
  end
end
```

### Relevant skills

`rails-api-patterns` applies on this axis value. `rails-views-patterns` and `hotwire-patterns` do not apply.

### Detection signals

App base controller is `ActionController::API`; no `app/views/` (mailer views aside); `gem "jwt"`, `gem "rack-cors"`, serializer gems with no ERB templates.
