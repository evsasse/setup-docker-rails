# Setup a new Rails project using Docker and Compose

## Goals:
- Be able to setup and run a new Rails project, without having the Ruby and Rails versions installed locally.
- Have an easy development setup that could be run in any machine.
- *(?) Easy production deployment.*

## Easy mode

- Clone this repo on your "projects" folder.
  - `cd ~/projects && git clone https://github.com/evsasse/setup-docker-rails.git`.

- Create a new folder for your new Rails project, with the content on the `banana` folder.
  Let's pretend the name of your project is `my-new-orange-project`.
  - `cp -R setup-docker-rails/banana my-new-orange-project`.

- Enter the new folder, and find and replace every `banana` string on the files by your project's name.
  - `cd my-new-orange-project`;
  - `find . -type f -exec sed -i '' 's/banana/my-new-orange-project/g;s/BANANA/MY_NEW_ORANGE_PROJECT/g' {} +`.

- Build the container image, and create a new Rails project from inside it.
  - `docker-compose run rails rails new . --database=postgresql --webpacker --webpack=react --skip-git --force`.

- Replace your `config/database.yml` with one that will reach into the Postgres container.
  - `mv fixed-database.yml config/database.yml`.

- Replace your `config/webpacker.yml` with one that will reach into the Webpack container.
  - `mv fixed-webpacker.yml config/webpacker.yml`

- Rebuild the image with the new dependencies added by `rails new`.
  - `docker-compose build`.

- Run the project. You should see the usual **Yay! You’re on Rails!** page.
  - `docker-compose up`.

- This is a good moment to make your initial commit. And send it to a remote repository.
  - `git init`;
  - `git commit -m "Initial commit"`;
  - `git add remote origin https://github.com/evsasse/my-new-orange-project.git`;
  - `git push origin master`.

- Later on, you may want to add Ruby or JS dependencies.
  You should not need to rebuild the container image to do so, in development.
  - Just `docker-compose run rails bundle add devise`, and `docker-compose run rails yarn add jquery`;
  - Or `docker-compose run rails bundle install`, and `docker-compose run rails yarn install`, as needed.

- Similarly, you may want to run some migrations.
  - Just `docker-compose run rails rails db:migrate` or `docker-compose run rails rails db:drop db:create db:migrate db:seed`, etc.

- You can also already remove the clone of this repository.
  - `rm -Rf ../setup-docker-rails`

- To be able to interact with `binding.pry`, attach a interactive session to the running Rails server.
  - `bin/pry` does just that.
  - Consider also changing `config/puma.rb` to just 1 thread on development.

## Details:
For debugging, and studying.

### Overview:
- Write a Dockerfile that will install Ruby, and the basic Rails dependencies inside the container.
- Set up a volume that will permit changes to the code made inside the container to reflect on our local folder.
- Run `rails new` inside the container, writing the Rails scaffold to our local folder.
- Fix the database connection generated by `rails new`, to be able to reach into the Postgres container that will be run alongside.
- Fix the webpacker configuration, to reach into the Webpack container that will be run alongside.
- Rebuild the container image to actually build the Rails dependencies into the container.
- Rerun the container to actually run the application.

### Steps:

- Create a `.gitignore` and an identical `.dockerignore`, just by inputting `Rails` on https://gitignore.io

- Write a minimal `Gemfile`, it will be overwritten by `rails new` later:
  ```Gemfile
  source 'https://rubygems.org'

  ruby '2.6.5'

  gem 'rails', '~> 6.0.2'
  ```

- Create an empty `Gemfile.lock`.

- Create a minimal `package.json`, it will be overwritten by `rails new` later.
  ```JSON
  {
    "name": "my-new-orange-project"
  }
  ```

- Create an empty `yarn.lock`.

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

  RUN mkdir /home/my-new-orange-project
  WORKDIR /home/my-new-orange-project

  COPY ./Gemfile ./Gemfile.lock ./

  RUN bundle install --jobs 3 --retry 3

  COPY ./package.json ./yarn.lock ./

  RUN yarn install && yarn cache clean

  COPY . .
  ```

- Create the `docker-compose.yml`:
  ```yml
  version: '3'
  services:
    postgres:
      image: postgres:12.1-alpine
      expose:
        - "5432"
      volumes:
        - database_data:/var/lib/postgresql/data
    rails:
      build: .
      command: bash -c "rm ./tmp/pids/server.pid ; rails db:prepare ; rails s -p 3000 -b '0.0.0.0'"
      volumes:
        - .:/home/banana
        - node_modules:/home/banana/node_modules
        - packs:/home/banana/public/packs
      ports:
        - "3000:3000"
      expose:
        - "3000"
      depends_on:
        - postgres
        - webpack
      tty: true
      stdin_open: true
    webpack:
      build: .
      command: bash -c "yarn install ; bin/webpack-dev-server"
      command: bin/webpack-dev-server
      volumes:
        - .:/home/banana
        - node_modules:/home/banana/node_modules
        - packs:/home/banana/public/packs
      ports:
        - "3035:3035"
      expose:
        - "3035"
  volumes:
    database_data:
    node_modules:
    packs:
  ```

- Run `docker-compose run rails rails new . --database=postgresql --webpacker --webpack=react --skip-git --force`
  - This shall build the container image using our `Dockerfile`, and run `rails new` inside it.
    But as we setted up a volume inside our `docker-compose.yml`, we shall get the Rails scaffold put into our local folder.

- Run `docker-compose up` to run the app. Access `localhost:3000` on your browser. **You should see an error** about not being able to connect to the database.

- Modify the `config/database.yml` created by `rails new`. Change it to match the configurations below.
  - Pay attention that the `test` key will change from inheriting from `default` to inherit from `development`.
  - The most important lines are the configuration of username, password and host of the development server.
  ```yml
  ...
  development: &development
    <<: *default
    database: banana_development
    username: postgres
    password:
    host: postgres

  test:
    <<: *development
    database: banana_test
  ...
  ```

- Run the migrations, to create the database `docker-compose run rails rails db:prepare`.
  We have a volume configured for the postgres container, so the data will be stored between executions.

- Run the application again, `docker-compose up`. **Yay! You’re on Rails!**

- As you develop on it, you will see that the JS/CSS assets are being provided by the Rails application running Webpacker on-the-fly.
  We should provide the assets using our Webpack container.
  Change the `config/webpacker.yml` to match the following:
  - The most important line is the configuration of host of the development server.
  ```yml
  ...
  development:
    <<: *default

    dev_server:
      https: false
      host: webpack
      port: 3035
      public: localhost:3035
  ...
  ```

- Later on, you can execute other things using the dependencies inside the container by running `docker-compose run rails ...`

  - Eg. `docker-compose run rails yarn add bootstrap jquery popper.js`

# References:
 - https://docs.docker.com/compose/rails/
 - https://github.com/pacuna/rails5-docker-alpine
 - https://gist.github.com/briankung/ebfb567d149209d2d308576a6a34e5d8
 - https://stackoverflow.com/a/54033752/4949153
