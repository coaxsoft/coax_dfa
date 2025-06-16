# Ruby examples - instructions for AI

This document serves as a guide for structuring Ruby classes in accordance with the ABDI (App Business Data Infrastructure) architectural pattern, adapted to specific naming conventions requested by the user.

## 1. ABDI Core Concepts
   The architecture revolves around 4 main nodes, each with distinct responsibilities:

App: Handles representation (views, serializers) and request input/output (controllers, channels). No business logic.

Business: Contains all use case validation, business rules execution, and business logic.

Data: Responsible solely for database read/write operations and persistence-level validation.
No business logic or callbacks beyond system requirements.
Infrastructure: Holds base classes (e.g., BaseOperation), external service clients, and common utilities.

More details at: https://gist.githubusercontent.com/OrestF/e44a39dd73eddbca76bd3a09bdd706d8/raw/263e5d80ca209ea1a2ef8e78ee87a6f1421a8bad/abdi.md

## 2. Code examples


### App Layer Example

#### Controller

```ruby
# app/api/v1/users_controller.rb

class Api::V1::UsersController < Api::V1::BaseController
  skip_before_action :verify_record, :verify_class,
                     only: %i[profile update_profile]

  def index
    collection_response(
      Search::Users.call(scope: policy_scope(User.all), search_params: params),
      root: :users
    )
  end

  def show
    record_response(record, root: :user, view: :full)
  end

  def profile
    authorize current_user

    record_response(current_user, root: :user, view: :full)
  end

  def update_profile
    authorize current_user

    operation_response(
      Users::Operations::Update.call(record: current_user, record_params: record_params),
      root: :user,
      view: :full
    )
  end

  def lock
    operation_response(Users::Operations::Lock.call(record: record), view: :full)
  end

  def unlock
    operation_response(Users::Operations::Unlock.call(record: record), view: :full)
  end
end
```

#### Serializer

```ruby
# app/serializers/user_serializer.rb

class UserSerializer < ApplicationSerializer
  DEFAULT_FIELDS = %i[
    email
    role
    full_name
    phone_number
    created_at
    updated_at
    invitation_sent_at
    invitation_accepted_at
    locked_at
    operator_organization_id
  ].freeze

  fields(*DEFAULT_FIELDS)
  association :avatar, blueprint: BlobSerializer

  view :full do
    association :avatar, blueprint: BlobSerializer
  end

  view :sign_in do
    field :jwt_token do |_, options|
      options[:jwt_token]
    end
  end
  
  view :with_invited_by do
    field :invited_by, blueprint: ->(record) { record.blueprint }
  end
end
```

#### Search Service

```ruby
# app/services/search/users.rb

class Search::Users < BaseSearch
  FORM_OPTIONS = User.options_for_search.merge(
    by_free_text: '*',
    by_status: User::FILTERABLE_STATUSES,
    by_tickets_operator_organizations: :operator_organization_id
  ).freeze
  PERMITTED_ATTRIBUTES = FORM_OPTIONS.keys
end
```

#### Policy

```ruby
# app/policies/user_policy.rb

class UserPolicy < ApplicationPolicy
  class Scope < ApplicationPolicy::Scope
    def resolve
      case user.role
      when 'admin', 'operator'
        scope
      else
        User.where(id: user)
      end
    end
  end

  def profile?
    user.can?(:user, :read) { my_profile? }
  end

  def update_profile?
    user.can?(:user, :update_profile) { my_profile? }
  end
  alias update_location? update_profile?

  def lock?
    user.can?(:user, :lock) { own_org? }
  end

  def unlock?
    user.can?(:user, :unlock) { own_org? }
  end

  private

  def my_profile?
    return false if user.nil?

    user == record
  end

  def own_org?
    record&.organization == organization
  end
end
```

### Business Layer Example

#### Operation

```ruby
# business/users/operations/invite.rb

class Users::Operations::Invite < BaseOperation
  attr_reader :initiated_by

  def call
    build_record
    patch_record
    skip_confirmation_notification

    mutate_record_call! do
      send_invitation
      success(record: record)
    end
  end

  private

  def form_class
    Users::Forms::Invite
  end

  def assign_attributes
    record.assign_attributes(record_params.merge!(password: "ABC$#{SecureRandom.hex}"))
  end

  def build_record
    @record = User.new(operator_organization_id: record_params[:operator_organization_id])
  end

  def send_invitation
    record.invite!(Current.user)
  end

  # customer could have multiple organizations. Relation is defined by OrganizationCustomer
  def patch_record
    record.operator_organization_id = nil unless belongs_to_operator?
  end

  def belongs_to_operator?
    record_params[:role].to_s.in?(%w[operator driver])
  end

  def skip_confirmation_notification
    record.skip_confirmation_notification!
  end
end
```

#### Form

```ruby
# business/users/forms/invite.rb

class Users::Forms::Invite < BaseForm
  PERMITTED_ATTRIBUTES = %i[
    full_name
    email
    role
    operator_organization_id
    accept_invitation_url
  ].freeze
  REQUIRED_ATTRIBUTES = %i[email role accept_invitation_url].freeze

  validate :email_uniq_validation

  private

  def email_uniq_validation
    return unless User.exists?(email: email)

    errors.add(:email, 'has already been taken')
  end
end
```

### Data Layer Example
```ruby
# data/user.rb

class User < ApplicationRecord
  FILTERABLE_STATUSES = %w[active deactivated with_pending_invitation].freeze

  store_accessor :payment_provider_details, %i[stripe_customer_id]

  include Auditable
  include Devise::JWT::RevocationStrategies::JTIMatcher
  include Passpartu
  include Users::TemporaryData
  include Users::Searchable
  include Users::Authenticatable
  include Users::Associations
  include ExpirableAttributes
  expirable_attributes longitude: -> { 15.minutes }, latitude: -> { 15.minutes }

  enum :role, {
    rider: 0,
    driver: 10,
    operator: 20,
    admin: 30
  }, validate: true

  validates :avatar, content_type: ALLOWED_IMAGE_FORMATS, size: { between: ALLOWED_FILE_SIZE_RANGE }

  def fcm_notifications?
    notification_tokens.fcm.any?
  end
  alias push_notifications? fcm_notifications?

  def financial_transactions_sum
    financial_transactions.sum(:amount)
  end

  def financial_transactions_count
    financial_transactions.count
  end

  # MOTOR ACTIONS
  def motor_resend_sign_up_confirmation_email
    return unless rider?

    send_confirmation_instructions
  end

  def motor_confirm_account
    confirm
  end
end
```

### Infrastructure Layer Example

```ruby
# infrastructure/base_operation.rb

class BaseOperation < Readymade::Operation
  include BaseOperations::Save

  def form_class
    self.class.name.gsub('Operations', 'Forms').constantize
  end
end
```

```ruby
# infrastructure/base_operations/save.rb

module BaseOperations
  module Save
    def save_record!
      record.save!
    end

    def mutate_record_call!
      return validation_fail unless form_valid?

      assign_attributes
      return validation_fail unless record_valid?

      save_record!

      return yield if block_given?

      success(record:)
    end
  end
end
```


```ruby
# infrastructure/my_branch_io/client.rb

class MyBranchIo::Client
  def initialize(**args)
    @key = args[:key] || RCreds.fetch(:branch_io, :key)
    @base_url = URI('https://api2.branch.io')
  end

  def create_deep_link(params)
    post('url', params)
  end

  def deep_link(url)
    get('url', { url: url })
  end

  # https://vt9g3.app.link/test

  private

  def http
    Net::HTTP.new(@base_url.host, @base_url.port).tap do |http|
      http.use_ssl = true
    end
  end

  def get(uri_path, params)
    url = build_url(uri_path)
    url.query = URI.encode_www_form(default_params.merge(params))
    parse_response(
      http.request(
        Net::HTTP::Get.new(url).tap do |request|
          request['Accept'] = 'application/json'
        end
      )
    )
  end

  def post(uri_path, params)
    url = build_url(uri_path)
    parse_response(
      http.request(
        Net::HTTP::Post.new(url).tap do |request|
          request['Content-Type'] = 'application/json'
          request['Accept'] = 'application/json'
          request.body = default_params.merge(params).to_json
        end
      )
    )
  end

  def build_url(path, version: 'v1')
    URI.join(@base_url, "/#{version}/", path)
  end

  def default_params
    { branch_key: @key }
  end

  def parse_response(response)
    JSON.parse(response.body)
  end
end
```
