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
ENV['POSTGRES_HOST'] ||= 'postgres'
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
- `cd ~/app`
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
- In `db/migrate/<timestamp>_enable_uuuid.rb`:
  ```
  enable_extension 'pgcrypto'
  ```
- `rails db:migrate`
- Generate User model with Devise:
  ```
  rails generate devise User
  ```
- In `db/migrate/<timestamp>_devise_create_users.rb`, add:
  ```
  t.boolean :admin, default: false
  t.uuid :uuid, index: { unique: true }
  ```
- `rails db:migrate`
- Install Factory Bot:
  - `bundle add factory_bot_rails --group "development, test"`
  - `bundle install`
- Factory for User Model:
  ```
  mkdir spec/factories
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
  include FactoryBot::Syntax::Methods
  FactoryBot.create(:user)
  ```


### 3. **Create Registration and Login Endpoints**
- Generate Controllers:
  ```
  rails g devise:controllers api/v1/auth -c sessions registrations
  ```
- Edit `config/routes.rb` to look like this:
  ```
  Rails.application.routes.draw do
    devise_for :users, path: '', path_names: {
      sign_in: 'api/v1/auth/login',
      sign_out: 'api/v1/auth/logout',
      registration: 'api/v1/auth/signup'
    }, controllers: {
      sessions: 'api/v1/auth/sessions',
      registrations: 'api/v1/auth/registrations'
    }
    namespace :api do
      namespace :v1 do
        get 'up', to: 'health#show'
        namespace :auth do
        get 'login', to: 'sessions#create' 
        get 'signup', to: 'registrations#create'
        get 'logout', to: 'sessions#destroy'
      end
      end
    end
  end
  ```

  **Test**
- Ensure routes for login, logout, and signup are working:
  ```
  rails routes | grep auth
  ```

### 4. **JWT Configuration**
- JWT Setup in devise.rb (`initializers/devise.rb`). This goes *before* the final `end`:
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
- In `db/migrate/<timestamp>_add_jti_to_users.rb` replace the `add_column` and `add_index` lines with this:
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
  user = User.create!(email: 'test@mail.com', password: 'password')
  user = User.find_by(email: "test@mail.com")
  user.jti
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
- Test login endpoint with `curl` by posting the user credentials:
  ```
  curl -X POST http://localhost:3000/api/v1/auth/login \
    -H "Content-Type: application/json" \
    -d '{"user": {"email": "test@mail.com", "password": "password"}}'
  ```

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
- Add the user serializer:
  - Create `backend/app/serializers/user_serializer.rb`:
  ```
  # app/serializers/user_serializer.rb
  class UserSerializer
    include JSONAPI::Serializer
    attributes :email, :uuid, :admin
  end
  ```

**Test**
- Test the registration with `curl` by sending POST requests with valid user data:
  ```
  curl -X POST http://localhost:3000/api/v1/auth/signup \
    -H "Content-Type: application/json" \
    -d '{"user": {"email": "newuser@mail.com", "password": "password"}}'
  ```

### 7. **Current User Endpoint**
- Create Current User Controller:
  - In `app/controllers/api/v1/auth/current_user_controller.rb`:
  ```
  class Api::V1::Auth::CurrentUserController < ApplicationController
    before_action :authenticate_user!

    def index
      render json: current_user.slice(:id, :email)
    end
  end
  ```
- Edit `config/routes.rb` to look like this:
  ```
  Rails.application.routes.draw do
    devise_for :users, path: '', path_names: {
      sign_in: 'api/v1/auth/login',
      sign_out: 'api/v1/auth/logout',
      registration: 'api/v1/auth/signup'
    }, controllers: {
      sessions: 'api/v1/auth/sessions',
      registrations: 'api/v1/auth/registrations'
    }
    namespace :api do
      namespace :v1 do
        get 'up', to: 'health#show'
        namespace :auth do
        get 'login', to: 'sessions#create' 
        get 'signup', to: 'registrations#create'
        get 'logout', to: 'sessions#destroy'
        get 'current_user', to: 'current_user#index'
      end
      end
    end
  end
  ```

**Test**
- Test the current user endpoint by sending a `GET` request with a valid token:
  - First login and copy the returned token:
  ```
  curl -X POST http://localhost:3000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"user": {"email": "test@mail.com", "password": "password"}}'
  ```
  - Then paste your token in place of the `<your_jwt_token>` here and run this:
  ```
  curl -X GET http://localhost:3000/api/v1/auth/current_user \
  -H "Authorization: Bearer <your_jwt_token>"
  ```

### 8. **User Seeds and Production Deployment**
- User Seeding:
  - In db/seeds.rb:
  ```
  User.create!(email: 'test@mail.com', password: 'password', admin: true)
  User.create!(email: 'test2@mail.com', password: 'password')
  ```

**Test**
- Test seeding:
  - First reset the database:
  ```
  rails db:drop db:create db:migrate
  ```
  - Then run the seeds:
  ```
  rails db:seed
  ```
  - Then check the seeds:
  ```
  rails console
  User.all
  ```

**Deploy Backend to Fly.io**
- `cd ~/app/backend`
- `fly deploy`

### 9. Setup Frontend Auth
- `cd ~/app/frontend`
- Create `frontend/composables/useAuth.ts`:
  ```
  import { ref, computed } from 'vue'

  const token = ref(null)
  const user = ref(null)

  export const useAuth = () => {
    const login = async (email: string, password: string) => {
      try {
        const res = await $fetch('/auth/login', {
          method: 'POST',
          baseURL: useRuntimeConfig().public.apiBase,
          body: { user: { email, password } },
        })
        token.value = res.token
        localStorage.setItem('token', res.token)
        await fetchCurrentUser()
        return true
      } catch (err) {
        console.error('Login failed', err)
        return false
      }
    }

    const logout = () => {
      token.value = null
      user.value = null
      localStorage.removeItem('token')
    }

    const fetchCurrentUser = async () => {
      if (!token.value) return null
      try {
        console.log('Fetching user...')
        const res = await $fetch('/auth/current_user', {
          baseURL: useRuntimeConfig().public.apiBase,
          headers: {
            Authorization: `Bearer ${token.value}`
          }
        })
        console.log('Fetched user:', res)
        user.value = res
      } catch (err) {
        console.error('Failed to fetch current user', err)
        logout()
      }
    }

    const status = computed(() => {
      return token.value ? 'authenticated' : 'guest'
    })

    onMounted(() => {
      const savedToken = localStorage.getItem('token')
      if (savedToken) {
        console.log('client token:', savedToken)
        token.value = savedToken
        fetchCurrentUser()
      }
    })

    return { token, user, login, logout, fetchCurrentUser, status }
  }
  ```

10. Add Index/Login/Signup/Private Pages
- Create `pages/index.vue`
  ```
  <template>
    <div>
      <h1>Welcome</h1>
      <blockquote class="blockquote blockquote--highlight">
        <p>That's good thinking there, Cool Breeze.</p>
      </blockquote>
      <p>
        Cool Breeze is a kid with three or four days' beard
        sitting next to me on the stamped metal bottom of the
        open back part of a pickup truck. Bouncing along.
        Dipping and rising and rolling on these rotten springs
        like a boat. Out the back of the truck the city of San 
        Francisco is bouncing down the hill, all those endless
        staggers of bay windows, slums with a view, bouncing
        and streaming down the hill. One after another, electric
        signs with neon martini glasses lit up on them, the San
        Francisco symbol of "bar"—thousands of neon-magenta
        martini glasses bouncing and streaming down the hill,
        and beneath them hundreds, thousands of people
        wheeling around to look at this freaking crazed truck
        we're in, their white faces erupting from their lapels
        like marshmallows—streaming and bouncing down the
        hill—and God knows they've got plenty to look at.
        The acid tests are in full swing, and you're about to be part of something <em>truly radical</em>.
      </p>
      <NuxtLink class="button" to="/signup">Join the Trip</NuxtLink>
    </div>
  </template>
  ```
- Create `pages/login.vue`:
  ```
  <template>
    <div>
      <h1>Login</h1>
      <form class="form" @submit.prevent="onSubmit">
        <label class="form__label" for="email">Email</label>
        <input id="email" v-model="email" class="form__input" type="text" placeholder="Your email" >

        <label class="form__label" for="password">Password</label>
        <input id="password" v-model="password" class="form__input" type="password" placeholder="Your password" >

        <button type="submit" class="button">Log In</button>
      </form>
    </div>
  </template>

  <script setup>
  import { ref } from 'vue'
  import { useAuth } from '~/composables/useAuth'

  const email = ref('')
  const password = ref('')
  const { login } = useAuth()

  const onSubmit = async () => {
    const success = await login(email.value, password.value)
    if (success) {
      navigateTo('/private')
    } else {
      alert('Login failed')
    }
  }
  </script>
  ```
- Create `pages/signup.vue`:
  ```
  <template>
    <div>
      <h1>Register</h1>
      <form class="form" @submit.prevent="register">
        <label class="form__label" for="email">Email</label>
        <input id="email" v-model="email" class="form__input" type="text" placeholder="Your email" >

        <label class="form__label" for="password">Email</label>
        <input id="password" v-model="password" class="form__input" type="password" placeholder="Your password" >

        <button type="submit" class="button">Sign Up</button>
      </form>
    </div>
  </template>

  <script setup>
  import { ref } from 'vue'
  import { useAuth } from '~/composables/useAuth'

  const email = ref('')
  const password = ref('')
  const { login } = useAuth()

  const register = async () => {
    try {
      await $fetch('/auth/signup', {
        method: 'POST',
        baseURL: useRuntimeConfig().public.apiBase,
        body: { user: { email: email.value, password: password.value } }
      })

      const success = await login(email.value, password.value)
      if (success) {
        navigateTo('/')
      } else {
        alert('Registered but login failed 🤷‍♂️')
      }
    } catch (err) {
      alert('Signup failed')
    }
  }
  </script>
  ```
- Create `pages/private.vue`:
  ```
  <template>
    <div>
      <h1>Private</h1>
      <blockquote class="blockquote blockquote--highlight">
        <p>"Jack&mdash;"</p>
      </blockquote>
      <p>
        And Frenchy hunkers down on the floor and
        opens the cheese spread and pulls the knife out of the
        scabbard and sinks the blade into it. Quite a blade! a
        foot long and engraved with Chinese demons. He wipes
        gobs of cheese spread onto his tongue with the blade.
        Sandra sits silent in a clump, grooving on the full life.
        Jack raps on about perfidy in high places . . .
      </p>
      <NuxtLink class="button" to="/signup">Join the Trip</NuxtLink>
    </div>
  </template>
  ```

- Create `components/HeaderNav.vue`:
  ```
  <script setup>
  import { useAuth } from '~/composables/useAuth'
  import { useRouter } from 'vue-router'

  const { logout, status, user, fetchCurrentUser } = useAuth()
  const router = useRouter()

  onMounted(() => {
    if (status.value === 'authenticated' && !user.value) {
      fetchCurrentUser()
    }
    console.log('client token:', localStorage.getItem('token'))
    console.log('auth status:', status.value)
    console.log('user:', user.value)
  })

  const handleLogout = () => {
    logout()
    router.push('/')
  }

  if (import.meta.client) {
    watch(user, () => {
      console.log('🧠 User updated:', user.value)
    })
  }
  </script>

  <template>
    <div>
      <nav>
        <ul>
          <li><NuxtLink to="/"><Icon class="logo" name="gg:pacman" /></NuxtLink></li>
          <li><NuxtLink to="/">Home</NuxtLink></li>

          <ClientOnly fallback=" ">
            <li v-if="status === 'authenticated'">
              <NuxtLink to="/private">Private</NuxtLink>
            </li>
            <li v-if="status === 'guest'">
              <NuxtLink to="/login">Login</NuxtLink>
            </li>
            <li v-if="status === 'guest'">
              <NuxtLink to="/signup">Register</NuxtLink>
            </li>
            <li v-if="status === 'authenticated'">
              <button @click="handleLogout">Logout</button>
            </li>
          </ClientOnly>
        </ul>

        <ClientOnly fallback=" ">
          <div v-if="status === 'authenticated' && user?.email" class="user-email">
            User logged in: <strong>{{ user.email }}</strong>
          </div>
          <div v-else>
            No users logged in
          </div>
        </ClientOnly>
      </nav>
    </div>
  </template>

  <style scoped>
  nav {
    padding: 10px 10px 10px 0;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  .logo {
    position: relative;
    bottom: 3px;
  }
  .user-email {
    margin-left: auto;
    white-space: nowrap;
  }
  ul {
    list-style: none;
    display: flex;
    padding-inline-start: 0;
    gap: 20px;
  }
  li {
    display: inline;
  }
  li a, li button {
    border: none;
    background: none;
    cursor: pointer;
    font: inherit;
    color: inherit;
  }
  li a:hover, li button:hover {
    color: #19e2b5;
    text-decoration: underline;
  }
  li a span.iconify {
    vertical-align: middle;
    font-size: 2.5rem;
  }
  </style>
  ```

- Edit `app.vue` to look like this:
  ```
  <template>
    <div class="container">
      <HeaderNav />
      <NuxtPage />
    </div>
  </template>
  ```

- Create a navigation guard in `frontend/middleware/auth.global.ts`:
  ```
  export default defineNuxtRouteMiddleware((to) => {
    if (import.meta.client) {
      const token = localStorage.getItem('token')
      if (to.path === '/private' && !token) {
        return navigateTo('/login')
      }
    }
  })
  ```

- edit your `frontend/nuxt.config.ts` to this:
  ```
  const isCI = process.env.CI === 'true'
  const isDev = process.env.NODE_ENV !== 'production'

  export default defineNuxtConfig({
    app: { head: { link: [{ rel: 'stylesheet', href: 'https://npmcdn.com/comet-css@1.2.0/dist/comet.min.css' }]}},
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

**Test Locally**
- Test out the UI:
  ```
  cd backend && rails s
  cd frontend && npm run dev
  ```

**Test Prod**
- redeploy and test
  ```
  cd backend && fly deploy
  cd frontend && fly deploy
  ```

### 11. Swagger API Documentation
- Install Swagger:
  ```
  cd ~/app/backend
  bundle add rswag
  bundle install
  rails g rswag:install
  ```

- Set server urls in `backend/spec/swagger_helper.rb`:
  ```
  # frozen_string_literal: true

  require 'rails_helper'

  RSpec.configure do |config|
    config.openapi_root = Rails.root.join('swagger').to_s
    config.openapi_specs = {
      'v1/swagger.yaml' => {
        openapi: '3.0.1',
        info: {
          title: 'API V1',
          version: 'v1'
        },
        paths: {},
        servers: [
          {
            url: 'http://localhost:3000',
            description: 'Local development'
          },
          {
            url: 'https://app001-backend.fly.dev',
            description: 'Production API server'
          }
        ]
      }
    }
    config.openapi_format = :yaml
  end
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
- `fly deploy`

**Test**
- In a browser, go to https://app001-backend.fly.dev/api-docs to view the API docs.

## Part III: Flutter

## Add Flutter With WebViews
- `cd` into the root folder of our app (`app/`)if you're not there already
- `flutter create flutter_app`
- `cd flutter_app`
- `flutter pub add webview_flutter`
- Let's modify `flutter_app/lib/main.dart` to load our web app (*And make sure to swap in our `<frontend web url>` towards the bottom*):
```
import 'package:flutter/material.dart';
import 'package:webview_flutter/webview_flutter.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return const MaterialApp(
      debugShowCheckedModeBanner: false,
      home: WebViewScreen(),
    );
  }
}

class WebViewScreen extends StatefulWidget {
  const WebViewScreen({super.key});

  @override
  State<WebViewScreen> createState() => _WebViewScreenState();
}

class _WebViewScreenState extends State<WebViewScreen> {
  late final WebViewController _controller;

  @override
  void initState() {
    super.initState();
    _controller = WebViewController()
      ..setJavaScriptMode(JavaScriptMode.unrestricted)
      ..loadRequest(Uri.parse("https://app001-frontend.fly.dev"));
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea( 
        child: WebViewWidget(controller: _controller),
      ),
    );
  }
}
```
- Let's setup an android emulator:
  - Open android studio
  - Tools -> Device Manager
  - Click the "plus" icon to create a new device -> Create Virtual Device
  - Pick the Pixel XL API 33 -> Next
  - Click the Additional Settings tab
  - Scroll down to the bottom where it says Emulated Performance
  - For graphics acceleration, select Hardware
  - For RAM enter `4` for GB
  - Click Finish
  - Start the emulator you just created in the Device Manager area by clicking the play icon to the right of your new emulator's name
- Let's setup an iPhone simulator:
  - Open XCode
  - XCode -> Open Developer Tool -> Simulator
  - That should open the iPhone simulator
- `flutter devices` <- you should see both the android emulator and the iphone simulator listed there (and probably other stuff too).
  - note the ids of the android emulator and the iphone simulator. The id is the first thing to the right of the device name. For me the android emulator id is `emulator-5554` and the iphone simulator id is `36C6816E-25E8-4669-9505-2A9A2BC9CD47`
- open two terminal tabs
  - in the first tab run `flutter run -d <iphone simulator id>`
    - this takes a few minutes
    - mid-run, this may print an error which can be ignored: "the following plugin(s) depend on a different Android NDK..."
  - in the second tab run `flutter run -d <android emulator id>`
    - this takes a few minutes

## Part IV: S3 Preparation

### AWS S3 Setup
Now we'll create our AWS S3 account so we can store our user avatar images there as well as any other file uploads we'll need. There are a few parts here. We want to create a S3 bucket to store the files. But a S3 bucket needs a IAM user. Both the S3 bucket and the IAM user need permissions policies. There's a little bit of a chicken and egg issue here - when we create the user permissions policy, we need the S3 bucket name. But when we create the S3 bucket permissions, we need the IAM user name. So we'll create everything and use placeholder strings in some of the policies. Then when we're all done, we'll go through the policies and update all the placeholder strings to what they really need to be.

#### AWS General Setup
- login to AWS (https://aws.amazon.com)
  - If you don't have an AWS account, you'll need to sign up. It's been awhile since I did this part - I think you have to create a root user and add you credit card or something. Google it if you run into trouble with this part.
- at top right, select a region if currently says `global` (I use the `us-east-1` region). If all the region options are grayed out, ignore this for now and we'll set it later.
- at top right click your name
  - next to Account ID, click the copy icon (two overlapping squares)
  - paste your Account ID in your `~/app/.secrets` file (It pastes without the dashes. Leave it that way - you need it without the dashes.)

#### AWS User Policy
- in searchbar at top, enter `iam` and select IAM
- click `Policies` in the left sidebar under Access Managment
  - click `Create policy` towards the top right
  - click the `JSON` tab on Policy Editor
  - under Policy Editor select all with `command + a` and then hit `delete` to clear out everything there
  - enter this under Policy Editor (we'll update it shortly, once we have our user and bucket names):
```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "AllowGetObject",
			"Effect": "Allow",
			 "Action": [
          "s3:PutObject",
          "s3:GetObject",
          "s3:DeleteObject"
            ],
			"Resource": ["arn:aws:s3:::<development bucket name>", "arn:aws:s3:::<production bucket name>"]
		}
	]
}
```

  - click Next towards bottom right
  - for Policy Name, enter `app-s3-user-policy`
  - paste your policy name in your `.secrets` file
  - click Create Policy towards the bottom right

#### AWS User
- click `Users` under Access Management in the left sidebar
  - click `Create User` towards the top right
  - enter name, something like `app-s3-user` (add this to your `.secrets` file - you'll need it later)
  - click Next
  - under Permissions Options click `Attach policies directly`
  - in the search bar under Permissions Policies, enter `app-s3-user-policy` -> this should then show the policy we just created above (`app-s3-user-policy`) under Policy Name
  - under Policy Name, click the checkbox to the left of `app-s3-user-policy`
  - click Next
  - click Create User towards the bottom right
- under Users, click the name of the user we just created (`app-user`)
  - click Security Credentials tab
  - click `Create Access key` towards the top right
    - Use case: `Local code`
    - check `I understand the above recommendation`
    - Next
    - Description tag value: enter tag name, like `app-user-access-key`
    - click `Create access key` towards the bottom right
    - paste both the Access Key and the Secret Access Key into your `.secrets` file - this is important!
    - click Done

#### AWS S3 Bucket
- in searchbar at top, enter `s3` and select S3
- Create Bucket
  - for Bucket Name, enter something like `app-s3-bucket-development` (below when you click Create Bucket, it may tell you this bucket already exists and you will have to make it more unique. Regardless, add this to your `.secrets` file - you'll need it later. Also, make sure it ends in `-development`.)
  - under Object Ownership, click ACLs Enabled
  - under Block Public Access settings
    - uncheck the first option, `Block All Public Access`
    - check the last two options, `Block public access to buckets and objects granted through new public bucket or access point policies` and `Block public and cross-account access to buckets and objects through any public bucket or access point policies`
  - check `I acknowledge that the current settings might result in this bucket and the objects within becoming public.`  
  - scroll to bottom and click Create Bucket (if nothing happens, scroll up and look for red error messages)
- under General Purpose Buckets, click the name of the bucket you just created -> then click the Permissions tab towards the top
  - in the Bucket Policy section, click Edit (to the right of "Bucket Policy")
  - copy/paste the boilerplate json bucket policy in the next line below this into the text editor area under Policy.
  - here is the boilerplate json bucket policy:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<aws acct id without dashes>:user/<iam username>"
            },
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::<bucket name>"
        },
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<aws acct id without dashes>:user/<iam username>"
            },
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:PutObjectAcl"
            ],
            "Resource": "arn:aws:s3:::<bucket name>/*"
        }
    ]
}
```

  - Update all the `<aws acct id without dashes>`, `<iam username>` and `<bucket name>` parts in the policy now in the text editor area under Policy with the account number, user name and bucket name you jotted down above in your `~/app/.secrets` file.
  - click Save Changes towards the bottom right
  - in the Cross-Origin Resource Sharing (CORS) section, click `Edit` (to the right of "Cross-origin resource sharing (CORS)")
  - under Cross-origin Resource Sharing (CORS) add this:
```
[
    {
        "AllowedHeaders": [
            "*"
        ],
        "AllowedMethods": [
            "GET",
            "POST",
            "PUT",
            "DELETE"
        ],
        "AllowedOrigins": [
            "*"
        ],
        "ExposeHeaders": [],
        "MaxAgeSeconds": 3000
    }
]
```

  - click Save Changes towards the bottom right
- now repeat this entire "AWS S3 Bucket" step above again, but make a production s3 bucket named something like `app-s3-bucket-production` and note the production bucket name in your `.secrets` file
- now that we know our bucket names, let's update the our user policy with the bucket name
  - in the searchbar at the top of the page, type `iam` and select `IAM`
  - click `Policies` in the left sidebar under Access Management
  - in the searchbar under Policies, type `app-s3-user-policy` -> click `app-s3-user-policy` under Policy Name
  - click Edit towards the top right
  - in the Policy Editor text editor area, change the line `"Resource": ["arn:aws:s3:::<development bucket name>", "arn:aws:s3:::<production bucket name>"]` replace `<development bucket name>` and `<production bucket name>` with your development bucket name and production bucket name, respectively, in your `.secrets` file
  - click Next towards the bottom right
  - click Save Changes towards the bottom right
- see what region you're logged into
  - click the AWS logo in the top left
  - in the top right there will be a region dropdown - click it
  - look at the highlighted region in the dropdown and look for the region string to the right of it - something like `us-east-1`
  - paste your region string in your `~/app/.secrets` file in the `aws region` line
- we're now done with our S3 setup and our AWS dashboard, at least for now. So let's go back to our terminal where we're building out our rails backend

## Part V: Setup S3 In Rails/Nuxt

### S3 Manual Avatar Upload
- If you don’t already have the AWS CLI installed, run
```
brew install awscli
```
- Then run this to configure the AWS CLI:
```
aws configure
```
  - when prompted, enter these:
    - Your Access Key ID
    - Your Secret Access Key
    - Default region name: us-east-1 (or whatever you’re using)
    - Default output format: json
- Put a test avatar image (`avatar.png`) on your desktop
- Run this to upload the avatar to your S3:
```
aws s3 cp ~/Desktop/avatar.png s3://app001-s3-bucket-production/avatars/avatar.png --acl public-read
```
- Double check it uploaded:
```
aws s3 ls s3://app001-s3-bucket-production/ --recursive
```
- the url for the image is:
```
https://<bucket-name>.s3.<region>.amazonaws.com/avatars/avatar.png
```
  - so for me that's:
  ```
  https://app001-s3-bucket-production.s3.us-east-1.amazonaws.com/avatars/avatar.png
  ```

### Hardcode avatar url in Nuxt
- Update the `<ClientOnly>` part of `frontend/components/HeaderNav.vue` to this:
```
<ClientOnly fallback=" ">
  <div v-if="status === 'authenticated' && user?.email" class="user-email">
    User logged in: <img src="https://app001-s3-bucket-production.s3.us-east-1.amazonaws.com/avatars/avatar.png" ><strong>{{ user.email }}</strong>
  </div>
  <div v-else>
    No users logged in
  </div>
</ClientOnly>
```
- And update the `.user-email` part of the css in `frontend/components/HeaderNav.vue` to this:
```
.user-email {
  margin-left: auto;
  white-space: nowrap;

  img {
    width: 30px;
    height: 30px;
    border-radius: 50%;
    vertical-align: middle;
    margin: 0 8px;
  }
}
```
- Test it out locally:
```
cd backend && rails s
cd frontend && npm run dev
```
- Then redeploy and test on prod:
```
cd frontend && fly deploy
```

### Pull avatar url from backend to frontend

- Let's add an avatar url string column to our user model on the backend:
  - `cd backend`
  - `rails g migration AddAvatarToUsers avatar:string`
  - `rails db:migrate`
- Let's update our `backend/db/seeds.db`:
```
User.create!(email: 'test@mail.com', password: 'password', avatar: 'https://app001-s3-bucket-production.s3.us-east-1.amazonaws.com/avatars/avatar.png', admin: true)
User.create!(email: 'test2@mail.com', password: 'password')
```
- Update `app/serializers/user_serializer.rb` to include the avatar url:
```
class UserSerializer
  include JSONAPI::Serializer
  attributes :email, :uuid, :admin, :avatar
end
```
- Update `app/controllers/api/v1/auth/current_user_controller.rb` to include the avatar url:
```
class Api::V1::Auth::CurrentUserController < ApplicationController
  before_action :authenticate_user!
  def index
    render json: UserSerializer.new(current_user).serializable_hash[:data][:attributes]
  end
end
```
- Now let's the `ClientOnly` part of `frontend/components/HeaderNav.vue` to:
```
<ClientOnly fallback=" ">
  <div v-if="status === 'authenticated' && user?.email" class="user-email">
    User logged in: <img v-if="user?.avatar" :src="user.avatar" ><strong>{{ user.email }}</strong>
  </div>
  <div v-else>
    No users logged in
  </div>
</ClientOnly>
```

- Test it out locally:
```
cd backend && rails db:drop db:create db:migrate db:seed
cd backend && rails s
cd frontend && npm run dev
```
- Redeploy:
- `cd ~/app/backend && fly deploy`
- `cd ~/app/frontend && fly deploy`
- `fly console`
- `user = User.find(1)`
- `user.update!(avatar: "https://app001-s3-bucket-production.s3.us-east-1.amazonaws.com/avatars/avatar.png")`
- Then test it out in prod







## Final Thoughts

I hope you've enjoyed this tutorial and gotten everything working. If anything didn't work for you, feel free to message me or leave a comment here!

*(End of Tutorial)*
