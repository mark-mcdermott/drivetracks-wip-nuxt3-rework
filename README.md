# Minimal Rails API & Nuxt 3 App Tutorial

## Overview

In this tutorial, you'll build a lean Rails API (backend) and a Nuxt 3 app (frontend), both hosted on fly.io. There will be RSpec, Vitest and Playwright tests, which will run locally and on CircleCI.

## Part I: Barebones App With CircleCI Tests On Fly.io

### 1. Initial Setup
- `cd ~`
- `mkdir app`
- `cd app`
- `npx nuxi@latest init frontend`
  - package manager: `npm`
  - init git repo: `no`
  - modules: `eslint`, `fonts`, `icon`, `image`, `test-utils`
- `rails new backend --api --database=postgresql`
- `rm -rf backend/.git`
- `touch .secrets`
- make `~/app/.secrets` look like this:
```
fly.io url details:
  frontend url: 
  backend url: 

fly.io postgres cluster details
  name: 
  Username: 
  Password: 
  Hostname: 
  Flycast: 
  Proxy port: 
  Postgres port: 
  Connection string: 

AWS details:
  aws acct id: 
  aws region:  
  s3 user policy: 
  s3 user: 
  s3 user access key: 
  s3 user secret access key: 
  s3 bucket dev: 
  s3 bucket prod: 
```
  
- **Fly.io Deployment:**  
- `cd ~/app/backend`
  - we have to make a small addition to the default `backend/Dockerfile` that rails made us. At the end of the `apt-get install` line on line 22, add one more package to install, `libjemalloc2`
  - `fly launch --name app001-backend`
  - hit enter (for "no") when it asks a question about wanting to tweak the settings
  - watch the output and look for the `Postgres cluster` details, which end with the line, `Save your credentials in a secure place -- you won't be able to see them again!` When you see it, copy and paste it to the corresponding section in your `~/app/.secrets` file.
  - at the end of all the output it will say, `Visit your newly deployed app at https://<your backend app name>.fly.dev/` - copy/paste the backend app name url it gives you to the `backend url:` part of your `.secrets` file
- `cd ~/app/frontend`
  - like in the backend `fly launch` line above, your fly.io frontend app name has to be unique in their system, so you may have to run this a few times with different names after the `--name ` part until you find a unique one that works
  - `fly launch --name app001-frontend`
    - hit enter (for "no") when it asks a question about wanting to tweak the settings
    - copy the frontend app url it gives you at the end of all the output and paste it into your `.secrets` file at the `frontend url:` line
- in a browser, go to your fly.io frontend app url. You should see the default Nuxt placeholder homepage.

### 2. Building the MVP Hello World Version

#### Backend Health Endpoint
- **Controller:** Create `backend/app/controllers/api/v1/health_controller.rb`:
  ```ruby
  class Api::V1::HealthController < ApplicationController
    def show
      render json: { status: 'OK' }, status: :ok
    end
  end
  ```
- **Routes:** Update `backend/config/routes.rb`:
  ```ruby
  Rails.application.routes.draw do
    namespace :api do
      namespace :v1 do
        get 'up', to: 'health#show'
      end
    end
  end
  ```

#### Nuxt Default Page
- **Home Page:** Create `frontend/components/HomeContent.vue`
  ```vue
  <script setup>
  const apiBase = useRuntimeConfig().public.apiBase;
  const healthStatus = await $fetch(`${apiBase}/up`);
  </script>

  <template>
    <div>
      <h1>Welcome to Our Minimal App!</h1>
      <p>The backend says: {{ JSON.stringify(healthStatus) }}</p>
    </div>
  </template>
  ```
- edit `frontend/app.vue`:
  ```vue
  <template>
    <div>
      <HomeContent />
    </div>
  </template>
  ```
- update `frontend/nuxt.config.ts`:
  ```
  const development = process.env.NODE_ENV !== 'production';

  export default defineNuxtConfig({
    devServer: { port: 3001 },
    modules: [
      '@nuxt/eslint',
      '@nuxt/fonts',
      '@nuxt/icon',
      '@nuxt/image',
      '@nuxt/test-utils'
    ],
    runtimeConfig: {
      public: {
        apiBase: development ? 'http://localhost:3000/api/v1' : 'https://app001-backend.fly.dev/api/v1'
      }
    }
  });
  ```
- **Rails CORS:** must be enabled or Playwright will get a 500 error from rails
  - `cd ~/app/backend`
  - `bundle add rack-cors`
  - `bundle install`
  -  Edit `backend/config/initializers/cors.rb`:
  ```ruby
  Rails.application.config.middleware.insert_before 0, Rack::Cors do
    allow do
      origins '*'
      resource '*', headers: :any, methods: [:get, :post, :options]
    end
  end
  ```
- `cd ~/app/backend && rails server`
- `cd ~/app/frontend && npm run dev`

- before redeploying, we want to change fly.io's health check endpoint, to our health check endpoint or we'll get a 503 when we hit the frontend. In `backend/fly.toml`, (which fly.io created for us when we did `fly launch`) change the `path = '/up'` line in the `[[http_service.checks]]` section to:
  ```
  path = '/api/v1/up'
  ```
- `cd ~/app/backend && fly deploy`
- `cd ~/app/frontend && fly deploy`

### 3. Setting Up Testing

#### RSpec Test (Backend)
- `cd ~/app/backend`
- `bundle add rspec-rails shoulda-matchers --group "development, test"`
- `bundle install`
- `rails generate rspec:install`
- **Request Spec:** Create `backend/spec/requests/api/v1/health_controller_spec.rb`:
  ```ruby
  require 'rails_helper'

  RSpec.describe "Health Endpoint", type: :request do
    it "returns OK" do
      get "/api/v1/up", headers: { "ACCEPT" => "application/json" }
      expect(response).to have_http_status(:success)
      expect(JSON.parse(response.body)['status']).to eq('OK')
    end
  end
  ```
  `rspec`

#### Vitest Component Test (Frontend)
- `cd ~/app/frontend`
- `npm i --save-dev @nuxt/test-utils vitest @vue/test-utils happy-dom playwright-core`
- create `frontend/vitest.config.ts`
  ```
  import { defineVitestConfig } from '@nuxt/test-utils/config'
  export default defineVitestConfig({ })
  ```
- **Component Test:** Create `tests/components/HomeContent.nuxt.spec.ts`:
  ```ts
  import { mountSuspended } from '@nuxt/test-utils/runtime'
  import { HomeContent } from '#components'
  import { expect, it, vi } from 'vitest';

  const mockResponse = {
    healthStatus: 'OK',
  };

  global.fetch = vi.fn(() =>
    Promise.resolve({
      json: () => Promise.resolve(mockResponse),
    }),
  );

  it('can mount some component', async () => {
    const component = await mountSuspended(HomeContent)
    expect(component.text()).toMatchInlineSnapshot(
      '"Welcome to Our Minimal App!The backend says:"'
    )
  })
  ```
  `vitest tests/components`

#### Playwright End-to-End Test
- `cd ~/app/frontend`
- `npm install @playwright/test pixelmatch playwright-expect`
- `npx playwright install`
- **E2E Test:** Create `tests/e2e/home.spec.ts`:
  ```ts
  import { test, expect } from '@playwright/test'

  test('homepage displays backend OK status', async ({ page }) => {
    await page.goto('http://localhost:3001')
    await expect(page.locator('p')).toContainText('"status":"OK"')
  })
  ```
- `cd ~/app/backend && rails server`
- `cd ~/app/frontend && npm run dev`
- `npx playwright test tests/e2e`

### 4. CircleCI Integration
- `cd ~/app`
- create `.circleci/config.yml`:
  ```
  version: 2.1

  executors:
    docker-executor:
      docker:
        - image: cimg/base:stable
      working_directory: ~/app

  jobs:
    test:
      executor: docker-executor
      steps:
        - checkout

        - setup_remote_docker:
            docker_layer_caching: true

        - run:
            name: Install Bundler 2.6.5
            command: |
              docker-compose -f docker-compose.ci.yml build --build-arg BUNDLER_VERSION=2.6.5

        - run:
            name: Build and Start Services
            command: |
              docker-compose -f docker-compose.ci.yml up -d --build

        - run:
            name: Wait for Backend to Be Ready
            when: always
            command: |
              for i in {1..30}; do
                echo "Trying to hit backend from inside container..."
                if docker-compose -f docker-compose.ci.yml exec -T backend curl -s http://localhost:3000/api/v1/up | grep 'OK'; then
                  echo "✅ Backend is ready!"
                  exit 0
                fi
                echo "Waiting for backend..."
                sleep 5
              done
              echo "❌ Backend failed to start in time"
              docker-compose -f docker-compose.ci.yml logs backend || echo "No logs found"
              touch backend_failed

        - run:
            name: Inspect Backend Container
            when: always
            command: |
              docker ps -a
              echo "------"
              docker-compose -f docker-compose.ci.yml logs backend || echo "No logs found"
              echo "------"
              docker-compose -f docker-compose.ci.yml exec backend ls -lah /app/log || echo "Log dir not found"

        - run:
            name: Debug Backend Logs
            when: always
            command: docker-compose -f docker-compose.ci.yml logs backend || echo "No logs found"

        - run:
            name: Tail Rails Test Log
            when: always
            command: docker-compose -f docker-compose.ci.yml exec backend sh -c "cat log/test.log || echo 'log/test.log not found'"

        - run:
            name: Run Backend Tests
            command: docker-compose -f docker-compose.ci.yml exec backend bundle exec rspec

        - run:
            name: Run Frontend Tests (Vitest + Playwright)
            command: docker-compose -f docker-compose.ci.yml exec frontend sh -c "npm run vitest -- --run tests/components && npx playwright test tests/e2e"

        - run:
            name: Show Rails Logs (if needed)
            when: always
            command: docker-compose -f docker-compose.ci.yml logs backend || echo "No logs found"

        - run:
            name: Fail Job If Backend Died
            command: |
              if [ -f backend_failed ]; then
                echo "❌ Backend failed flag detected. Failing job."
                exit 1
              fi

  workflows:
    build_and_test:
      jobs:
        - test
  ```
- create `docker-compose.ci.yml`:
  ```
  version: '3.8'

  services:
    postgres:
      image: postgres:15
      environment:
        POSTGRES_USER: postgres
        POSTGRES_PASSWORD: password
        POSTGRES_DB: backend_test
      ports:
        - "5432:5432"

    backend:
      build:
        context: .
        dockerfile: backend/Dockerfile.ci
      environment:
        RAILS_ENV: test
        POSTGRES_USER: postgres
        POSTGRES_PASSWORD: password
        POSTGRES_DB: backend_test
        POSTGRES_HOST: postgres
      depends_on:
        - postgres
      ports:
        - "3000:3000"
      command: >
        sh -c "bundle exec rails db:prepare && \
              echo '✅ DB ready' && \
              bundle exec rails server -b 0.0.0.0 -p 3000"
              # bundle exec rails server -b 0.0.0.0 -p 3000 || \
              # (echo '❌ Rails failed to boot, dumping log:' && tail -n 50 log/test.log)"

    frontend:
      build:
        context: .
        dockerfile: frontend/Dockerfile.ci
      environment:
        NODE_ENV: test
        CI: "true"
        API_BASE: http://backend:3000/api/v1
      depends_on:
        - backend
      ports:
        - "3001:3000"
      command: npm run dev
  ```
- create `backend/Dockerfile.ci`:
  ```
  # backend/Dockerfile.ci
  # This is only for CircleCI testing, not Fly.io deployment
  ARG RUBY_VERSION=3.3.0
  ARG BUNDLER_VERSION=2.6.5

  FROM ruby:${RUBY_VERSION}-slim

  # Re-declare ARG before setting ENV
  ARG BUNDLER_VERSION
  ENV BUNDLER_VERSION=${BUNDLER_VERSION}

  WORKDIR /app

  # Install system dependencies
  RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
    build-essential \
    libpq-dev \
    libvips \
    curl \
    git \
    libjemalloc2 \
    && rm -rf /var/lib/apt/lists/*

  # Install matching bundler version using ENV (not ARG directly)
  RUN gem install bundler -v "$BUNDLER_VERSION"

  # Copy over Gemfiles and install dependencies
  COPY backend/Gemfile backend/Gemfile.lock ./
  RUN bundle _"$BUNDLER_VERSION"_ install

  # Copy backend app code
  COPY backend/ .
  ```
- create `frontend/Dockerfile.ci`:
  ```
  # frontend/Dockerf4ile.ci
  # This is only for CircleCI testing, not Fly.io deployment
  FROM node:21.7.1-slim

  WORKDIR /app

  # Install system dependencies for headless testing
  RUN apt-get update -qq && apt-get install --no-install-recommends -y \
    wget \
    gnupg \
    ca-certificates \
    libnss3 \
    libatk1.0-0 \
    libatk-bridge2.0-0 \
    libcups2 \
    libdrm2 \
    libxcomposite1 \
    libxdamage1 \
    libxrandr2 \
    libgbm1 \
    libasound2 \
    libxshmfence1 \
    libxfixes3 \
    libxrender1 \
    libxtst6 \
    libxss1 \
    xdg-utils \
    fonts-liberation \
    libgtk-3-0 \
    libcurl4 \
    python-is-python3 \
    && rm -rf /var/lib/apt/lists/*

  # Install dependencies
  COPY frontend/package*.json ./
  RUN npm ci

  # Install dev dependencies for Vitest and Playwright
  RUN npm install --save-dev @nuxt/test-utils vitest @vue/test-utils happy-dom playwright-core \
    @playwright/test pixelmatch playwright-expect

  # Copy app source code
  COPY frontend/ .

  # Install Playwright browser dependencies
  RUN npx playwright install --with-deps

  # Build Nuxt app
  RUN npm run build
  ```
- `bundle add dotenv-rails --group "development, test"`
- `bundle install`
- create `backend/.env`:
  ```
  POSTGRES_HOST=localhost
  POSTGRES_USER=postgres
  POSTGRES_PASSWORD=password
  POSTGRES_DB=backend_test
  RAILS_ENV=test
  ```
- at the top of `backend/spec/rails_helper.rb`, add this:
  ```
  require 'dotenv/load'
  ```
- also in `backend/spec/rails_helper.rb` right below `ENV['RAILS_ENV'] ||= 'test'`, add this:
```
`ENV['POSTGRES_HOST'] ||= 'postgres'`
```
- in `backend/Gemfile` on line 3, change `ruby "3.3.0"` to `ruby "~> 3.3"`
- in `backend/config/database.yml`, change the `test` section to:
  ```
  test:
    <<: *default
    database: backend_test
    host: <%= ENV.fetch("POSTGRES_HOST", "postgres") %>
    username: <%= ENV.fetch("POSTGRES_USER", "postgres") %>
    password: <%= ENV.fetch("POSTGRES_PASSWORD", "password") %>
    port: 5432
  ```
- in `frontend/package.json`, add this to the `scripts` section: `"vitest": "vitest",`
- edit your `frontend/nuxt.config.ts` to this:
  ```
  const isCI = process.env.CI === 'true'
  const isDev = process.env.NODE_ENV !== 'production'

  export default defineNuxtConfig({
    devServer: { port: 3001 },

    modules: [
      '@nuxt/eslint',
      '@nuxt/fonts',
      '@nuxt/icon',
      '@nuxt/image',
      '@nuxt/test-utils'
    ],

    runtimeConfig: {
      public: {
        apiBase: isCI
          ? 'http://backend:3000/api/v1'
          : isDev
            ? 'http://localhost:3000/api/v1'
            : 'https://app001-backend.fly.dev/api/v1'
      }
    },
  });
  ```
- `echo ".DS_Store\n.secrets\n.env" > .gitignore`
- `git init`
- `git add .`
- `git commit -m "Init app"`
- create an empty repo on github and note the git url (something like https://github.com/mark-mcdermott/rails-nuxt-flyio-circleci-mvp-app.git)
- `git remote add origin <git url>`
- `git push --set-upstream origin main`
- go to your CircleCI projects page (something like `https://app.circleci.com/projects/project-dashboard/github/mark-mcdermott/`)
- next to repo name (`drivetracks-wip-nuxt3-rework`), click Set Up Project
- click `Fastest` -> `main` -> `Set Up Project`
- CI frontend/backend tests should pass 🎉

- **Double Check Local Tests Stil Pass:** 
- `cd ~/app/backend && rspec` <-- rspec tests should pass locally 🎉
- `cd ~/app/frontend && vitest tests/components` <-- vitest tests should pass locally 🎉
- `cd ~/app/backend && rails server`
- `cd ~/app/frontend && npm run dev`
- `cd ~/app/frontend && npx playwright test tests/e2e` <-- playwright tests should pass locally 🎉

### 5. Production Deployment
- **Redeploy:**  
  Once tests pass locally and in CircleCI, deploy the backend and frontend to fly.io.
- `cd ~/app/backend`
- `fly deploy`
- `cd ../frontend`
- `fly deploy`

- **Verification:**  
  Visit the frontend prod URL to verify that the page displays the `{ "status": "OK" }` response from the backend.

## Part II: Auth

### 1. **Install and Configure Devise with JWT and Sidebase Auth**
- **Install Dependencies**
- `cd ~/app/backend`
- `rails db:create` (or `rails db:drop db:create` if already exists)
- `bundle add devise devise-jwt jsonapi-serializer`
- `bundle install`
- `rails generate devise:install`

- **Devise Configuration**
- In config/environments/development.rb:
  ```
  config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }
  ```
- In `config/initializers/devise.rb`, uncomment and change the line:
  ```
  config.navigational_formats = []
  ```
- In `config/application.rb`, add session store and middleware:
  ```
  config.session_store :cookie_store, key: '_interslice_session'
  config.middleware.use ActionDispatch::Cookies
  config.middleware.use config.session_store, config.session_options
  ```

  **Test**
- Verify if Devise is working by checking if the views are generated:
  ```
  rails generate devise:views
  ```

### 2. **Create User Model with Devise & JWT**
- `rails g migration EnableUuid`
- In db/migrate/<timestamp>_enable_uuuid.rb:
  ```
  enable_extension 'pgcrypto'
  ```
- `rails db:migrate`
- Generate User model with Devise:
  ```
  rails generate devise User
  ```
- In `db/migrate/<timestamp>_devise_create_users.rb`, update:
  ```
  t.boolean :admin, default: false
  t.uuid :uuid, index: { unique: true }
  ```
- `rails db:migrate`
- Factory for User Model:
  ```
  touch spec/factories/users.rb
  ```
- In spec/factories/users.rb:
  ```
  FactoryBot.define do
    factory :user do
      sequence(:email) { |n| "test#{n}@mail.com" }
      password { 'password' }
      trait :confirmed do
        confirmed_at { Time.zone.now }
      end
    end
  end
  ```

  **Test**
- Test if users can be created:
  ```
  rails console
  FactoryBot.create(:user)
  ```


### 3. **Create Registration and Login Endpoints**
- Generate Controllers:
  ```
  rails g devise:controllers api/v1/auth -c sessions registrations
  ```
- In config/routes.rb:
  ```
  devise_for :users, path: '', path_names: {
    sign_in: 'api/v1/auth/login',
    sign_out: 'api/v1/auth/logout',
    registration: 'api/v1/auth/signup'
  }, controllers: {
    sessions: 'api/v1/auth/sessions',
    registrations: 'api/v1/auth/registrations'
  }
  ```

  **Test**
- Ensure routes for login, logout, and signup are working:
  ```
  rails routes | grep auth
  ```

### 4. **JWT Configuration**
- JWT Setup in devise.rb (initializers/devise.rb):
  ```
  config.jwt do |jwt|
    jwt.secret = ENV['SECRET_KEY_BASE'] || 'dummy_secret_key_for_tests'
    jwt.dispatch_requests = [
      ['POST', %r{^/api/v1/auth/login$}]
    ]
    jwt.revocation_requests = [
      ['DELETE', %r{^/api/v1/auth/logout$}]
    ]
    jwt.expiration_time = 30.minutes.to_i
  end
  ```
- Migration to Add JWT Field:
  ```
  rails g migration addJtiToUsers jti:string:index:unique
  ```
- In `db/migrate/<timestamp>_add_jti_to_users.rb`:
  ```
  add_column :users, :jti, :string, null: false
  add_index :users, :jti, unique: true
  ```
- `rails db:migrate`

- **User Model with JWT**
- In app/models/user.rb:
  ```
  class User < ApplicationRecord
    include Devise::JWT::RevocationStrategies::JTIMatcher
    devise :database_authenticatable, :registerable,
          :recoverable, :rememberable, :validatable,
          :jwt_authenticatable, jwt_revocation_strategy: self
    before_create :set_uuid

    private

    def set_uuid
      self.uuid = SecureRandom.uuid if uuid.blank?
    end
  end
  ```

**Test**
- Check if users have a jti field:
  ```
  rails console
  user = User.find_by(email: "test@mail.com")
  puts user.jti
  ```

### 5. **Session Controller for Login**
- Create Sessions Controller:
  - In `app/controllers/api/v1/auth/sessions_controller.rb`:
  ```
  class Api::V1::Auth::SessionsController < ApplicationController
    def create
      user = User.find_by(email: params[:user][:email])
      if user&.valid_password?(params[:user][:password])
        token = Warden::JWTAuth::UserEncoder.new.call(user, :user, nil).first
        render json: { token: token, status: { code: 200, message: 'Logged in successfully.' } }
      else
        render json: { status: 401, message: 'Invalid email or password.' }, status: :unauthorized
      end
    end
  end
  ```

**Test**
- Test login endpoint with `curl` by posting the user credentials.

### 6. **Registration Controller**
- Create Registration Controller:
  - In `app/controllers/api/v1/auth/registrations_controller.rb`:
  ```
  class Api::V1::Auth::RegistrationsController < Devise::RegistrationsController
    respond_to :json

    def create
      user = User.new(user_params)
      if user.save
        render json: { status: { code: 200, message: "Signed up successfully." }, data: UserSerializer.new(user).serializable_hash[:data][:attributes] }
      else
        render json: { status: 422, message: user.errors.full_messages.to_sentence }, status: :unprocessable_entity
      end
    end

    private

    def user_params
      params.require(:user).permit(:email, :password)
    end
  end
  ```

**Test**
- Test the registration with `curl` by sending POST requests with valid user data.

### 7. **Current User Endpoint**
- Create Current User Controller:
  - In `app/controllers/api/v1/auth/current_user_controller.rb`:
  ```
  class Api::V1::Auth::CurrentUserController < ApplicationController
    before_action :authenticate_user!

    def index
      render json: UserSerializer.new(current_user).serializable_hash[:data][:attributes], status: :ok
    end
  end
  ```

**Test**
- Test the current user endpoint by sending a `GET` request with a valid token.

### 8. **User Seeds and Production Deployment**
- User Seeding:
  - In db/seeds.rb:
  ```
  User.create!(email: 'test@mail.com', password: 'password', admin: true)
  User.create!(email: 'test2@mail.com', password: 'password')
  ```

**Test**
- Test seeding in production by running `rails db:seed`.

**Deploy Backend to Fly.io**
- `cd ~/app/backend`
- `fly deploy`

### 9. Frontend Updates
- Nuxt Configuration for API Calls:
  - In nuxt.config.ts, update to use the correct backend URL:
  ```
  runtimeConfig: { 
    public: { 
      apiBase: process.env.API_BASE || '<backend url>/api/v1' 
    }
  }  
  ```

**Test**
- Verify that the frontend is correctly calling the backend endpoints and handles JWT authentication.

### 10. Swagger API Documentation
- Install Swagger:
  ```
  cd ~/app/backend
  bundle add rswag
  bundle install
  rails g rswag:install
  ```
- Generate swagger for controllers:
  ```
  rails generate rspec:swagger Api::V1::Auth::CurrentUserController
  rails generate rspec:swagger Api::V1::Auth::RegistrationsController
  rails generate rspec:swagger Api::V1::Auth::SessionsController
  rails generate rspec:swagger Api::V1::UsersController
  ```
- Run to generate docs:
  ```
  rake rswag:specs:swaggerize
  rails s
  ```
- In a browser, go to http://localhost:3000/api-docs to view the API docs.

**Test**
- Test Local: Ensure all parts are working by testing locally (curl, Postman, etc.).
- Test on CircleCI: Check CircleCI integration for backend and frontend tests.












## Final Thoughts

I hope you've enjoyed this tutorial and gotten everything working. If anything didn't work for you, feel free to message me or leave a comment here!

*(End of Tutorial)*
