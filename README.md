# Minimal Rails API & Nuxt 3 App Tutorial

## Overview

In this tutorial, you'll build a lean Rails API (backend) and a Nuxt 3 app (frontend), both hosted on fly.io.

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
  const development = process.env.NODE_ENV !== 'production'
  export default defineNuxtConfig({
    devServer: { port: 3001 },
    modules: [
      '@nuxt/eslint',
      '@nuxt/fonts',
      '@nuxt/icon',
      '@nuxt/image',
      '@nuxt/test-utils'
    ],
    public: {
      apiBase: development ? 'http://localhost:3000/api/v1' : 'https://app001-backend.fly.dev/api/v1',
    }
  })
  ```
  `cd ~/app/backend && rails server`
  `cd ~/app/frontend && npm run dev`

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
      get "/api/v1/up"
      expect(response).to have_http_status(:success)
      expect(JSON.parse(response.body)['status']).to eq('OK')
    end
  end
  ```
  `rspec`

#### Vitest Component Test (Frontend)
- `cd ~/app/frontend`
- `npm i --save-dev vitest @vue/test-utils happy-dom`
- **Component Test:** Create `tests/component/HomeContent.nuxt.js`:
  ```js
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
`npx playwright test tests/e2e`

### 4. CircleCI Integration
- `cd ~/app`
- **Minimal CircleCI Config:** Create `.circleci/config.yml`:
  ```yaml
  version: 2.1
  jobs:
    test:
      docker:
        - image: cimg/ruby:3.3      # Backend container
        - image: node:18            # Frontend container
          name: node      
        - image: cimg/postgres:15.2 # PostgreSQL container
          name: postgres 
          environment:
            POSTGRES_USER: postgres
            POSTGRES_PASSWORD: password
            POSTGRES_DB: backend_test
            POSTGRES_HOST: postgres # 👈 Forces TCP connection for Rails tests
      steps:
        - checkout

        # Set up Ruby in the backend folder
        - run:
            name: Set up Ruby & Install Gems
            working_directory: backend
            command: |
              bundle config set path 'vendor/bundle'
              bundle install

        # Wait for PostgreSQL to be ready
        - run:
            name: Wait for PostgreSQL to be ready
            command: |
              for i in {1..30}; do
                if PGPASSWORD=password psql -h postgres -U postgres -d postgres -c '\q' 2>/dev/null; then
                  echo "PostgreSQL is ready"
                  exit 0
                fi
                echo "Waiting for PostgreSQL..."
                sleep 5
              done
              echo "PostgreSQL did not become ready in time" && exit 1


        # Set up database
        - run:
            name: Setup Database
            working_directory: backend
            environment:
              RAILS_ENV: test
              POSTGRES_HOST: postgres
            command: |
              echo "Checking PostgreSQL connection..."
              until PGPASSWORD=password psql -h postgres -U postgres -d postgres -c '\q' 2>/dev/null; do
                echo "PostgreSQL is still starting up..."
                sleep 5
              done
              echo "PostgreSQL is up and running!"
              
              bin/rails db:create || echo "Database already exists"
              bin/rails db:migrate

        # Run RSpec tests in the backend folder
        - run:
            name: Run RSpec tests
            working_directory: backend
            environment:
              RAILS_ENV: test
              POSTGRES_HOST: postgres
            command: bundle exec rspec

        # Run frontend steps inside the Node.js container
        - run:
            name: Install Node.js dependencies
            working_directory: frontend
            environment:
              NODE_ENV: test
            command: npm install

        # Run Vitest in the frontend folder
        - run:
            name: Run Vitest
            working_directory: frontend
            command: npm run vitest -- --run

        # Run Playwright in the frontend folder
        - run:
            name: Run Playwright tests
            working_directory: frontend
            command: npx playwright test

  workflows:
    version: 2
    build_and_test:
      jobs:
        - test
  ```
- in `backend/Gemfile` on line 3, change `ruby "3.3.0"` to `ruby "~> 3.3.0"`
- in `backend/config/database.yml`, change the `default` section to:
  ```
  default: &default
    adapter: postgresql
    encoding: unicode
    pool: 5
    host: <%= ENV.fetch("POSTGRES_HOST", "localhost") %>
    username: <%= ENV.fetch("POSTGRES_USER", "postgres") %>
    password: <%= ENV.fetch("POSTGRES_PASSWORD", "password") %>
  ```
  and make sure the `test` section looks like this (TODO: it probably looks like this already, if so delete this part of tut):
  ```
  test:
    <<: *default
    database: backend_test
    host: <%= ENV.fetch("POSTGRES_HOST", "postgres") %>
  ```
- in `backend/spec/rails_helper.rb`, add `ENV['POSTGRES_HOST'] ||= 'postgres'` above `ActiveRecord::Migration.maintain_test_schema!`
- `echo ".DS_Store\n.secrets" > .gitignore`
- `git init`
- `git add .`
- `git commit -m "Init app"`
- create an empty repo on github and note the git url (something like https://github.com/mark-mcdermott/rails-nuxt-flyio-circleci-mvp-app.git)
- `git remote add origin <git url>`
- `git push --set-upstream origin main`
- go to your CircleCI projects page (something like `https://app.circleci.com/projects/project-dashboard/github/mark-mcdermott/`)
- next to repo name (`drivetracks-wip-nuxt3-rework`), click Set Up Project
- click `Fastest` -> `main` -> `Set Up Project`

### 5. Production Deployment
- **Redeploy:**  
  Once tests pass locally and in CircleCI, deploy the backend and frontend to fly.io.
- `cd ~/app/backend`
- `fly deploy`
- `cd ../frontend`
- `fly deploy`

- **Verification:**  
  Visit the frontend URL to verify that the page displays the `{ "status": "OK" }` response from the backend.

### 6. Incremental Authentication Setup
- **Add Sidebase Nuxt Auth (Frontend):**  
  Run `nuxi module add @sidebase/nuxt-auth` and configure it in your `nuxt.config.ts` so that pages are protected by default (except for public pages).
- **Add Devise with JWT (Backend):**  
  - Install `devise` and `devise-jwt`.
  - Generate the necessary controllers for sessions and registrations.
  - Write tests to verify login, logout, and current_user endpoints.
- **Test Incrementally:**  
  Validate each new authentication feature locally and in CircleCI before moving on.

### 7. Environment Variables & Simplified Configurations
- **Using .env Files:**  
  Utilize `dotenv` to load environment variables in both frontend and backend. For example, in your Nuxt config:
  ```js
  import dotenv from 'dotenv'
  dotenv.config()

  export default defineNuxtConfig({
    runtimeConfig: {
      public: {
        apiBase: process.env.API_BASE || 'http://localhost:3000/api/v1'
      }
    }
  })
  ```
- **Simplify Docker/Compose:**  
  Create a minimal `docker-compose.yml` for production and CI:
  ```yaml
  version: '3'
  services:
    backend:
      build: ./backend
      environment:
        - RAILS_ENV=production
        - SECRET_KEY_BASE=${SECRET_KEY_BASE}
      ports:
        - "3000:3000"
    frontend:
      build: ./frontend
      environment:
        - NODE_ENV=production
        - API_BASE=${API_BASE}
      ports:
        - "3001:3000"
  ```

## Final Thoughts

Build your app like a workout: one exercise at a time. Verify each step with tests and CI before moving on to the next feature. This method ensures you stay strong and never overwhelmed. Happy coding—and remember, you're building something amazing one rep at a time!

*(End of Tutorial)*
