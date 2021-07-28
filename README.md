# Rails API + Nuxt.js
Reference by [this](https://qiita.com/and__/items/d59409bfb7a35d363898) page

## 1. Directory

```Bash
root $ git init

root $ mkdir {api,front}

root $ cd api

api $ touch {Dockerfile,Gemfile,Gemfile.lock}

api $ ls
Dockerfile  Gemfile  Gemfile.lock
```

## 2. Dockerfile, Gemfile

### 2-1. api/

```Bash:api/Dockerfile
FROM ruby:2.7.1-alpine

ARG WORKDIR

ENV RUNTIME_PACKAGES="linux-headers libxml2-dev make gcc libc-dev nodejs tzdata postgresql-dev postgresql git" \
    DEV_PACKAGES="build-base curl-dev" \
    HOME=/${WORKDIR} \
    LANG=C.UTF-8 \
    TZ=Asia/Tokyo

# ENV test（このRUN命令は確認のためなので無くても良い）
RUN echo ${HOME}

WORKDIR ${HOME}

COPY Gemfile* ./

RUN apk update && \
    apk upgrade && \
    apk add --no-cache ${RUNTIME_PACKAGES} && \
    apk add --virtual build-dependencies --no-cache ${DEV_PACKAGES} && \
    bundle install -j4 && \
    apk del build-dependencies

COPY . .

CMD ["rails", "server", "-b", "0.0.0.0"]
```

```ruby:api/Gemfile
source 'https://rubygems.org'
gem 'rails', '~> 6.0.0'
```

### 2-2. front/

```Bash:front/Dockerfile
FROM node:14.4.0-alpine

ARG WORKDIR
ARG CONTAINER_PORT

ENV HOME=/${WORKDIR} \
    LANG=C.UTF-8 \
    TZ=Asia/Tokyo \
    HOST=0.0.0.0

# ENV check（このRUN命令は確認のためなので無くても良い）
RUN echo ${HOME}
RUN echo ${CONTAINER_PORT}

WORKDIR ${HOME}

EXPOSE ${CONTAINER_PORT}
```

## 3. root setting

```Bash:root
root $ touch {.gitignore,.env,docker-compose.yml}

root $ ls -a
.                  .git               api
..                 .gitignore         docker-compose.yml
.env               README.md          front
```

```Bash:.gitignore
/.env
```

```Bash:.env
# commons
WORKDIR=app
CONTAINER_PORT=3000
API_PORT=3000
FRONT_PORT=8080

# db
POSTGRES_PASSWORD=password
```

```Bash:docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:12.3-alpine
    environment:
      TZ: UTC
      PGTZ: UTC
      POSTGRES_PASSWORD: $POSTGRES_PASSWORD
    volumes:
      - ./api/tmp/db:/var/lib/postgresql/data

  api:
    build:
      context: ./api
      args:
        WORKDIR: $WORKDIR
    environment:
      POSTGRES_PASSWORD: $POSTGRES_PASSWORD
    volumes:
      - ./api:/$WORKDIR
    depends_on:
      - db
    ports:
      - "$API_PORT:$CONTAINER_PORT"

  front:
    build:
      context: ./front
      args:
        WORKDIR: $WORKDIR
        CONTAINER_PORT: $CONTAINER_PORT
    command: yarn run dev
    volumes:
      - ./front:/$WORKDIR
    ports:
      - "$FRONT_PORT:$CONTAINER_PORT"
    depends_on:
      - api
```

## 4. Docker image

Launch the Docker app. Create Docker image.

```Bash
root $ docker-compose build
root $ docker images
```

## 5. Rails app

### 5-1. create new Rails app

```Bash
root $ docker-compose run --rm api rails new . -f -B -d postgresql --api

# Rebuilding api directory for update Gemfile
root $ docker-compose build api
```

### 5-2. Rails DB setting

```Bash:api/config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  host: db            # add 
  username: postgres  # add(default name) 
  password: <%= ENV["POSTGRES_PASSWORD"] %>  # add 
    ...
```

```Bash
root $ docker-compose run --rm api rails db:create
```

```Bash
# start Rails
root $ docker-compose up api

# access this url
http://localhost:3000

# stop Rails
control + c
root $ docker-compose ps

# delete container
root $ docker-compose rm -f
root $ docker-compose ps

# or this command
root $ docker-compose down
root $ docker-compose ps

Name   Command   State   Ports
------------------------------
```

#### setting DB passcode

```Bash
root $ docker-compose up -d db
root $ docker-compose exec -u username db psql
```

```Bash
postgres=# ALTER USER username WITH PASSWORD 'new passcode';

# check
postgres=# SELECT * FROM pg_shadow;

# quit
postgres=# \q
```

```Bash
docker-compose down
```

```Bash:api/config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  host: db          
  username: postgres
  password: new passcode # replace 
  ...
```

## 6. Nuxt.js App

### 6-1. create new Nuxt.js App

```bash
root $ docker-compose run --rm front yarn create nuxt-app
```

```bash
# start Nuxt
root $ docker-compose up front

# access this url
http://localhost:8080

# stop Nuxt
control + c
root $ docker-compose ps

# delete container
root $ docker-compose stop
root $ docker-compose rm -f
root $ docker-compose ps

# or this command
root $ docker-compose down
root $ docker-compose ps
Name   Command   State   Ports
------------------------------
```

## 7. Rails API x Nuxt.js

### 7-1. setting Environment variable

```Bash:docker-compose.yml
api:
    build:
      context: ./api
      args:
        WORKDIR: $WORKDIR
    environment:
      POSTGRES_PASSWORD: $POSTGRES_PASSWORD
      API_DOMAIN: "localhost:$FRONT_PORT"       # add 

  ...

front:
    build:
      context: ./front
      args:
        WORKDIR: $WORKDIR
        CONTAINER_PORT: $CONTAINER_PORT
        API_URL: "http://localhost:$API_PORT"   # add
```

```Bash:front/Dockerfile

FROM node:14.4.0-alpine

ARG WORKDIR
ARG CONTAINER_PORT
# 追加
ARG API_URL

ENV HOME=/${WORKDIR} \
    LANG=C.UTF-8 \
    TZ=Asia/Tokyo \
    # \ 追記
    HOST=0.0.0.0  \
    # 追加
    API_URL=${API_URL}

# ENV check（このRUN命令は確認のためなので無くても良い）
RUN echo ${HOME}
RUN echo ${CONTAINER_PORT}
# 追加
RUN echo ${API_URL}

WORKDIR ${HOME}

EXPOSE ${CONTAINER_PORT}
```

```Bash
# Rebuilding api directory for update Dockerfile 
root $ docker-compose build api
```

### 7-2. create API on Rails

```Bash
# controller
root $ docker-compose run --rm api rails g controller api::v1::hello
```

```ruby:api/app/controllers/api/v1/hello_controller.rb
class Api::V1::HelloController < ApplicationController
  def index
    render json: "Hello"
  end
end 
```

```ruby:api/config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      # api test action
      resources :hello, only:[:index]
    end
  end
end
```

```Bash
# start Rails
root $ docker-compose up api

# access this url
http://localhost:3000/api/v1/hello

# delete container
control + c
root $ docker-compose down
root $ docker-compose ps
```

### 7-2. setting axios on Nuxt.js

```Bash
# install axios
root $ docker-compose run --rm front yarn add @nuxtjs/axios

front $ touch plugins/axios.js
```

```JavaScript:front/plugins/axios.js
export default ({ $axios }) => {
  // リクエストログ
  $axios.onRequest((config) => {
    console.log(config)
  })
  // レスポンスログ
  $axios.onResponse((config) => {
    console.log(config)
  })
  // エラーログ
  $axios.onError((e) => {
    console.log(e.response)
  })
}
```

```JavaScript:front/nuxt.config.js
...
plugins: [
  'plugins/axios' // add 
],
```

```JavaScript:front/nuxt.config.js
...
axios: {
  // サーバーサイドで行うリクエストに使用されるURL
  // baseURL: process.env.API_URL
  // クライアントサイドで行うリクエストに使用されるURL(デフォルト: baseURL)
  // browserBaseURL: <URL>
},
```

```JavaScript:front/pages/index.vue
<template>
  <div>
    <button
      type="button"
      name="button"
      @click="getMsg"
    >
      RailsからAPIを取得する
    </button>
    <div
      v-for="(msg, i) in msgs"
      :key="i"
    >
      {{ msg }}
    </div>
  </div>
</template>

<script>
export default {
  data () {
    return {
      msgs: []
    }
  },
  methods: {
    getMsg () {
      this.$axios.$get('/api/v1/hello')
        .then(res => this.msgs.push(res))
    }
  }
}
</script>
```

### 7-3. setting CORS counterplan

```Bash:docker-compose.yml
api:
    build:
      context: ./api
      args:
        WORKDIR: $WORKDIR
    environment:
      POSTGRES_PASSWORD: $POSTGRES_PASSWORD
      API_DOMAIN: "localhost:$FRONT_PORT"       # add

  ...

front:
    build:
      context: ./front
      args:
        WORKDIR: $WORKDIR
        CONTAINER_PORT: $CONTAINER_PORT
        API_URL: "http://localhost:$API_PORT"   # add
```

```ruby:api/Gemfile
# active line 26
gem 'rack-cors'
```

```Bash
# Rebuilding api directory for update Gemfile
root $ docker-compose build api

# check
root $ docker-compose run --rm api bundle info rack-cors
```

```ruby:api/config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins ENV["API_DOMAIN"] || ""

    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end
end
```
