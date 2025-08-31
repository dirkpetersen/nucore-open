# NUcore REST API Implementation Specification

## Current State Analysis

### Existing API Endpoints

NUcore currently has very limited API functionality:

#### Main API (`/api`)
- **GET /api/order_details/:id** - Retrieve order detail information
- **GET /api/order_details?id=:id** - Alternative query parameter format (for Qualtrics integration)
- Authentication: HTTP Basic Auth via `Rails.application.secrets.api[:basic_auth_name/password]`
- Response Format: JSON only
- Returns: Order number, account info (ID, owner), and ordered_for user details

#### Secure Rooms API (`/secure_rooms_api`) - Engine-specific
- **POST /secure_rooms_api/scans** - Card reader access control
- Authentication: HTTP Basic Auth via `Rails.application.secrets.secure_rooms_api`
- Purpose: Hardware integration for card reader access control

#### Health Check API
- **GET /health_check** - System health monitoring
- Provided by `health_check` gem

### Current API Limitations

1. **Extremely Limited Scope**: Only one main endpoint for order details
2. **No CRUD Operations**: Read-only access, no create/update/delete capabilities  
3. **No Resource Coverage**: Missing endpoints for core resources (facilities, products, reservations, accounts, users)
4. **No Versioning**: No API version management
5. **Basic Authentication Only**: No OAuth, API keys, or modern auth mechanisms
6. **No Rate Limiting**: No protection against abuse
7. **No Documentation**: No OpenAPI/Swagger documentation
8. **No Pagination**: Limited data retrieval capabilities
9. **No Filtering/Searching**: No query capabilities beyond basic ID lookup
10. **No Relationships**: Cannot traverse resource relationships via API

## Comprehensive REST API Implementation Plan

### 1. API Foundation & Infrastructure

#### Authentication & Authorization
```ruby
# Implement multiple authentication strategies
# app/controllers/api/v1/base_controller.rb
class Api::V1::BaseController < ApplicationController
  include ApiAuthentication
  include ApiAuthorization
  
  skip_before_action :authenticate_user!
  before_action :authenticate_api_user!
  
  private
  
  def authenticate_api_user!
    # Support multiple auth methods:
    # 1. API Token (recommended)
    # 2. OAuth 2.0 Bearer tokens  
    # 3. HTTP Basic Auth (current, legacy support)
    authenticate_with_token || authenticate_with_oauth || authenticate_with_basic_auth
  end
end
```

#### API Versioning
```ruby
# config/routes.rb
namespace :api do
  namespace :v1 do
    # All v1 routes here
  end
  
  # Future versions
  # namespace :v2 do
  #   # v2 routes
  # end
end
```

#### Serialization Framework
```ruby
# Use active_model_serializers or jsonapi-serializer
# app/serializers/api/v1/facility_serializer.rb
class Api::V1::FacilitySerializer < ActiveModel::Serializer
  attributes :id, :name, :abbreviation, :description, :is_active
  
  has_many :products
  has_many :orders
  
  link(:self) { api_v1_facility_url(object) }
end
```

### 2. Core Resource APIs

#### Facilities API
```
GET    /api/v1/facilities                    # List all facilities
POST   /api/v1/facilities                    # Create facility (admin only)
GET    /api/v1/facilities/:id                # Get facility details
PUT    /api/v1/facilities/:id                # Update facility (staff+)
DELETE /api/v1/facilities/:id                # Deactivate facility (admin only)
GET    /api/v1/facilities/:id/products       # List facility products
GET    /api/v1/facilities/:id/orders         # List facility orders
GET    /api/v1/facilities/:id/reservations   # List facility reservations
GET    /api/v1/facilities/:id/accounts       # List facility accounts
```

#### Products API (Instruments, Items, Services, Bundles)
```
GET    /api/v1/products                      # List all products (with filtering)
GET    /api/v1/products/:id                  # Get product details
PUT    /api/v1/products/:id                  # Update product (staff+)

# Product-specific endpoints
GET    /api/v1/instruments                   # List instruments only
GET    /api/v1/instruments/:id               # Get instrument details  
GET    /api/v1/instruments/:id/reservations  # Get instrument reservations
GET    /api/v1/instruments/:id/schedule      # Get instrument schedule/availability

GET    /api/v1/items                         # List items only
GET    /api/v1/services                      # List services only
GET    /api/v1/bundles                       # List bundles only
```

#### Orders & Order Details API
```
GET    /api/v1/orders                        # List orders (filtered by user/facility)
POST   /api/v1/orders                        # Create new order
GET    /api/v1/orders/:id                    # Get order details
PUT    /api/v1/orders/:id                    # Update order (limited fields)
DELETE /api/v1/orders/:id                    # Cancel order (if allowed)

GET    /api/v1/orders/:id/order_details      # List order line items
POST   /api/v1/orders/:id/order_details      # Add item to order
GET    /api/v1/order_details/:id             # Get order detail (existing endpoint)
PUT    /api/v1/order_details/:id             # Update order detail
DELETE /api/v1/order_details/:id             # Remove from order
```

#### Reservations API
```
GET    /api/v1/reservations                  # List reservations (filtered)
POST   /api/v1/reservations                  # Create reservation
GET    /api/v1/reservations/:id              # Get reservation details
PUT    /api/v1/reservations/:id              # Update reservation (if allowed)
DELETE /api/v1/reservations/:id              # Cancel reservation

GET    /api/v1/reservations/availability     # Check availability
POST   /api/v1/reservations/batch            # Batch reservation operations
```

#### Accounts API
```
GET    /api/v1/accounts                      # List user's accounts
POST   /api/v1/accounts                      # Create account (if allowed)
GET    /api/v1/accounts/:id                  # Get account details
PUT    /api/v1/accounts/:id                  # Update account details

GET    /api/v1/accounts/:id/transactions     # Account transaction history
GET    /api/v1/accounts/:id/statements       # Account statements
GET    /api/v1/accounts/:id/users            # Account users/members
```

#### Users API
```
GET    /api/v1/users/profile                 # Current user profile
PUT    /api/v1/users/profile                 # Update current user profile
GET    /api/v1/users/:id                     # Get user details (if authorized)

# Admin/Staff endpoints
GET    /api/v1/users                         # List users (admin/staff only)
POST   /api/v1/users                         # Create user (admin only)
PUT    /api/v1/users/:id                     # Update user (admin/staff)
```

### 3. Advanced API Features

#### Search & Filtering
```ruby
# app/controllers/api/v1/products_controller.rb
def index
  @products = Product.accessible_by(current_ability)
  
  # Apply filters
  @products = @products.where(facility: params[:facility_id]) if params[:facility_id]
  @products = @products.where(type: params[:type]) if params[:type]
  @products = @products.where("name ILIKE ?", "%#{params[:search]}%") if params[:search]
  @products = @products.active if params[:active] == 'true'
  
  # Apply sorting
  @products = @products.order(params[:sort] || 'name')
  
  # Paginate
  @products = @products.page(params[:page]).per(params[:per_page] || 25)
  
  render json: @products, meta: pagination_meta(@products)
end
```

#### Pagination Response Format
```json
{
  "data": [...],
  "meta": {
    "current_page": 1,
    "per_page": 25,
    "total_pages": 4,
    "total_count": 95
  },
  "links": {
    "self": "/api/v1/products?page=1",
    "first": "/api/v1/products?page=1", 
    "next": "/api/v1/products?page=2",
    "last": "/api/v1/products?page=4"
  }
}
```

#### Error Handling
```ruby
# app/controllers/concerns/api_error_handling.rb
module ApiErrorHandling
  extend ActiveSupport::Concern
  
  included do
    rescue_from ActiveRecord::RecordNotFound, with: :record_not_found
    rescue_from ActiveRecord::RecordInvalid, with: :record_invalid
    rescue_from CanCan::AccessDenied, with: :access_denied
  end
  
  private
  
  def record_not_found(exception)
    render json: {
      error: {
        type: "record_not_found",
        message: exception.message,
        details: {}
      }
    }, status: 404
  end
  
  def record_invalid(exception)
    render json: {
      error: {
        type: "validation_error", 
        message: "Validation failed",
        details: exception.record.errors.as_json
      }
    }, status: 422
  end
end
```

### 4. Implementation Steps

#### Phase 1: Foundation (Week 1-2)
1. **Create API base controller** with authentication, authorization, error handling
2. **Set up API routing structure** with versioning
3. **Implement API token authentication system**
4. **Add serialization framework** (active_model_serializers recommended)
5. **Create API documentation structure** (OpenAPI/Swagger)

#### Phase 2: Read-Only Core Resources (Week 3-4)  
1. **Facilities API** - GET endpoints
2. **Products API** - GET endpoints with filtering
3. **Orders/OrderDetails API** - Enhance existing endpoint, add listing
4. **Users API** - Profile and lookup endpoints
5. **Add pagination and search capabilities**

#### Phase 3: Extended Read Operations (Week 5-6)
1. **Reservations API** - GET endpoints
2. **Accounts API** - GET endpoints  
3. **Availability checking** for reservations
4. **Relationship traversal** (facility products, order details, etc.)
5. **Advanced filtering and sorting**

#### Phase 4: Write Operations (Week 7-8)
1. **Order creation and management** 
2. **Reservation creation and updates**
3. **Account management** (where permitted)
4. **User profile updates**
5. **Bulk operations support**

#### Phase 5: Advanced Features (Week 9-10)
1. **Rate limiting** implementation
2. **Webhook support** for real-time notifications  
3. **File upload/download** endpoints
4. **Batch operations** for efficiency
5. **API analytics and monitoring**

### 5. Required Code Components

#### Authentication System
```ruby
# app/models/api_token.rb
class ApiToken < ApplicationRecord
  belongs_to :user
  
  validates :name, presence: true
  validates :token, presence: true, uniqueness: true
  
  before_create :generate_token
  
  scope :active, -> { where(revoked_at: nil) }
  
  def active?
    revoked_at.nil? && (expires_at.nil? || expires_at > Time.current)
  end
  
  private
  
  def generate_token
    self.token = SecureRandom.hex(32) 
  end
end

# Migration
class CreateApiTokens < ActiveRecord::Migration[7.0]
  def change
    create_table :api_tokens do |t|
      t.references :user, null: false, foreign_key: true
      t.string :name, null: false
      t.string :token, null: false
      t.datetime :expires_at
      t.datetime :revoked_at
      t.datetime :last_used_at
      t.timestamps
    end
    
    add_index :api_tokens, :token, unique: true
    add_index :api_tokens, [:user_id, :revoked_at]
  end
end
```

#### Rate Limiting
```ruby
# app/controllers/concerns/api_rate_limiting.rb  
module ApiRateLimiting
  extend ActiveSupport::Concern
  
  included do
    before_action :check_rate_limit
  end
  
  private
  
  def check_rate_limit
    # Use Redis or database-based rate limiting
    # Example: 1000 requests per hour per token
    rate_limit_key = "api_rate_limit:#{current_api_token&.id || request.ip}"
    
    if rate_limit_exceeded?(rate_limit_key)
      render json: { error: "Rate limit exceeded" }, status: 429
      return
    end
    
    increment_rate_limit(rate_limit_key)
  end
end
```

#### API Documentation
```ruby
# Use rswag gem for OpenAPI documentation
# spec/requests/api/v1/facilities_spec.rb  
describe 'Facilities API' do
  path '/api/v1/facilities' do
    get 'List facilities' do
      tags 'Facilities'
      produces 'application/json'
      
      parameter name: :page, in: :query, type: :integer, description: 'Page number'
      parameter name: :per_page, in: :query, type: :integer, description: 'Items per page'
      parameter name: :active, in: :query, type: :boolean, description: 'Filter by active status'
      
      response 200, 'facilities found' do
        schema type: :object,
               properties: {
                 data: {
                   type: :array,
                   items: { '$ref' => '#/components/schemas/facility' }
                 },
                 meta: { '$ref' => '#/components/schemas/pagination_meta' }
               }
      end
    end
  end
end
```

### 6. Configuration Requirements

#### Secrets Configuration
```yaml
# config/secrets.yml
api:
  basic_auth_name: <%= ENV.fetch("NUCORE_API_USERNAME", "nucore") %>
  basic_auth_password: <%= ENV.fetch("NUCORE_API_PASSWORD", "changeme") %>
  
  # New token-based auth
  token_expiry_days: <%= ENV.fetch("API_TOKEN_EXPIRY_DAYS", "90") %>
  rate_limit_requests_per_hour: <%= ENV.fetch("API_RATE_LIMIT", "1000") %>
```

#### Routes Configuration  
```ruby
# config/routes.rb
Rails.application.routes.draw do
  # API routes
  namespace :api do
    namespace :v1 do
      resources :facilities do
        resources :products, only: [:index]
        resources :orders, only: [:index] 
        resources :reservations, only: [:index]
      end
      
      resources :products do
        member do
          get :availability if product_type == 'Instrument'
        end
      end
      
      resources :orders do
        resources :order_details
      end
      
      resources :order_details, only: [:show, :update, :destroy]
      resources :reservations
      resources :accounts
      
      # User endpoints
      get 'users/profile', to: 'users#profile'
      put 'users/profile', to: 'users#update_profile'
      resources :users, only: [:show, :index] # Admin only
      
      # Utility endpoints
      get 'health', to: 'health#check'
      post 'auth/token', to: 'authentication#create_token'
    end
  end
end
```

### 7. Testing Strategy

#### Controller Tests
```ruby
# spec/controllers/api/v1/facilities_controller_spec.rb
describe Api::V1::FacilitiesController do
  let(:api_token) { create(:api_token) }
  
  before do
    request.headers['Authorization'] = "Bearer #{api_token.token}"
  end
  
  describe 'GET #index' do
    it 'returns facilities accessible to user' do
      create_list(:facility, 3)
      
      get :index
      
      expect(response).to have_http_status(:ok)
      expect(json_response['data']).to have(3).items
    end
    
    it 'filters by active status' do
      active_facility = create(:facility, is_active: true)
      inactive_facility = create(:facility, is_active: false)
      
      get :index, params: { active: true }
      
      expect(json_response['data']).to have(1).item
    end
  end
end
```

### 8. Security Considerations

1. **Authentication**: Multi-method support (tokens, OAuth, basic auth)
2. **Authorization**: CanCanCan integration for resource-level permissions
3. **Rate Limiting**: Prevent API abuse
4. **Input Validation**: Strong parameter filtering and validation
5. **HTTPS Only**: Force SSL in production
6. **Token Management**: Expiration, revocation, rotation capabilities
7. **Audit Logging**: Track API usage and changes
8. **CORS Configuration**: Proper cross-origin request handling

This comprehensive API would transform NUcore from a web-only application into a platform that can integrate with external systems, mobile applications, and automation tools while maintaining the security and permission models already established in the system.