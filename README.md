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
    tzdata

  RUN mkdir /home/deploy
  WORKDIR /home/deploy

  COPY ./Gemfile ./Gemfile.lock ./

  RUN bundle install --jobs 3 --retry 3
  ```
- Create the `docker-compose.yml`:
  ```yml
  version: '3'
  services:
    db:
      image: postgres:12.1-alpine
    web:
      build: .
      command: rails s -p 3000 -b '0.0.0.0'
      volumes:
        - .:/home/deploy
      ports:
        - "3000:3000"
      depends_on:
        - db
  ```
- Run `docker-compose run web rails new . --database=postgresql --webpacker --webpack=react --no-deps`
  - This shall build the container image using our `Dockerfile`, and run `rails new` inside it.
    But as we setted up a volume inside our `docker-compose.yml`, we shall get the Rails scaffold put into the our folder.
