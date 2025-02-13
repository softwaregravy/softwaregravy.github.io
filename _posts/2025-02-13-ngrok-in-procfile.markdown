---
layout: post
title: "Automating ngrok to start with Rails"
date: 2025-02-13 06:50:00 -0500
categories: rails development ngrok
---

Here's how to automatically start ngrok alongside your Rails server, properly configured for development.

## The Problem

I want everything for my development efforts to start with a single command `./bin/dev`. 
I need my dev server to be accessible from the internet (hence the use of ngrok).

## The Solution

### 0. Environment Setup

Ensure your ngrok authentication token is set in your environment.
```sh
$ echo $NGROK_AUTHTOKEN
3Tjd3QU3nSIA8OHjEA9urhPGu5h3u_CKZhEw1KjKQb9O
# not my real authtoken, nice try "hackers"
```

### 1. Configure ngrok endpoint

Go to your ngrok account and set up a domain name, e.g., `example.ngrok.dev`

### 2. Configure Rails Hosts

[Rails configurations](https://guides.rubyonrails.org/configuring.html#configuring-middleware) requires allowlisting domains. 
Add ngrok's domain to your development configuration in `config/environments/development.rb`:

```ruby
config.hosts << ".ngrok.dev" 
# or lockdown for just your domain
# config.hosts << "example.ngrok.dev"
```

### 3. (Optional) Create Local ngrok Config

If you need to set or overwrite any configs specific to your project, create a local ngrok config file ([NGrok Config Reference](https://ngrok.com/docs/agent/config/v3)).
Create `.ngrok` in your project root with minimal settings:

```yaml
version: 3
agent:
  # required because my system ngrok.yml file has console_ui: true
  console_ui: false
```

### 4. Modify bin/dev

From [Foreman Docs](https://ddollar.github.io/foreman/):
```
The $PORT value starts as the base port as specified by -p, then increments by 
100 for each new process line. Multiple instances of the same process are 
assigned $PORT values that increment by 1.
```

Update `bin/dev` to make sure our ngrok process gets the same port as Rails. **We assume the Rails Webserver is listed first in the Procfile.**

```bash
export PORT="${PORT:-3000}" # this exists
export NGROK_PORT=$PORT # we add this on the next line
```

### 5. Set Up Procfile.dev

Add ngrok to your `Procfile.dev`:

```
web: bin/rails server -p $PORT
css: bin/rails tailwindcss:watch
ngrok: bash -c 'echo "Using port: $NGROK_PORT" && ngrok http --log stdout --url=example.ngrok.dev $NGROK_PORT --config="${HOME}/Library/Application Support/ngrok/ngrok.yml" --config=.ngrok'
```

Notes:
* (Optional) Having the wrong port configured is the most error-prone aspect of this setup due to foreman's port assignment, so we print it out
* (Optional) Log to stdout to show the port - if you enable this, you could remove the initial echo
* (Optional) Configs are loaded in order specified, so later configs overwrite earlier ones. This order has your local `.ngrok` overwrite any options in the system ngrok

Here's a minimal working example:
```
ngrok: ngrok http --url=example.ngrok.dev $NGROK_PORT 
```

## And Now

Now I can type `./bin/dev` and not open ngrok in a separate tab.

### Next Steps

This works for me, but I need to read the URL out of my .env file to make it transferrable across my team or projects.
