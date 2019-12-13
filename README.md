# Setup a new Rails project using Docker and Compose
Based on https://docs.docker.com/compose/rails/
And https://github.com/pacuna/rails5-docker-alpine

## Goals:
- Be able to setup and run a new Rails project, without having the Ruby and Rails versions installed locally.
- Have an easy development setup that could be run in any machine.
- *(?) Easy production deployment.*

## Overview:
- Write a Dockerfile that will install Ruby, and the basic Rails dependencies inside the container.
- Set up a volume that will permit changes to the code made inside the container to reflect on our local folder.
- Run `rails new` inside the container, writing the Rails scaffold to our local folder.
- Increment the Dockerfile, and run it again to actually build the Rails dependencies into the container.
- Rerun the container to actually run the application.
- Configure the database connection.
- Finally rerun the application to have it working.

## Step-by-step:

- Write a minimal `Gemfile`, it will be overwritten by `rails new` later:
  ```Gemfile
  source 'https://rubygems.org'

  ruby '2.6.5'

  gem 'rails', '~> 6.0.1'
  ```
  
- Create an empty `Gemfile.lock`.

- Create a minimal `Dockerfile`:
  ```Dockerfile
  FROM ruby:2.6.5-alpine3.10

  # Minimal requirements to run a Rails app
  RUN apk add --no-cache --update \
    build-base \
    linux-headers \
    git \
    postgresql-dev \
    nodejs \
    tzdata \
    yarn \
    bash

  RUN mkdir /home/deploy
  WORKDIR /home/deploy

  COPY ./Gemfile ./Gemfile.lock ./

  RUN bundle install --jobs 3 --retry 3

  COPY . .
  ```
  
- Create the `docker-compose.yml`:
  ```yml
  version: '3'
  services:
    db:
      image: postgres:12.1-alpine
      expose:
        - "5432"
      volumes:
        - ./tmp/db:/var/lib/postgresql/data
    web:
      build: .
      command: rails s -p 3000 -b '0.0.0.0'
      volumes:
        - .:/home/deploy
      ports:
        - "3000:3000"
      expose:
        - "3000"
      depends_on:
        - db
  ```
  
- Run `docker-compose run web rails new . --database=postgresql --webpacker --webpack=react --skip-bundle --skip-webpack-install --force`
  - This shall build the container image using our `Dockerfile`, and run `rails new` inside it.
    But as we setted up a volume inside our `docker-compose.yml`, we shall get the Rails scaffold put into the our folder.
  - Attention to `--skip-bundle --skip-webpack-install`, as we are not writing these changes to the container image, it doesnt make sense to install the dependencies yet.
  
- Run `docker-compose build` to rebuild the container image.
  Now the `bundle install` that we have on the `Dockerfile` will install the dependencies written by `rails new` into the container image.

- Run `docker-compose up` to run the app. Access `localhost:3000` on your browser. **You should see an error** about not being able to connect to the database.

- Modify the `config/database.yml` created by `rails new`. Change it to match the configurations below. Pay attention that the `test` key will change from inheriting from `default` to inherit from `development`:
  ```yml
  default: &default
    adapter: postgresql
    encoding: unicode
    pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

  development: &development
    <<: *default
    database: deploy_development
    username: postgres
    password:
    host: db

  test:
    <<: *development
    database: deploy_test

  production:
    <<: *default
    database: deploy_production
    username: deploy
    password: <%= ENV['DEPLOY_DATABASE_PASSWORD'] %>
  ```

- Run the migrations, to create the database `docker-compose run web rails db:create`. We have a volume configured for the postgres container, so the data will be stored between executions. 

- Run the application again, `docker-compose up`. **Yay! You’re on Rails!**

- Later on, you can execute other things using the dependencies inside the container by running `docker-compose run web ...`
  
  - Eg. `docker-compose run web yarn add bootstrap jquery popper.js`
