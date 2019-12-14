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

RUN mkdir /home/banana
WORKDIR /home/banana

COPY ./Gemfile ./Gemfile.lock ./

RUN bundle install --jobs 3 --retry 3

COPY ./package.json ./yarn.lock ./

RUN yarn install && yarn cache clean

COPY . .
