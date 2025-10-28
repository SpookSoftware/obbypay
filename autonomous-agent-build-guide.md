# ObbyPay: Autonomous Agent Build Guide

**Purpose**: This guide enables an autonomous coding agent to build a complete Rails-based monetization platform for Obsidian plugins, similar to ExtensionPay for Chrome extensions. Each task includes clear success criteria, test commands, and validation steps.

**Project Overview**: Build a Ruby on Rails application that enables Obsidian plugin developers to monetize their plugins through Stripe, with automatic license generation, API-based validation, and minimal integration friction.

---

## Phase 0: Environment Setup & Project Initialization

### Task 0.1: Initialize Rails Project
**Objective**: Create a new Rails 7+ application with PostgreSQL database.

**Commands**:
```bash
# Check Ruby version (need 3.1+)
ruby -v

# Install Rails if needed
gem install rails

# Create new Rails app with PostgreSQL
rails new obbypay --database=postgresql --css=tailwind

# Navigate into project
cd obbypay

# Create databases
rails db:create
```

**Validation**:
- [x] `rails -v` returns version 7.0 or higher
- [x] Project directory created with standard Rails structure
- [x] `config/database.yml` configured for PostgreSQL
- [x] `rails db:create` completes without errors
- [x] `rails server` starts successfully on http://localhost:3000

**Success Criteria**: Can visit http://localhost:3000 and see Rails welcome page.

---

### Task 0.2: Install Core Dependencies
**Objective**: Add all required gems to Gemfile.

**Action**: Update `Gemfile` with:
```ruby
# Core authentication
gem 'devise'

# Stripe integration
gem 'stripe'

# Background jobs
gem 'sidekiq'
gem 'redis'

# Authorization
gem 'pundit'

# Environment variables
gem 'dotenv-rails', groups: [:development, :test]

# API utilities
gem 'rack-cors'
gem 'rack-attack'

# Testing
group :development, :test do
  gem 'rspec-rails'
  gem 'factory_bot_rails'
  gem 'faker'
end
```

**Commands**:
```bash
bundle install
rails generate rspec:install
rails generate devise:install
```

**Validation**:
- [ ] `bundle install` completes successfully
- [ ] All gems appear in `Gemfile.lock`
- [ ] `rails generate devise:install` creates config files
- [ ] `spec/` directory exists after rspec install

**Success Criteria**: No bundle errors, all gems installed.

---

### Task 0.3: Configure Environment Variables
**Objective**: Set up secure credential management.

**Action**: Create `.env` file in project root:
```bash
# Stripe Keys (get from stripe.com/test dashboard)
STRIPE_PUBLISHABLE_KEY=pk_test_your_key_here
STRIPE_SECRET_KEY=sk_test_your_key_here
STRIPE_WEBHOOK_SECRET=whsec_your_webhook_secret
STRIPE_CONNECT_CLIENT_ID=ca_your_client_id

# Database
DATABASE_URL=postgresql://localhost/obbypay_development

# Redis (for Sidekiq)
REDIS_URL=redis://localhost:6379/0

# Application
APP_HOST=http://localhost:3000
```

**Action**: Create `config/initializers/stripe.rb`:
```ruby
Rails.configuration.stripe = {
  publishable_key: ENV['STRIPE_PUBLISHABLE_KEY'],
  secret_key: ENV['STRIPE_SECRET_KEY'],
  webhook_secret: ENV['STRIPE_WEBHOOK_SECRET'],
  connect_client_id: ENV['STRIPE_CONNECT_CLIENT_ID']
}

Stripe.api_key = Rails.configuration.stripe[:secret_key]
```

**Validation**:
- [ ] `.env` file exists and contains all keys
- [ ] `.gitignore` includes `.env` (prevent committing secrets)
- [ ] `stripe.rb` initializer loads without errors
- [ ] Can access `ENV['STRIPE_SECRET_KEY']` in rails console

**Success Criteria**: `rails console` → `Stripe.api_key` returns your test key.

---

## Phase 1: Database Schema & Models

### Task 1.1: Generate User Model with Devise
**Objective**: Create developer accounts with authentication.

**Commands**:
```bash
# Generate User model with Devise
rails generate devise User

# Add custom fields to migration
```

**Action**: Edit the generated migration file (`db/migrate/*_devise_create_users.rb`):
```ruby
class DeviseCreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      ## Database authenticatable
      t.string :email,              null: false, default: ""
      t.string :encrypted_password, null: false, default: ""

      ## Recoverable
      t.string   :reset_password_token
      t.datetime :reset_password_sent_at

      ## Rememberable
      t.datetime :remember_created_at

      ## Custom fields for Stripe Connect
      t.string :stripe_account_id
      t.boolean :charges_enabled, default: false
      t.boolean :payouts_enabled, default: false
      t.string :stripe_customer_id

      t.timestamps null: false
    end

    add_index :users, :email,                unique: true
    add_index :users, :reset_password_token, unique: true
    add_index :users, :stripe_account_id,    unique: true
  end
end
```

**Commands**:
```bash
rails db:migrate
```

**Validation**:
- [ ] Migration runs without errors
- [ ] `users` table exists in database
- [ ] All columns present: email, encrypted_password, stripe_account_id, etc.
- [ ] Indexes created for email and stripe_account_id

**Test**: 
```bash
rails console
> User.create(email: 'test@example.com', password: 'password123')
> User.last
# Should return user with all fields
```

**Success Criteria**: Can create users via console with all fields.

---

### Task 1.2: Generate Plugin Model
**Objective**: Store plugin metadata and pricing information.

**Commands**:
```bash
rails generate model Plugin user:references name:string slug:string stripe_product_id:string one_time_price_id:string subscription_price_id:string one_time_amount_cents:integer subscription_amount_cents:integer currency:string trial_days:integer
```

**Action**: Edit the generated migration (`db/migrate/*_create_plugins.rb`):
```ruby
class CreatePlugins < ActiveRecord::Migration[7.0]
  def change
    create_table :plugins do |t|
      t.references :user, null: false, foreign_key: true
      t.string :name, null: false
      t.string :slug, null: false
      t.text :description
      
      # Stripe references
      t.string :stripe_product_id
      t.string :one_time_price_id
      t.string :subscription_price_id
      
      # Pricing (in cents)
      t.integer :one_time_amount_cents, default: 0
      t.integer :subscription_amount_cents, default: 0
      t.string :currency, default: 'usd'
      
      # Trial settings
      t.integer :trial_days, default: 0

      t.timestamps
    end

    add_index :plugins, :slug, unique: true
    add_index :plugins, :stripe_product_id
  end
end
```

**Action**: Update `app/models/plugin.rb`:
```ruby
class Plugin < ApplicationRecord
  belongs_to :user
  has_many :licenses, dependent: :destroy
  
  validates :name, presence: true
  validates :slug, presence: true, uniqueness: true, format: { with: /\A[a-z0-9-]+\z/ }
  validates :currency, inclusion: { in: %w[usd eur gbp] }
  
  before_validation :generate_slug, on: :create
  
  # Convert cents to dollars for display
  def one_time_price
    one_time_amount_cents / 100.0 if one_time_amount_cents
  end
  
  def subscription_price
    subscription_amount_cents / 100.0 if subscription_amount_cents
  end
  
  private
  
  def generate_slug
    self.slug ||= name.parameterize if name.present?
  end
end
```

**Action**: Update `app/models/user.rb`:
```ruby
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable

  has_many :plugins, dependent: :destroy
  has_many :licenses, through: :plugins
  
  def stripe_connected?
    stripe_account_id.present? && charges_enabled?
  end
end
```

**Commands**:
```bash
rails db:migrate
```

**Validation**:
- [ ] Migration runs successfully
- [ ] `plugins` table exists with all columns
- [ ] Foreign key constraint exists on user_id
- [ ] Slug index is unique
- [ ] Can create plugin via console

**Test**:
```bash
rails console
> user = User.first
> plugin = user.plugins.create(name: "Test Plugin", one_time_amount_cents: 1500)
> plugin.slug
# Should return "test-plugin"
> plugin.one_time_price
# Should return 15.0
```

**Success Criteria**: Plugin model validates and auto-generates slug.

---

### Task 1.3: Generate License Model
**Objective**: Store license keys and subscription status.

**Commands**:
```bash
rails generate model License plugin:references license_key:string email:string stripe_customer_id:string stripe_charge_id:string stripe_subscription_id:string status:string expires_at:datetime
```

**Action**: Edit migration (`db/migrate/*_create_licenses.rb`):
```ruby
class CreateLicenses < ActiveRecord::Migration[7.0]
  def change
    create_table :licenses do |t|
      t.references :plugin, null: false, foreign_key: true
      
      # License identification
      t.string :license_key, null: false
      t.string :email
      
      # Stripe references
      t.string :stripe_customer_id
      t.string :stripe_charge_id
      t.string :stripe_subscription_id
      t.string :stripe_checkout_session_id
      
      # Status tracking
      t.string :status, default: 'active', null: false
      t.datetime :expires_at
      
      t.timestamps
    end

    add_index :licenses, :license_key, unique: true
    add_index :licenses, :stripe_customer_id
    add_index :licenses, :stripe_subscription_id
    add_index :licenses, [:plugin_id, :email]
  end
end
```

**Action**: Update `app/models/license.rb`:
```ruby
class License < ApplicationRecord
  belongs_to :plugin
  
  validates :license_key, presence: true, uniqueness: true
  validates :status, inclusion: { in: %w[active inactive trial expired canceled] }
  
  before_validation :generate_license_key, on: :create
  
  scope :active, -> { where(status: 'active') }
  scope :valid_licenses, -> { where(status: %w[active trial]).where('expires_at IS NULL OR expires_at > ?', Time.current) }
  
  def valid?
    return false unless %w[active trial].include?(status)
    return false if expires_at.present? && expires_at < Time.current
    true
  end
  
  def expired?
    expires_at.present? && expires_at < Time.current
  end
  
  private
  
  def generate_license_key
    self.license_key ||= SecureRandom.alphanumeric(32).upcase
  end
end
```

**Commands**:
```bash
rails db:migrate
```

**Validation**:
- [ ] Migration runs successfully
- [ ] `licenses` table exists
- [ ] Foreign key to plugins exists
- [ ] License key index is unique
- [ ] Auto-generates 32-character license keys

**Test**:
```bash
rails console
> plugin = Plugin.first
> license = plugin.licenses.create(email: 'user@example.com', status: 'active')
> license.license_key
# Should return something like "A1B2C3D4E5F6G7H8I9J0K1L2M3N4O5P6"
> license.valid?
# Should return true
```

**Success Criteria**: License auto-generates unique keys and validates correctly.

---

## Phase 2: Authentication & Dashboard

### Task 2.1: Configure Devise Routes and Views
**Objective**: Set up user authentication system.

**Commands**:
```bash
rails generate devise:views
rails generate devise:controllers users
```

**Action**: Update `config/routes.rb`:
```ruby
Rails.application.routes.draw do
  devise_for :users, controllers: {
    registrations: 'users/registrations',
    sessions: 'users/sessions'
  }
  
  root 'pages#home'
  
  # Dashboard for authenticated developers
  authenticate :user do
    get '/dashboard', to: 'dashboard#index'
  end
  
  # Stripe Connect OAuth
  get '/stripe/connect', to: 'stripe_connect#new', as: :stripe_connect
  get '/stripe/callback', to: 'stripe_connect#create', as: :stripe_callback
  
  # Plugin management
  resources :plugins do
    resources :licenses, only: [:index, :create, :destroy]
  end
  
  # API endpoints for plugins
  namespace :api do
    namespace :v1 do
      post 'checkout-session', to: 'checkout#create'
      get 'licenses/validate', to: 'licenses#validate'
      get 'licenses/:key', to: 'licenses#show'
    end
  end
  
  # Stripe webhooks
  post '/webhooks/stripe', to: 'webhooks#stripe'
end
```

**Action**: Create `app/controllers/pages_controller.rb`:
```ruby
class PagesController < ApplicationController
  def home
    if user_signed_in?
      redirect_to dashboard_path
    end
  end
end
```

**Action**: Create basic landing page `app/views/pages/home.html.erb`:
```erb
<div class="max-w-4xl mx-auto px-4 py-16">
  <h1 class="text-5xl font-bold text-center mb-8">
    Monetize Your Obsidian Plugins
  </h1>
  
  <p class="text-xl text-gray-600 text-center mb-12">
    Accept payments, manage licenses, and earn revenue from your Obsidian plugins.
    Integration takes less than 30 minutes.
  </p>
  
  <div class="flex justify-center gap-4">
    <%= link_to "Get Started", new_user_registration_path, 
        class: "bg-blue-600 text-white px-8 py-3 rounded-lg text-lg font-semibold hover:bg-blue-700" %>
    <%= link_to "Sign In", new_user_session_path, 
        class: "bg-gray-200 text-gray-800 px-8 py-3 rounded-lg text-lg font-semibold hover:bg-gray-300" %>
  </div>
  
  <div class="mt-16 grid md:grid-cols-3 gap-8">
    <div class="text-center">
      <h3 class="text-xl font-bold mb-2">Zero Setup Cost</h3>
      <p class="text-gray-600">Only pay when you earn. No monthly fees or upfront costs.</p>
    </div>
    <div class="text-center">
      <h3 class="text-xl font-bold mb-2">30 Minute Integration</h3>
      <p class="text-gray-600">Drop-in SDK with simple API. Start selling in under an hour.</p>
    </div>
    <div class="text-center">
      <h3 class="text-xl font-bold mb-2">You Keep Control</h3>
      <p class="text-gray-600">Direct deposits to your Stripe account. Your customers, your data.</p>
    </div>
  </div>
</div>
```

**Validation**:
- [ ] Can access `/` (root path)
- [ ] Can access `/users/sign_up` (registration)
- [ ] Can access `/users/sign_in` (login)
- [ ] Redirects to dashboard when logged in
- [ ] Tailwind CSS styling renders correctly

**Test**:
```bash
rails server
# Visit http://localhost:3000
# Try signing up as a new user
# Should redirect to dashboard after signup
```

**Success Criteria**: Can register, login, and see landing page.

---

### Task 2.2: Create Developer Dashboard
**Objective**: Build main dashboard for developers to manage plugins.

**Action**: Create `app/controllers/dashboard_controller.rb`:
```ruby
class DashboardController < ApplicationController
  before_action :authenticate_user!
  
  def index
    @plugins = current_user.plugins.includes(:licenses)
    @total_revenue = calculate_total_revenue
    @active_licenses = current_user.licenses.valid_licenses.count
    @needs_stripe_connection = !current_user.stripe_connected?
  end
  
  private
  
  def calculate_total_revenue
    # This will be calculated from Stripe data later
    current_user.licenses.active.sum(:one_time_amount_cents) / 100.0
  end
end
```

**Action**: Create `app/views/dashboard/index.html.erb`:
```erb
<div class="max-w-7xl mx-auto px-4 py-8">
  <div class="flex justify-between items-center mb-8">
    <h1 class="text-3xl font-bold">Dashboard</h1>
    <%= link_to "Sign Out", destroy_user_session_path, data: { turbo_method: :delete },
        class: "text-gray-600 hover:text-gray-800" %>
  </div>
  
  <% if @needs_stripe_connection %>
    <div class="bg-yellow-50 border border-yellow-200 rounded-lg p-6 mb-8">
      <h3 class="text-lg font-semibold text-yellow-800 mb-2">
        Connect Your Stripe Account
      </h3>
      <p class="text-yellow-700 mb-4">
        Connect your Stripe account to start accepting payments for your plugins.
      </p>
      <%= link_to "Connect with Stripe", stripe_connect_path,
          class: "bg-blue-600 text-white px-6 py-2 rounded-lg inline-block hover:bg-blue-700" %>
    </div>
  <% end %>
  
  <!-- Stats Cards -->
  <div class="grid md:grid-cols-3 gap-6 mb-8">
    <div class="bg-white rounded-lg shadow p-6">
      <h3 class="text-gray-500 text-sm font-medium mb-2">Total Revenue</h3>
      <p class="text-3xl font-bold">$<%= @total_revenue %></p>
    </div>
    
    <div class="bg-white rounded-lg shadow p-6">
      <h3 class="text-gray-500 text-sm font-medium mb-2">Active Licenses</h3>
      <p class="text-3xl font-bold"><%= @active_licenses %></p>
    </div>
    
    <div class="bg-white rounded-lg shadow p-6">
      <h3 class="text-gray-500 text-sm font-medium mb-2">Plugins</h3>
      <p class="text-3xl font-bold"><%= @plugins.count %></p>
    </div>
  </div>
  
  <!-- Plugins List -->
  <div class="bg-white rounded-lg shadow">
    <div class="p-6 border-b flex justify-between items-center">
      <h2 class="text-xl font-semibold">Your Plugins</h2>
      <%= link_to "New Plugin", new_plugin_path,
          class: "bg-blue-600 text-white px-4 py-2 rounded-lg hover:bg-blue-700" %>
    </div>
    
    <% if @plugins.any? %>
      <div class="divide-y">
        <% @plugins.each do |plugin| %>
          <div class="p-6 hover:bg-gray-50">
            <div class="flex justify-between items-start">
              <div>
                <h3 class="text-lg font-semibold"><%= plugin.name %></h3>
                <p class="text-gray-600 text-sm">Slug: <%= plugin.slug %></p>
                <p class="text-gray-600 text-sm mt-1">
                  <%= plugin.licenses.valid_licenses.count %> active licenses
                </p>
              </div>
              <div class="flex gap-2">
                <%= link_to "View", plugin_path(plugin),
                    class: "text-blue-600 hover:text-blue-800" %>
                <%= link_to "Edit", edit_plugin_path(plugin),
                    class: "text-blue-600 hover:text-blue-800" %>
              </div>
            </div>
          </div>
        <% end %>
      </div>
    <% else %>
      <div class="p-12 text-center text-gray-500">
        <p class="mb-4">You haven't created any plugins yet.</p>
        <%= link_to "Create Your First Plugin", new_plugin_path,
            class: "bg-blue-600 text-white px-6 py-3 rounded-lg inline-block hover:bg-blue-700" %>
      </div>
    <% end %>
  </div>
</div>
```

**Validation**:
- [ ] Dashboard shows after login
- [ ] Displays Stripe connection prompt if not connected
- [ ] Shows stats cards (revenue, licenses, plugins)
- [ ] Lists all user's plugins
- [ ] "New Plugin" button works

**Test**: Login and verify dashboard displays correctly with zero plugins.

**Success Criteria**: Dashboard renders with all sections visible.

---

## Phase 3: Stripe Connect Integration

### Task 3.1: Implement Stripe Connect OAuth Flow
**Objective**: Allow developers to connect their Stripe accounts.

**Action**: Create `app/controllers/stripe_connect_controller.rb`:
```ruby
class StripeConnectController < ApplicationController
  before_action :authenticate_user!
  
  def new
    # Redirect to Stripe OAuth
    client_id = Rails.configuration.stripe[:connect_client_id]
    redirect_url = stripe_callback_url
    
    stripe_url = "https://connect.stripe.com/oauth/authorize?" + {
      response_type: 'code',
      client_id: client_id,
      scope: 'read_write',
      redirect_uri: redirect_url,
      'stripe_user[email]': current_user.email
    }.to_query
    
    redirect_to stripe_url, allow_other_host: true
  end
  
  def create
    # Handle OAuth callback
    if params[:error].present?
      redirect_to dashboard_path, alert: "Stripe connection failed: #{params[:error_description]}"
      return
    end
    
    begin
      response = Stripe::OAuth.token({
        grant_type: 'authorization_code',
        code: params[:code]
      })
      
      # Get connected account details
      account = Stripe::Account.retrieve(response.stripe_user_id)
      
      current_user.update!(
        stripe_account_id: response.stripe_user_id,
        charges_enabled: account.charges_enabled,
        payouts_enabled: account.payouts_enabled
      )
      
      redirect_to dashboard_path, notice: "Stripe account connected successfully!"
    rescue Stripe::OAuth::InvalidGrantError => e
      redirect_to dashboard_path, alert: "Invalid authorization code: #{e.message}"
    rescue => e
      redirect_to dashboard_path, alert: "Connection error: #{e.message}"
    end
  end
end
```

**Validation**:
- [ ] `/stripe/connect` redirects to Stripe OAuth page
- [ ] After OAuth, redirects back to `/stripe/callback`
- [ ] User's `stripe_account_id` gets saved
- [ ] `charges_enabled` and `payouts_enabled` flags update

**Test**:
```bash
# Manual test in browser:
# 1. Login to dashboard
# 2. Click "Connect with Stripe"
# 3. Complete Stripe Connect flow (use test mode)
# 4. Verify redirect back to dashboard
# 5. Check user in rails console has stripe_account_id
```

**Success Criteria**: Stripe account connects and saves account ID.

---

### Task 3.2: Create Stripe Product and Price on Plugin Creation
**Objective**: Automatically create Stripe Product and Prices when developer creates plugin.

**Action**: Create `app/services/stripe_product_creator.rb`:
```ruby
class StripeProductCreator
  def initialize(plugin)
    @plugin = plugin
    @user = plugin.user
  end
  
  def create_product_and_prices
    return unless @user.stripe_account_id.present?
    
    product = create_product
    @plugin.update!(stripe_product_id: product.id)
    
    # Create one-time price if amount specified
    if @plugin.one_time_amount_cents > 0
      price = create_one_time_price(product.id)
      @plugin.update!(one_time_price_id: price.id)
    end
    
    # Create subscription price if amount specified
    if @plugin.subscription_amount_cents > 0
      price = create_subscription_price(product.id)
      @plugin.update!(subscription_price_id: price.id)
    end
    
    @plugin
  end
  
  private
  
  def create_product
    Stripe::Product.create({
      name: @plugin.name,
      description: @plugin.description || "Obsidian Plugin: #{@plugin.name}",
    }, {
      stripe_account: @user.stripe_account_id
    })
  end
  
  def create_one_time_price(product_id)
    Stripe::Price.create({
      product: product_id,
      unit_amount: @plugin.one_time_amount_cents,
      currency: @plugin.currency,
    }, {
      stripe_account: @user.stripe_account_id
    })
  end
  
  def create_subscription_price(product_id)
    Stripe::Price.create({
      product: product_id,
      unit_amount: @plugin.subscription_amount_cents,
      currency: @plugin.currency,
      recurring: { interval: 'month' },
    }, {
      stripe_account: @user.stripe_account_id
    })
  end
end
```

**Action**: Create `app/controllers/plugins_controller.rb`:
```ruby
class PluginsController < ApplicationController
  before_action :authenticate_user!
  before_action :set_plugin, only: [:show, :edit, :update, :destroy]
  
  def index
    @plugins = current_user.plugins
  end
  
  def show
    @licenses = @plugin.licenses.order(created_at: :desc).limit(50)
  end
  
  def new
    @plugin = Plugin.new
  end
  
  def create
    @plugin = current_user.plugins.build(plugin_params)
    
    if @plugin.save
      # Create Stripe product and prices
      if current_user.stripe_connected?
        StripeProductCreator.new(@plugin).create_product_and_prices
      end
      
      redirect_to @plugin, notice: 'Plugin created successfully!'
    else
      render :new, status: :unprocessable_entity
    end
  end
  
  def edit
  end
  
  def update
    if @plugin.update(plugin_params)
      redirect_to @plugin, notice: 'Plugin updated successfully!'
    else
      render :edit, status: :unprocessable_entity
    end
  end
  
  def destroy
    @plugin.destroy
    redirect_to plugins_path, notice: 'Plugin deleted successfully!'
  end
  
  private
  
  def set_plugin
    @plugin = current_user.plugins.find(params[:id])
  end
  
  def plugin_params
    params.require(:plugin).permit(
      :name, :description, :one_time_amount_cents, 
      :subscription_amount_cents, :currency, :trial_days
    )
  end
end
```

**Action**: Create `app/views/plugins/new.html.erb`:
```erb
<div class="max-w-2xl mx-auto px-4 py-8">
  <h1 class="text-3xl font-bold mb-8">Create New Plugin</h1>
  
  <%= form_with(model: @plugin, class: "space-y-6") do |form| %>
    <% if @plugin.errors.any? %>
      <div class="bg-red-50 border border-red-200 rounded-lg p-4">
        <h3 class="text-red-800 font-semibold mb-2">
          <%= pluralize(@plugin.errors.count, "error") %> prohibited this plugin from being saved:
        </h3>
        <ul class="list-disc list-inside text-red-700">
          <% @plugin.errors.each do |error| %>
            <li><%= error.full_message %></li>
          <% end %>
        </ul>
      </div>
    <% end %>
    
    <div>
      <%= form.label :name, class: "block text-sm font-medium text-gray-700 mb-2" %>
      <%= form.text_field :name, class: "w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500" %>
      <p class="text-sm text-gray-500 mt-1">The name of your Obsidian plugin</p>
    </div>
    
    <div>
      <%= form.label :description, class: "block text-sm font-medium text-gray-700 mb-2" %>
      <%= form.text_area :description, rows: 4, class: "w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500" %>
    </div>
    
    <div class="border-t pt-6">
      <h3 class="text-lg font-semibold mb-4">Pricing</h3>
      
      <div class="grid md:grid-cols-2 gap-4">
        <div>
          <%= form.label :one_time_amount_cents, "One-time Price (cents)", class: "block text-sm font-medium text-gray-700 mb-2" %>
          <%= form.number_field :one_time_amount_cents, class: "w-full px-4 py-2 border border-gray-300 rounded-lg" %>
          <p class="text-sm text-gray-500 mt-1">e.g., 1500 = $15.00</p>
        </div>
        
        <div>
          <%= form.label :subscription_amount_cents, "Monthly Subscription (cents)", class: "block text-sm font-medium text-gray-700 mb-2" %>
          <%= form.number_field :subscription_amount_cents, class: "w-full px-4 py-2 border border-gray-300 rounded-lg" %>
          <p class="text-sm text-gray-500 mt-1">e.g., 500 = $5.00/month</p>
        </div>
      </div>
    </div>
    
    <div>
      <%= form.label :trial_days, "Free Trial (days)", class: "block text-sm font-medium text-gray-700 mb-2" %>
      <%= form.number_field :trial_days, class: "w-full px-4 py-2 border border-gray-300 rounded-lg" %>
      <p class="text-sm text-gray-500 mt-1">Optional: Offer a free trial period</p>
    </div>
    
    <div class="flex gap-4">
      <%= form.submit "Create Plugin", class: "bg-blue-600 text-white px-6 py-3 rounded-lg font-semibold hover:bg-blue-700" %>
      <%= link_to "Cancel", dashboard_path, class: "text-gray-600 hover:text-gray-800 px-6 py-3" %>
    </div>
  <% end %>
</div>
```

**Action**: Create `app/views/plugins/show.html.erb`:
```erb
<div class="max-w-4xl mx-auto px-4 py-8">
  <%= link_to "← Back to Dashboard", dashboard_path, class: "text-blue-600 hover:text-blue-800 mb-4 inline-block" %>
  
  <div class="bg-white rounded-lg shadow p-6 mb-6">
    <div class="flex justify-between items-start mb-6">
      <div>
        <h1 class="text-3xl font-bold mb-2"><%= @plugin.name %></h1>
        <p class="text-gray-600"><%= @plugin.description %></p>
      </div>
      <%= link_to "Edit", edit_plugin_path(@plugin), 
          class: "text-blue-600 hover:text-blue-800" %>
    </div>
    
    <div class="grid md:grid-cols-3 gap-4 mb-6">
      <div>
        <p class="text-sm text-gray-500">Slug</p>
        <p class="font-mono text-sm"><%= @plugin.slug %></p>
      </div>
      <div>
        <p class="text-sm text-gray-500">One-time Price</p>
        <p class="font-semibold">
          <% if @plugin.one_time_amount_cents > 0 %>
            $<%= @plugin.one_time_price %>
          <% else %>
            Not set
          <% end %>
        </p>
      </div>
      <div>
        <p class="text-sm text-gray-500">Subscription</p>
        <p class="font-semibold">
          <% if @plugin.subscription_amount_cents > 0 %>
            $<%= @plugin.subscription_price %>/month
          <% else %>
            Not set
          <% end %>
        </p>
      </div>
    </div>
    
    <div class="border-t pt-6">
      <h3 class="font-semibold mb-2">Integration Code</h3>
      <p class="text-sm text-gray-600 mb-4">
        Add this to your Obsidian plugin's main.ts file:
      </p>
      <pre class="bg-gray-50 p-4 rounded-lg overflow-x-auto text-sm"><code>import { ObsidianMonetization } from 'obsidian-monetization-sdk';

const monetization = new ObsidianMonetization('<%= @plugin.slug %>');

// Check if user has valid license
const isLicensed = await monetization.checkLicense();

if (!isLicensed) {
  // Open payment page
  await monetization.openPurchasePage();
}</code></pre>
    </div>
  </div>
  
  <div class="bg-white rounded-lg shadow">
    <div class="p-6 border-b">
      <h2 class="text-xl font-semibold">Recent Licenses</h2>
    </div>
    
    <% if @licenses.any? %>
      <div class="divide-y">
        <% @licenses.each do |license| %>
          <div class="p-6">
            <div class="flex justify-between items-start">
              <div>
                <p class="font-mono text-sm text-gray-600"><%= license.license_key %></p>
                <p class="text-sm text-gray-500 mt-1">
                  <%= license.email %> • <%= license.status.titleize %>
                </p>
                <p class="text-xs text-gray-400 mt-1">
                  Created <%= time_ago_in_words(license.created_at) %> ago
                </p>
              </div>
              <span class="px-3 py-1 text-sm rounded-full <%= license.status == 'active' ? 'bg-green-100 text-green-800' : 'bg-gray-100 text-gray-800' %>">
                <%= license.status.titleize %>
              </span>
            </div>
          </div>
        <% end %>
      </div>
    <% else %>
      <div class="p-12 text-center text-gray-500">
        <p>No licenses yet. Share your plugin to start earning!</p>
      </div>
    <% end %>
  </div>
</div>
```

**Validation**:
- [ ] Can create new plugin via form
- [ ] Stripe Product created automatically
- [ ] Stripe Prices created based on amounts entered
- [ ] Plugin slug auto-generated from name
- [ ] Integration code displayed on plugin show page

**Test**:
```bash
# In browser:
# 1. Login and connect Stripe
# 2. Create new plugin with name "Test Plugin" and $15.00 one-time price (1500 cents)
# 3. Check Stripe dashboard for Product creation
# 4. Verify plugin show page displays integration code

# In rails console:
> plugin = Plugin.last
> plugin.stripe_product_id
# Should have Stripe product ID (prod_...)
> plugin.one_time_price_id
# Should have Stripe price ID (price_...)
```

**Success Criteria**: Plugin creation auto-creates Stripe Product and Prices.

---

## Phase 4: Checkout & License Generation

### Task 4.1: Create Checkout Session API
**Objective**: API endpoint for plugins to initiate Stripe Checkout.

**Action**: Create `app/controllers/api/v1/checkout_controller.rb`:
```ruby
module Api
  module V1
    class CheckoutController < ApplicationController
      skip_before_action :verify_authenticity_token
      
      def create
        plugin = Plugin.find_by(slug: params[:plugin_slug])
        
        if plugin.nil?
          render json: { error: 'Plugin not found' }, status: :not_found
          return
        end
        
        price_id = determine_price_id(plugin, params[:plan_type])
        
        if price_id.nil?
          render json: { error: 'Invalid plan type or price not configured' }, status: :bad_request
          return
        end
        
        session = create_stripe_checkout_session(plugin, price_id, params[:customer_email])
        
        render json: { 
          session_url: session.url,
          session_id: session.id
        }
      rescue Stripe::StripeError => e
        render json: { error: e.message }, status: :unprocessable_entity
      end
      
      private
      
      def determine_price_id(plugin, plan_type)
        case plan_type
        when 'one_time'
          plugin.one_time_price_id
        when 'subscription'
          plugin.subscription_price_id
        else
          plugin.one_time_price_id || plugin.subscription_price_id
        end
      end
      
      def create_stripe_checkout_session(plugin, price_id, customer_email)
        session_params = {
          payment_method_types: ['card'],
          line_items: [{
            price: price_id,
            quantity: 1,
          }],
          mode: price_id == plugin.subscription_price_id ? 'subscription' : 'payment',
          success_url: "#{ENV['APP_HOST']}/purchase/success?session_id={CHECKOUT_SESSION_ID}",
          cancel_url: "#{ENV['APP_HOST']}/purchase/canceled",
          metadata: {
            plugin_id: plugin.id,
            plugin_slug: plugin.slug
          }
        }
        
        # Add trial if configured
        if price_id == plugin.subscription_price_id && plugin.trial_days > 0
          session_params[:subscription_data] = {
            trial_period_days: plugin.trial_days
          }
        end
        
        # Add customer email if provided
        session_params[:customer_email] = customer_email if customer_email.present?
        
        Stripe::Checkout::Session.create(
          session_params,
          { stripe_account: plugin.user.stripe_account_id }
        )
      end
    end
  end
end
```

**Action**: Update routes to add success/cancel pages:
```ruby
# In config/routes.rb, add:
get '/purchase/success', to: 'purchase#success'
get '/purchase/canceled', to: 'purchase#canceled'
```

**Action**: Create `app/controllers/purchase_controller.rb`:
```ruby
class PurchaseController < ApplicationController
  def success
    @session_id = params[:session_id]
    # License will be created via webhook, so just show success message
  end
  
  def canceled
    # User canceled payment
  end
end
```

**Action**: Create `app/views/purchase/success.html.erb`:
```erb
<div class="max-w-2xl mx-auto px-4 py-16 text-center">
  <div class="bg-green-50 rounded-full w-16 h-16 flex items-center justify-center mx-auto mb-6">
    <svg class="w-8 h-8 text-green-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7"></path>
    </svg>
  </div>
  
  <h1 class="text-3xl font-bold mb-4">Purchase Successful!</h1>
  <p class="text-gray-600 mb-8">
    Your license key has been sent to your email address.
    You can now activate the premium features in your Obsidian plugin.
  </p>
  
  <p class="text-sm text-gray-500">
    You can close this window and return to Obsidian.
  </p>
</div>
```

**Validation**:
- [ ] POST to `/api/v1/checkout-session` creates Stripe Checkout Session
- [ ] Returns session URL in JSON response
- [ ] Session includes correct price and metadata
- [ ] Success URL redirects to purchase success page

**Test**:
```bash
# Using curl or Postman:
curl -X POST http://localhost:3000/api/v1/checkout-session \
  -H "Content-Type: application/json" \
  -d '{"plugin_slug":"test-plugin","plan_type":"one_time","customer_email":"test@example.com"}'

# Should return:
# {"session_url":"https://checkout.stripe.com/c/pay/cs_test_...","session_id":"cs_test_..."}

# Test in browser:
# 1. Copy the session_url
# 2. Paste in browser
# 3. Complete test payment (use card 4242 4242 4242 4242)
# 4. Should redirect to success page
```

**Success Criteria**: API creates checkout sessions and payments complete.

---

### Task 4.2: Implement Webhook Handler for License Creation
**Objective**: Automatically create licenses when payments succeed.

**Action**: Create `app/controllers/webhooks_controller.rb`:
```ruby
class WebhooksController < ApplicationController
  skip_before_action :verify_authenticity_token
  
  def stripe
    payload = request.body.read
    sig_header = request.env['HTTP_STRIPE_SIGNATURE']
    
    begin
      event = Stripe::Webhook.construct_event(
        payload, sig_header, Rails.configuration.stripe[:webhook_secret]
      )
    rescue JSON::ParserError => e
      render json: { error: 'Invalid payload' }, status: :bad_request
      return
    rescue Stripe::SignatureVerificationError => e
      render json: { error: 'Invalid signature' }, status: :bad_request
      return
    end
    
    # Handle the event
    case event.type
    when 'checkout.session.completed'
      handle_checkout_completed(event.data.object)
    when 'customer.subscription.updated'
      handle_subscription_updated(event.data.object)
    when 'customer.subscription.deleted'
      handle_subscription_deleted(event.data.object)
    when 'invoice.payment_succeeded'
      handle_invoice_paid(event.data.object)
    when 'invoice.payment_failed'
      handle_invoice_failed(event.data.object)
    end
    
    render json: { status: 'success' }
  end
  
  private
  
  def handle_checkout_completed(session)
    plugin_id = session.metadata['plugin_id']
    plugin = Plugin.find_by(id: plugin_id)
    return unless plugin
    
    # Retrieve full session to get customer details
    full_session = Stripe::Checkout::Session.retrieve(
      { id: session.id, expand: ['customer'] },
      { stripe_account: plugin.user.stripe_account_id }
    )
    
    customer_email = full_session.customer_details&.email || full_session.customer&.email
    
    # Determine if subscription or one-time
    if full_session.mode == 'subscription'
      create_subscription_license(plugin, full_session, customer_email)
    else
      create_one_time_license(plugin, full_session, customer_email)
    end
  end
  
  def create_one_time_license(plugin, session, email)
    license = plugin.licenses.create!(
      email: email,
      stripe_customer_id: session.customer,
      stripe_charge_id: session.payment_intent,
      stripe_checkout_session_id: session.id,
      status: 'active'
    )
    
    # Send email with license key
    LicenseMailer.license_created(license).deliver_later
    
    license
  end
  
  def create_subscription_license(plugin, session, email)
    subscription_id = session.subscription
    
    # Check if already exists
    existing = plugin.licenses.find_by(stripe_subscription_id: subscription_id)
    return existing if existing
    
    # Retrieve subscription to check trial status
    subscription = Stripe::Subscription.retrieve(
      subscription_id,
      { stripe_account: plugin.user.stripe_account_id }
    )
    
    status = subscription.status == 'trialing' ? 'trial' : 'active'
    
    license = plugin.licenses.create!(
      email: email,
      stripe_customer_id: session.customer,
      stripe_subscription_id: subscription_id,
      stripe_checkout_session_id: session.id,
      status: status,
      expires_at: subscription.trial_end ? Time.at(subscription.trial_end) : nil
    )
    
    LicenseMailer.license_created(license).deliver_later
    
    license
  end
  
  def handle_subscription_updated(subscription)
    license = License.find_by(stripe_subscription_id: subscription.id)
    return unless license
    
    # Update status based on subscription status
    new_status = case subscription.status
    when 'active' then 'active'
    when 'trialing' then 'trial'
    when 'past_due' then 'inactive'
    when 'canceled', 'unpaid' then 'canceled'
    else 'inactive'
    end
    
    license.update!(
      status: new_status,
      expires_at: subscription.current_period_end ? Time.at(subscription.current_period_end) : nil
    )
  end
  
  def handle_subscription_deleted(subscription)
    license = License.find_by(stripe_subscription_id: subscription.id)
    return unless license
    
    license.update!(status: 'canceled')
  end
  
  def handle_invoice_paid(invoice)
    # Renewal payment succeeded
    return unless invoice.subscription
    
    license = License.find_by(stripe_subscription_id: invoice.subscription)
    return unless license
    
    license.update!(status: 'active')
  end
  
  def handle_invoice_failed(invoice)
    # Payment failed
    return unless invoice.subscription
    
    license = License.find_by(stripe_subscription_id: invoice.subscription)
    return unless license
    
    license.update!(status: 'inactive')
  end
end
```

**Action**: Create email mailer `app/mailers/license_mailer.rb`:
```ruby
class LicenseMailer < ApplicationMailer
  default from: 'noreply@obbypay.com'
  
  def license_created(license)
    @license = license
    @plugin = license.plugin
    
    mail(
      to: @license.email,
      subject: "Your #{@plugin.name} License Key"
    )
  end
end
```

**Action**: Create email view `app/views/license_mailer/license_created.html.erb`:
```erb
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <style>
    body { font-family: Arial, sans-serif; line-height: 1.6; color: #333; }
    .container { max-width: 600px; margin: 0 auto; padding: 20px; }
    .header { background: #4F46E5; color: white; padding: 20px; text-align: center; }
    .content { background: #f9fafb; padding: 30px; }
    .license-key { background: white; border: 2px solid #4F46E5; padding: 15px; font-family: monospace; font-size: 16px; text-align: center; margin: 20px 0; }
    .footer { text-align: center; padding: 20px; color: #666; font-size: 12px; }
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <h1>Thank You for Your Purchase!</h1>
    </div>
    
    <div class="content">
      <p>Your purchase of <strong><%= @plugin.name %></strong> was successful.</p>
      
      <p>Here is your license key:</p>
      
      <div class="license-key">
        <%= @license.license_key %>
      </div>
      
      <p><strong>To activate your license:</strong></p>
      <ol>
        <li>Open Obsidian</li>
        <li>Go to Settings → Community Plugins</li>
        <li>Find <%= @plugin.name %> in your installed plugins</li>
        <li>Click the settings icon</li>
        <li>Enter your license key</li>
      </ol>
      
      <p>If you have any questions or need support, please contact the plugin developer.</p>
    </div>
    
    <div class="footer">
      <p>This email was sent because you purchased a license for an Obsidian plugin.</p>
    </div>
  </div>
</body>
</html>
```

**Validation**:
- [ ] Webhook endpoint accepts POST requests at `/webhooks/stripe`
- [ ] Verifies Stripe webhook signature
- [ ] Creates license on `checkout.session.completed` event
- [ ] Sends email with license key
- [ ] Updates license status on subscription changes

**Test**:
```bash
# Set up webhook in Stripe Dashboard:
# 1. Go to stripe.com/test/webhooks
# 2. Click "Add endpoint"
# 3. URL: http://localhost:3000/webhooks/stripe (use ngrok for local testing)
# 4. Events to send: checkout.session.completed, customer.subscription.updated, etc.
# 5. Copy webhook signing secret to .env as STRIPE_WEBHOOK_SECRET

# Use Stripe CLI to test webhooks locally:
stripe listen --forward-to localhost:3000/webhooks/stripe

# Trigger a test event:
stripe trigger checkout.session.completed

# Check rails console for license creation:
rails console
> License.last
# Should show newly created license with key
```

**Success Criteria**: Webhook creates licenses automatically on payment.

---

## Phase 5: License Validation API

### Task 5.1: Create License Validation Endpoint
**Objective**: API for plugins to check if a license key is valid.

**Action**: Create `app/controllers/api/v1/licenses_controller.rb`:
```ruby
module Api
  module V1
    class LicensesController < ApplicationController
      skip_before_action :verify_authenticity_token
      before_action :check_rate_limit, only: [:validate]
      
      def validate
        plugin_slug = params[:plugin_slug]
        license_key = params[:license_key]
        
        if plugin_slug.blank? || license_key.blank?
          render json: { 
            valid: false, 
            error: 'Missing plugin_slug or license_key' 
          }, status: :bad_request
          return
        end
        
        plugin = Plugin.find_by(slug: plugin_slug)
        
        if plugin.nil?
          render json: { 
            valid: false, 
            error: 'Plugin not found' 
          }, status: :not_found
          return
        end
        
        license = plugin.licenses.find_by(license_key: license_key)
        
        if license.nil?
          render json: { 
            valid: false, 
            error: 'License not found' 
          }
          return
        end
        
        # Check if license is valid
        if license.valid?
          render json: {
            valid: true,
            status: license.status,
            email: license.email,
            expires_at: license.expires_at,
            plugin_name: plugin.name
          }
        else
          render json: {
            valid: false,
            status: license.status,
            reason: license.expired? ? 'expired' : 'inactive'
          }
        end
      end
      
      def show
        license = License.find_by(license_key: params[:key])
        
        if license.nil?
          render json: { error: 'License not found' }, status: :not_found
          return
        end
        
        render json: {
          license_key: license.license_key,
          status: license.status,
          valid: license.valid?,
          plugin_name: license.plugin.name,
          email: license.email,
          expires_at: license.expires_at,
          created_at: license.created_at
        }
      end
      
      private
      
      def check_rate_limit
        # Simple rate limiting: max 100 requests per hour per IP
        # In production, use Redis-based rate limiting with Rack::Attack
        cache_key = "license_validation:#{request.remote_ip}"
        count = Rails.cache.read(cache_key) || 0
        
        if count >= 100
          render json: { error: 'Rate limit exceeded' }, status: :too_many_requests
          return
        end
        
        Rails.cache.write(cache_key, count + 1, expires_in: 1.hour)
      end
    end
  end
end
```

**Action**: Configure rate limiting with Rack::Attack. Create `config/initializers/rack_attack.rb`:
```ruby
class Rack::Attack
  # Allow 100 requests per hour per IP for license validation
  throttle('api/license/validate', limit: 100, period: 1.hour) do |req|
    if req.path == '/api/v1/licenses/validate' && req.get?
      req.ip
    end
  end
  
  # Allow 10 requests per minute for checkout session creation
  throttle('api/checkout/create', limit: 10, period: 1.minute) do |req|
    if req.path == '/api/v1/checkout-session' && req.post?
      req.ip
    end
  end
end

# Enable Rack::Attack
Rails.application.config.middleware.use Rack::Attack
```

**Validation**:
- [ ] GET `/api/v1/licenses/validate?plugin_slug=test-plugin&license_key=ABC123` returns validation result
- [ ] Returns `valid: true` for active licenses
- [ ] Returns `valid: false` for expired/inactive licenses
- [ ] Returns 404 for non-existent plugins
- [ ] Rate limiting kicks in after 100 requests per hour

**Test**:
```bash
# Create a test license first:
rails console
> plugin = Plugin.first
> license = plugin.licenses.create!(email: 'test@example.com', status: 'active')
> license.license_key
# Copy the license key

# Test validation API:
curl "http://localhost:3000/api/v1/licenses/validate?plugin_slug=test-plugin&license_key=PASTE_KEY_HERE"

# Should return:
# {"valid":true,"status":"active","email":"test@example.com","expires_at":null,"plugin_name":"Test Plugin"}

# Test invalid key:
curl "http://localhost:3000/api/v1/licenses/validate?plugin_slug=test-plugin&license_key=INVALID"

# Should return:
# {"valid":false,"error":"License not found"}
```

**Success Criteria**: License validation API works correctly with rate limiting.

---

## Phase 6: Plugin SDK (JavaScript/TypeScript)

### Task 6.1: Create Plugin SDK
**Objective**: Build a simple JavaScript library that Obsidian plugins can use.

**Action**: Create `sdk/` directory in project root:
```bash
mkdir -p sdk
cd sdk
npm init -y
npm install --save-dev typescript @types/node
```

**Action**: Create `sdk/tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2018",
    "module": "commonjs",
    "lib": ["ES2018"],
    "declaration": true,
    "outDir": "./dist",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**Action**: Create `sdk/src/index.ts`:
```typescript
export class ObsidianMonetization {
  private pluginSlug: string;
  private apiBaseUrl: string;
  private licenseKey: string | null;
  
  constructor(pluginSlug: string, options: { apiBaseUrl?: string } = {}) {
    this.pluginSlug = pluginSlug;
    this.apiBaseUrl = options.apiBaseUrl || 'https://obbypay.com';
    this.licenseKey = null;
  }
  
  /**
   * Set the license key (typically from plugin settings)
   */
  setLicenseKey(key: string): void {
    this.licenseKey = key;
  }
  
  /**
   * Check if the stored license key is valid
   */
  async checkLicense(): Promise<boolean> {
    if (!this.licenseKey) {
      return false;
    }
    
    try {
      const response = await fetch(
        `${this.apiBaseUrl}/api/v1/licenses/validate?` +
        `plugin_slug=${encodeURIComponent(this.pluginSlug)}&` +
        `license_key=${encodeURIComponent(this.licenseKey)}`
      );
      
      const data = await response.json();
      return data.valid === true;
    } catch (error) {
      console.error('License validation error:', error);
      return false;
    }
  }
  
  /**
   * Get detailed license information
   */
  async getLicenseInfo(): Promise<any> {
    if (!this.licenseKey) {
      return null;
    }
    
    try {
      const response = await fetch(
        `${this.apiBaseUrl}/api/v1/licenses/validate?` +
        `plugin_slug=${encodeURIComponent(this.pluginSlug)}&` +
        `license_key=${encodeURIComponent(this.licenseKey)}`
      );
      
      return await response.json();
    } catch (error) {
      console.error('License info error:', error);
      return null;
    }
  }
  
  /**
   * Open the purchase page in the user's browser
   */
  async openPurchasePage(options: {
    planType?: 'one_time' | 'subscription';
    customerEmail?: string;
  } = {}): Promise<void> {
    try {
      const response = await fetch(`${this.apiBaseUrl}/api/v1/checkout-session`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          plugin_slug: this.pluginSlug,
          plan_type: options.planType || 'one_time',
          customer_email: options.customerEmail,
        }),
      });
      
      const data = await response.json();
      
      if (data.session_url) {
        // Open in external browser
        if (typeof window !== 'undefined') {
          window.open(data.session_url, '_blank');
        }
      }
    } catch (error) {
      console.error('Failed to create checkout session:', error);
      throw error;
    }
  }
}
```

**Action**: Create `sdk/package.json` scripts:
```json
{
  "name": "obsidian-monetization-sdk",
  "version": "1.0.0",
  "description": "SDK for monetizing Obsidian plugins",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "prepublishOnly": "npm run build"
  },
  "keywords": ["obsidian", "plugin", "monetization", "licensing"],
  "author": "Your Name",
  "license": "MIT"
}
```

**Action**: Build the SDK:
```bash
cd sdk
npm run build
```

**Action**: Create usage example in `sdk/README.md`:
```markdown
# Obsidian Monetization SDK

Simple SDK for adding payments and licensing to your Obsidian plugins.

## Installation

```bash
npm install obsidian-monetization-sdk
```

## Usage

```typescript
import { ObsidianMonetization } from 'obsidian-monetization-sdk';

// Initialize with your plugin slug
const monetization = new ObsidianMonetization('my-plugin-slug');

// Set license key from plugin settings
monetization.setLicenseKey(this.settings.licenseKey);

// Check if license is valid
const isLicensed = await monetization.checkLicense();

if (!isLicensed) {
  // Show upgrade prompt
  new Notice('Please purchase a license to use premium features');
  
  // Open purchase page
  await monetization.openPurchasePage({
    planType: 'one_time', // or 'subscription'
    customerEmail: 'user@example.com' // optional
  });
}

// Use in your plugin
if (isLicensed) {
  // Enable premium features
  this.enablePremiumFeatures();
}
```

## API Reference

### `new ObsidianMonetization(pluginSlug, options)`

Creates a new monetization instance.

- `pluginSlug`: Your plugin's unique slug
- `options.apiBaseUrl`: Optional custom API URL (defaults to production)

### `setLicenseKey(key: string): void`

Sets the license key to validate.

### `async checkLicense(): Promise<boolean>`

Checks if the current license key is valid.

### `async getLicenseInfo(): Promise<object>`

Gets detailed information about the license.

### `async openPurchasePage(options): Promise<void>`

Opens the Stripe Checkout page in the user's browser.

Options:
- `planType`: 'one_time' or 'subscription'
- `customerEmail`: Pre-fill customer email (optional)
```

**Validation**:
- [ ] TypeScript compiles without errors
- [ ] `dist/` folder contains compiled JavaScript
- [ ] `dist/index.d.ts` contains type definitions
- [ ] README explains usage clearly

**Success Criteria**: SDK builds successfully and exports all methods.

---

## Phase 7: Testing & Deployment

### Task 7.1: Write API Tests
**Objective**: Ensure all API endpoints work correctly.

**Action**: Create `spec/requests/api/v1/checkout_spec.rb`:
```ruby
require 'rails_helper'

RSpec.describe 'API::V1::Checkout', type: :request do
  let(:user) { create(:user, stripe_account_id: 'acct_test123', charges_enabled: true) }
  let(:plugin) { create(:plugin, user: user, one_time_amount_cents: 1500, one_time_price_id: 'price_test123') }
  
  describe 'POST /api/v1/checkout-session' do
    it 'creates a checkout session' do
      post '/api/v1/checkout-session', params: {
        plugin_slug: plugin.slug,
        plan_type: 'one_time',
        customer_email: 'test@example.com'
      }, as: :json
      
      expect(response).to have_http_status(:success)
      json = JSON.parse(response.body)
      expect(json['session_url']).to be_present
      expect(json['session_id']).to start_with('cs_')
    end
    
    it 'returns 404 for non-existent plugin' do
      post '/api/v1/checkout-session', params: {
        plugin_slug: 'nonexistent',
        plan_type: 'one_time'
      }, as: :json
      
      expect(response).to have_http_status(:not_found)
    end
  end
end
```

**Action**: Create `spec/requests/api/v1/licenses_spec.rb`:
```ruby
require 'rails_helper'

RSpec.describe 'API::V1::Licenses', type: :request do
  let(:user) { create(:user) }
  let(:plugin) { create(:plugin, user: user) }
  let(:license) { create(:license, plugin: plugin, status: 'active') }
  
  describe 'GET /api/v1/licenses/validate' do
    it 'validates an active license' do
      get '/api/v1/licenses/validate', params: {
        plugin_slug: plugin.slug,
        license_key: license.license_key
      }
      
      expect(response).to have_http_status(:success)
      json = JSON.parse(response.body)
      expect(json['valid']).to be true
      expect(json['status']).to eq('active')
    end
    
    it 'returns invalid for expired license' do
      expired_license = create(:license, plugin: plugin, status: 'expired')
      
      get '/api/v1/licenses/validate', params: {
        plugin_slug: plugin.slug,
        license_key: expired_license.license_key
      }
      
      json = JSON.parse(response.body)
      expect(json['valid']).to be false
    end
    
    it 'returns error for missing license_key' do
      get '/api/v1/licenses/validate', params: {
        plugin_slug: plugin.slug
      }
      
      expect(response).to have_http_status(:bad_request)
    end
  end
end
```

**Action**: Create factories `spec/factories.rb`:
```ruby
FactoryBot.define do
  factory :user do
    email { Faker::Internet.email }
    password { 'password123' }
  end
  
  factory :plugin do
    user
    name { Faker::App.name }
    one_time_amount_cents { 1500 }
    currency { 'usd' }
  end
  
  factory :license do
    plugin
    email { Faker::Internet.email }
    status { 'active' }
  end
end
```

**Commands**:
```bash
# Run tests
bundle exec rspec

# Check test coverage
bundle exec rspec --format documentation
```

**Validation**:
- [ ] All tests pass
- [ ] API endpoints tested for success and failure cases
- [ ] Edge cases covered (missing params, invalid data)

**Success Criteria**: Test suite passes with green results.

---

### Task 7.2: Deploy to Production
**Objective**: Get the application running on a production server.

**Action**: Configure for deployment (choose Heroku, Render, or Fly.io):

**Heroku Deployment**:
```bash
# Install Heroku CLI
# https://devcenter.heroku.com/articles/heroku-cli

# Login
heroku login

# Create app
heroku create obbypay

# Add PostgreSQL
heroku addons:create heroku-postgresql:mini

# Add Redis for Sidekiq
heroku addons:create heroku-redis:mini

# Set environment variables
heroku config:set STRIPE_PUBLISHABLE_KEY=pk_live_your_key
heroku config:set STRIPE_SECRET_KEY=sk_live_your_key
heroku config:set STRIPE_WEBHOOK_SECRET=whsec_your_secret
heroku config:set STRIPE_CONNECT_CLIENT_ID=ca_your_client_id
heroku config:set APP_HOST=https://obbypay.herokuapp.com

# Deploy
git push heroku main

# Run migrations
heroku run rails db:migrate

# Open app
heroku open
```

**Action**: Configure Procfile for Heroku:
```
web: bundle exec puma -C config/puma.rb
worker: bundle exec sidekiq -C config/sidekiq.yml
```

**Action**: Set up Stripe webhook in production:
```
1. Go to stripe.com/webhooks
2. Add endpoint: https://your-app.com/webhooks/stripe
3. Select events: checkout.session.completed, customer.subscription.*, invoice.*
4. Copy webhook signing secret to STRIPE_WEBHOOK_SECRET env var
```

**Action**: Configure custom domain (optional):
```bash
# Add custom domain
heroku domains:add obbypay.com

# Get DNS target
heroku domains

# Update DNS records at your domain registrar
# Add CNAME record pointing to Heroku DNS target
```

**Validation**:
- [ ] Application deployed successfully
- [ ] Database migrations run
- [ ] Environment variables set correctly
- [ ] Can access application at production URL
- [ ] Stripe webhooks configured
- [ ] SSL certificate active (HTTPS)

**Test Production**:
```bash
# Test API endpoint
curl https://your-app.com/api/v1/licenses/validate?plugin_slug=test&license_key=test

# Create test plugin in production
# Complete test purchase
# Verify webhook creates license
```

**Success Criteria**: Application fully functional in production.

---

## Phase 8: Documentation & Launch Prep

### Task 8.1: Create Developer Documentation
**Objective**: Comprehensive docs for plugin developers to integrate.

**Action**: Create `docs/` directory and `docs/getting-started.md`:
```markdown
# Getting Started with Obsidian Plugin Monetization

## Overview

This platform enables you to monetize your Obsidian plugins with just a few lines of code. No backend required, no payment processing complexity—just simple integration and start earning.

## Step 1: Create Account & Connect Stripe

1. Sign up at [obbypay.com](https://obbypay.com)
2. Click "Connect with Stripe" to link your Stripe account
3. Complete Stripe onboarding (takes 5-10 minutes)

## Step 2: Create Your Plugin

1. Click "New Plugin" in the dashboard
2. Enter your plugin name (e.g., "My Awesome Plugin")
3. Set pricing:
   - **One-time purchase**: Enter amount in cents (e.g., 1500 = $15)
   - **Monthly subscription**: Enter amount in cents (e.g., 500 = $5/month)
   - **Free trial**: Optional trial period in days
4. Click "Create Plugin"
5. Copy your plugin slug (e.g., `my-awesome-plugin`)

## Step 3: Install SDK in Your Plugin

```bash
npm install obsidian-monetization-sdk
```

## Step 4: Add Integration Code

### 4.1: Add License Key Setting

In your plugin's settings:

```typescript
interface MyPluginSettings {
  licenseKey: string;
  // ... other settings
}

const DEFAULT_SETTINGS: MyPluginSettings = {
  licenseKey: '',
}

// In your settings tab:
new Setting(containerEl)
  .setName('License Key')
  .setDesc('Enter your license key to unlock premium features')
  .addText(text => text
    .setPlaceholder('Enter your license key')
    .setValue(this.plugin.settings.licenseKey)
    .onChange(async (value) => {
      this.plugin.settings.licenseKey = value;
      await this.plugin.saveSettings();
    }));
```

### 4.2: Add License Validation

```typescript
import { ObsidianMonetization } from 'obsidian-monetization-sdk';

export default class MyPlugin extends Plugin {
  settings: MyPluginSettings;
  monetization: ObsidianMonetization;
  isLicensed: boolean = false;

  async onload() {
    await this.loadSettings();
    
    // Initialize monetization
    this.monetization = new ObsidianMonetization('my-awesome-plugin');
    this.monetization.setLicenseKey(this.settings.licenseKey);
    
    // Check license on startup
    this.isLicensed = await this.monetization.checkLicense();
    
    // Register commands based on license status
    if (this.isLicensed) {
      this.addCommand({
        id: 'premium-feature',
        name: 'Premium Feature',
        callback: () => this.premiumFeature()
      });
    } else {
      this.addCommand({
        id: 'upgrade-prompt',
        name: 'Unlock Premium Features',
        callback: () => this.showUpgradePrompt()
      });
    }
  }
  
  async showUpgradePrompt() {
    const modal = new Modal(this.app);
    modal.titleEl.setText('Unlock Premium Features');
    
    modal.contentEl.createEl('p', { 
      text: 'Get access to premium features with a license.' 
    });
    
    const button = modal.contentEl.createEl('button', { 
      text: 'Purchase License',
      cls: 'mod-cta'
    });
    
    button.addEventListener('click', async () => {
      await this.monetization.openPurchasePage({
        planType: 'one_time'
      });
      modal.close();
    });
    
    modal.open();
  }
}
```

## Step 5: Test Integration

1. Build your plugin
2. Install in Obsidian
3. Click "Purchase License" button
4. Use Stripe test card: `4242 4242 4242 4242`
5. Complete purchase
6. Check email for license key
7. Enter license key in plugin settings
8. Verify premium features unlock

## Pricing Models

### One-Time Purchase
- User pays once, gets perpetual license
- Best for: Tools, templates, utilities
- Example: $10-$50

### Monthly Subscription
- Recurring payment, cancel anytime
- Best for: Services, APIs, ongoing value
- Example: $3-$10/month

### Free Trial
- Let users try before buying (subscriptions only)
- Example: 7-14 day trial

## Best Practices

1. **Clear Value Proposition**: Show users exactly what premium features they get
2. **Graceful Degradation**: Free version should still be useful
3. **Cache License Checks**: Check on startup, not every action
4. **Handle Offline**: Allow cached license for offline use
5. **Good Error Messages**: Tell users why license check failed

## Support

- Documentation: [docs.obbypay.com](https://docs.obbypay.com)
- Contact: support@obbypay.com
- Issues: [github.com/yourrepo/issues](https://github.com/yourrepo/issues)
```

**Action**: Create `docs/api-reference.md` with all API endpoints documented.

**Validation**:
- [ ] Documentation covers all integration steps
- [ ] Code examples are correct and tested
- [ ] Screenshots included for key steps
- [ ] Troubleshooting section included

**Success Criteria**: Developer can integrate following docs alone.

---

### Task 8.2: Prepare Marketing Materials
**Objective**: Create landing page and launch announcement.

**Action**: Update landing page `app/views/pages/home.html.erb` with compelling copy:
```erb
<!-- Hero Section -->
<div class="bg-gradient-to-br from-blue-50 to-indigo-100 py-20">
  <div class="max-w-6xl mx-auto px-4 text-center">
    <h1 class="text-5xl md:text-6xl font-bold text-gray-900 mb-6">
      Turn Your Obsidian Plugin<br/>Into a Business
    </h1>
    
    <p class="text-xl md:text-2xl text-gray-700 mb-8 max-w-3xl mx-auto">
      Accept payments, manage licenses, and earn revenue from your plugins.
      Integration takes 30 minutes. No monthly fees—only pay when you earn.
    </p>
    
    <div class="flex flex-col sm:flex-row gap-4 justify-center mb-12">
      <%= link_to "Start Earning →", new_user_registration_path, 
          class: "bg-blue-600 text-white px-8 py-4 rounded-lg text-lg font-semibold hover:bg-blue-700 transition" %>
      <%= link_to "View Documentation", "#docs", 
          class: "bg-white text-gray-800 px-8 py-4 rounded-lg text-lg font-semibold hover:bg-gray-50 transition border-2 border-gray-200" %>
    </div>
    
    <p class="text-sm text-gray-600">
      Join 50+ plugin developers already earning with their Obsidian plugins
    </p>
  </div>
</div>

<!-- Social Proof -->
<div class="py-16 bg-white">
  <div class="max-w-6xl mx-auto px-4">
    <p class="text-center text-gray-500 mb-12">Trusted by developers of popular plugins</p>
    <!-- Add logos or testimonials here -->
  </div>
</div>

<!-- Features Section -->
<div class="py-20 bg-gray-50">
  <div class="max-w-6xl mx-auto px-4">
    <h2 class="text-4xl font-bold text-center mb-16">
      Everything You Need to Monetize
    </h2>
    
    <div class="grid md:grid-cols-3 gap-12">
      <div class="text-center">
        <div class="w-16 h-16 bg-blue-100 rounded-full flex items-center justify-center mx-auto mb-6">
          <svg class="w-8 h-8 text-blue-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8c-1.657 0-3 .895-3 2s1.343 2 3 2 3 .895 3 2-1.343 2-3 2m0-8c1.11 0 2.08.402 2.599 1M12 8V7m0 1v8m0 0v1m0-1c-1.11 0-2.08-.402-2.599-1M21 12a9 9 0 11-18 0 9 9 0 0118 0z"/>
          </svg>
        </div>
        <h3 class="text-xl font-bold mb-3">Zero Upfront Cost</h3>
        <p class="text-gray-600">
          No monthly fees or setup costs. Only pay 5% when you make a sale.
          Your first $1,000 is commission-free.
        </p>
      </div>
      
      <div class="text-center">
        <div class="w-16 h-16 bg-green-100 rounded-full flex items-center justify-center mx-auto mb-6">
          <svg class="w-8 h-8 text-green-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 10V3L4 14h7v7l9-11h-7z"/>
          </svg>
        </div>
        <h3 class="text-xl font-bold mb-3">30-Minute Integration</h3>
        <p class="text-gray-600">
          Drop-in SDK with simple API. Copy 10 lines of code and start selling.
          No backend server required.
        </p>
      </div>
      
      <div class="text-center">
        <div class="w-16 h-16 bg-purple-100 rounded-full flex items-center justify-center mx-auto mb-6">
          <svg class="w-8 h-8 text-purple-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12l2 2 4-4m5.618-4.016A11.955 11.955 0 0112 2.944a11.955 11.955 0 01-8.618 3.04A12.02 12.02 0 003 9c0 5.591 3.824 10.29 9 11.622 5.176-1.332 9-6.03 9-11.622 0-1.042-.133-2.052-.382-3.016z"/>
          </svg>
        </div>
        <h3 class="text-xl font-bold mb-3">You Keep Control</h3>
        <p class="text-gray-600">
          Money goes directly to your Stripe account. You own the customer
          relationship and all data.
        </p>
      </div>
    </div>
  </div>
</div>

<!-- How It Works -->
<div class="py-20 bg-white">
  <div class="max-w-4xl mx-auto px-4">
    <h2 class="text-4xl font-bold text-center mb-16">How It Works</h2>
    
    <div class="space-y-12">
      <div class="flex gap-6">
        <div class="flex-shrink-0 w-12 h-12 bg-blue-600 text-white rounded-full flex items-center justify-center font-bold text-xl">
          1
        </div>
        <div>
          <h3 class="text-xl font-bold mb-2">Connect Your Stripe Account</h3>
          <p class="text-gray-600">
            Sign up and link your existing Stripe account (or create one).
            Takes 5 minutes.
          </p>
        </div>
      </div>
      
      <div class="flex gap-6">
        <div class="flex-shrink-0 w-12 h-12 bg-blue-600 text-white rounded-full flex items-center justify-center font-bold text-xl">
          2
        </div>
        <div>
          <h3 class="text-xl font-bold mb-2">Create Your Plugin & Set Pricing</h3>
          <p class="text-gray-600">
            Add your plugin, choose one-time or subscription pricing.
            We auto-generate everything in Stripe.
          </p>
        </div>
      </div>
      
      <div class="flex gap-6">
        <div class="flex-shrink-0 w-12 h-12 bg-blue-600 text-white rounded-full flex items-center justify-center font-bold text-xl">
          3
        </div>
        <div>
          <h3 class="text-xl font-bold mb-2">Add SDK to Your Plugin</h3>
          <p class="text-gray-600">
            Copy 10 lines of code into your plugin. Check license status and
            trigger purchases with simple API calls.
          </p>
        </div>
      </div>
      
      <div class="flex gap-6">
        <div class="flex-shrink-0 w-12 h-12 bg-blue-600 text-white rounded-full flex items-center justify-center font-bold text-xl">
          4
        </div>
        <div>
          <h3 class="text-xl font-bold mb-2">Start Earning</h3>
          <p class="text-gray-600">
            Users purchase directly through Stripe Checkout. License keys
            auto-generated and emailed. Revenue hits your account immediately.
          </p>
        </div>
      </div>
    </div>
  </div>
</div>

<!-- Pricing -->
<div class="py-20 bg-gray-50">
  <div class="max-w-3xl mx-auto px-4 text-center">
    <h2 class="text-4xl font-bold mb-6">Simple, Fair Pricing</h2>
    <p class="text-xl text-gray-600 mb-12">
      Only pay when you make money. No surprises.
    </p>
    
    <div class="bg-white rounded-xl shadow-lg p-12">
      <div class="text-5xl font-bold text-blue-600 mb-4">5%</div>
      <div class="text-2xl font-semibold mb-6">Per Transaction</div>
      <ul class="text-left space-y-4 mb-8 max-w-md mx-auto">
        <li class="flex items-start gap-3">
          <svg class="w-6 h-6 text-green-600 flex-shrink-0 mt-1" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7"/>
          </svg>
          <span>First $1,000 commission-free</span>
        </li>
        <li class="flex items-start gap-3">
          <svg class="w-6 h-6 text-green-600 flex-shrink-0 mt-1" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7"/>
          </svg>
          <span>No monthly fees</span>
        </li>
        <li class="flex items-start gap-3">
          <svg class="w-6 h-6 text-green-600 flex-shrink-0 mt-1" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7"/>
          </svg>
          <span>No setup costs</span>
        </li>
        <li class="flex items-start gap-3">
          <svg class="w-6 h-6 text-green-600 flex-shrink-0 mt-1" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7"/>
          </svg>
          <span>Direct deposits to your Stripe</span>
        </li>
      </ul>
      
      <p class="text-sm text-gray-500 mb-8">
        + Standard Stripe fees (2.9% + $0.30 per transaction)
      </p>
      
      <%= link_to "Get Started Free", new_user_registration_path, 
          class: "bg-blue-600 text-white px-8 py-4 rounded-lg text-lg font-semibold hover:bg-blue-700 inline-block" %>
    </div>
  </div>
</div>

<!-- CTA -->
<div class="py-20 bg-blue-600 text-white">
  <div class="max-w-4xl mx-auto px-4 text-center">
    <h2 class="text-4xl md:text-5xl font-bold mb-6">
      Ready to Monetize Your Plugin?
    </h2>
    <p class="text-xl mb-8 text-blue-100">
      Join developers already earning from their Obsidian plugins.
      Integration takes 30 minutes.
    </p>
    <%= link_to "Create Free Account →", new_user_registration_path, 
        class: "bg-white text-blue-600 px-8 py-4 rounded-lg text-lg font-semibold hover:bg-gray-100 inline-block" %>
  </div>
</div>
```

**Validation**:
- [ ] Landing page is compelling and clear
- [ ] All links work correctly
- [ ] Responsive design on mobile
- [ ] Fast page load times
- [ ] Clear call-to-actions

**Success Criteria**: Landing page ready for public launch.

---

## Launch Checklist

### Pre-Launch
- [ ] All tests passing
- [ ] Documentation complete and accurate
- [ ] SDK published to npm
- [ ] Production environment configured
- [ ] Stripe webhooks tested
- [ ] Email delivery working
- [ ] SSL certificate active
- [ ] Custom domain configured (if applicable)
- [ ] Error tracking set up (Sentry, Honeybadger, etc.)
- [ ] Analytics installed (Google Analytics, Plausible, etc.)

### Launch Day
- [ ] Post to Obsidian Forum "Share & Showcase"
- [ ] Post to r/ObsidianMD with [Showcase] tag
- [ ] Announce in Discord #plugin-dev
- [ ] Tweet with #ObsidianMD hashtag
- [ ] Submit to Obsidian Roundup newsletter
- [ ] Reach out to 5-10 popular plugin developers personally
- [ ] Monitor for issues and respond quickly

### Post-Launch
- [ ] Collect feedback from early adopters
- [ ] Fix any bugs reported
- [ ] Create video tutorial
- [ ] Write blog post about development journey
- [ ] Reach out to Obsidian influencers
- [ ] Iterate based on user feedback

---

## Troubleshooting Guide

### Common Issues

**Stripe Connection Fails**:
- Check CLIENT_ID is correct
- Verify callback URL matches exactly
- Ensure redirect_uri in OAuth call is correct

**Webhook Not Receiving Events**:
- Verify webhook secret matches
- Check webhook signature verification code
- Test with Stripe CLI: `stripe listen --forward-to localhost:3000/webhooks/stripe`
- Ensure endpoint is publicly accessible (use ngrok for local testing)

**License Key Not Generated**:
- Check webhook handler is processing `checkout.session.completed`
- Verify License model validations
- Check logs for errors during license creation
- Ensure email delivery is configured

**API Returns 500 Error**:
- Check logs: `heroku logs --tail` or `rails log`
- Verify all environment variables are set
- Check database migrations ran successfully
- Verify Stripe API keys are valid

**SDK Can't Validate License**:
- Check API endpoint is accessible
- Verify CORS settings allow requests
- Check rate limiting isn't blocking requests
- Ensure license key is being sent correctly

---

## Success Metrics to Track

- Number of registered developers
- Number of plugins created
- Total transaction volume (GMV)
- Active licenses across all plugins
- Average revenue per developer
- Conversion rate (signups → connected Stripe)
- Time to first sale for new developers
- Support ticket volume
- API uptime percentage

---

## Next Steps for Autonomous Agent

After completing all tasks above, the platform is functionally complete. Additional enhancements can include:

1. **Analytics Dashboard**: Show revenue graphs, conversion funnels
2. **Email Notifications**: Alert developers of new purchases
3. **Webhooks for Developers**: Let developers receive events
4. **Team Accounts**: Multiple developers per plugin
5. **Advanced Pricing**: Tiers, discounts, bundles
6. **Refund Management**: UI for issuing refunds
7. **Customer Portal**: Let end-users manage subscriptions
8. **API Rate Limiting**: More sophisticated limiting
9. **Developer API Keys**: For advanced integrations
10. **A/B Testing**: Test different pricing strategies

The platform is now ready for developers to start integrating and monetizing their Obsidian plugins!
