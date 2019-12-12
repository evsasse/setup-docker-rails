# Setup a new Rails project using Docker and Compose
Based on https://docs.docker.com/compose/rails/

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
