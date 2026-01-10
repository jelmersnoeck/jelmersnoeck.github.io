---
title: "Build a container for your static JavaScript application"
date: 2018-02-28
---

This article was originally posted on [the Manifold blog](https://manifold.co/blog/building-a-production-grade-container-for-your-static-javascript-application-b2b2eff83fbd)

![js-container](/images/js-container-000.png)

In this blog post we'll look at how you can build a production grade container for your static JavaScript application and how you can use a single container image for all your environments.

To do this, we need to go over several steps. Here we'll look at how to prepare your application for production inside a container, make it possible to run this across multiple environments and actually run the container.

For this blogpost, we'll assume you're already familiar with the [Manifold CLI tool](https://blog.manifold.co/manage-your-cloud-services-like-a-developer-with-our-cli-tool-f02c88d9a7fd) and are able to set up your resources. We've also set up a [demo application](https://github.com/manifoldco/static-react-demo-app) with [React](https://reactjs.org/) for you to follow along with.

## Serving static files within a container

Using [React](https://reactjs.org/) for our frontend application means that we could just serve static files to our users. We don't have to run this through a [Node.js](https://nodejs.org/en/) server.

To do this, we'll use [nginx](https://www.nginx.com/) to serve static content from within a container. This means we can have a very lightweight container image which will serve our files quickly and be resource friendly.

Generally speaking, when building an application, you have environment specific credentials for third party services like [Stripe](https://stripe.com) or [Segment.io](https://segment.com/). When minimising your JavaScript files, you need to inline these credentials so you can serve this to your users. This causes your image to be aware of the environment it needs to run in, which is not desirable as you want to be able to reuse these images. Therefore, we'll look at making an environment agnostic container image.

## Building the static files

First off, we'll have to generate our static files. This can be done through [Docker](https://www.docker.com/) as well. As an example, we have a [very simple React application](https://github.com/manifoldco/static-react-demo-app) which just serves a "Hello World!" example.

To build our files, we can use Yarn as follows:

```
manifold run yarn build
```

This will build all our static files according to our configuration and put the static files under the build/ directory. Our environment variables will be injected through the manifold run command and placed inline. These compiled files are what nginx will serve to our audience.

For this to work, we need to set up our .env file to read from our environment:

```
REACT_APP_STRIPE_KEY=${STRIPE_KEY}
REACT_APP_SEGMENT_KEY=${SEGMENT_KEY}
```

The dotenv package that ships with our react application expects our environment variables to be prefixed with REACT_APP_ so we set that up in the .env file and use the ${} annotation to read our actual environment variables.

## Building a production grade container

The next step is actually building the production container which will make it possible to serve our application. We'll use nginx to serve our application and there is a pre-made nginx image for Docker which we can use to create our own image. First, we'll need to create an nginx configuration file which knows how to serve our static files. In your root directory, create a file nginx.conf.

```
server {
    listen 80;
    server_name _;

    root /var/www/;
    index index.html;

    # Force all paths to load either itself (js files) or go through index.html.
    location / {
        try_files $uri /index.html;
    }
}
```

This is a very simplistic configuration which will try to match a requested URL to a file under the /var/www/ directory first. If this file is not available, it will serve index.html which in its turn will know how to handle routes etc. through the underlying JS application.

To actually build our nginx container, we can provide a Dockerfile.

```dockerfile
FROM nginx:stable

COPY ./build/ /var/www
COPY ./nginx.conf /etc/nginx/conf.d/default.conf

CMD ["nginx -g 'daemon off;'"]
```

We can also use Docker to build the static files and use the multi-stage functionality to define all of this within a single Dockerfile.

```dockerfile
FROM node AS build

WORKDIR /app
COPY . .

RUN yarn build

FROM nginx:stable

COPY --from=build /app/build/ /var/www
COPY ./nginx.conf /etc/nginx/conf.d/default.conf

CMD ["nginx -g 'daemon off;'"]
```

In this example we'll first use a temporary Docker container to build the static files. In our nginx build, we'll copy the generated files over which will then be embedded into the image. This means that we don't need to have Node installed in our final container, only Nginx and the static files.

To actually build the image, you can run the following command:

```
docker build -t manifoldco/demo-app:latest .
```

The downside for this image is that it is environment specific. When we built our static files with manifold run yarn build, we got the credentials for a specific environment through the manifold-cli and injected those into our static files. This means that we can't use this specific image for any other environment.

## Making your static files dynamic

To use the same image across different environments, we'll have to make sure our injected configuration can be different. Normally, you'd inject environment variables into your running container which your application can detect. Since we're serving static files, our application — being a static file — can't detect these environment variables.

What we'll have to do in this case is override the values we've set up in our static files with placeholder values which we can replace at runtime with the values we want for our environment.

First of all, we'll recreate our static files with placeholder values instead of the actual credentials. To do so, we can utilise the sed tool to replace our key values with a placeholder value. We want to give these a distinct name so we can easily replace them later on.

```
cat .env | grep = | sort | sed -e 's|REACT_APP_\([a-zA-Z_]*\)=\(.*\)|REACT_APP_\1=NGINX_REPLACE_\1|' > .env.local
```

This will replace all our values with the key prefixed by NGINX_REPLACE_. This value will then be used to build the static files which means that the files have these prefixed strings as values, which we can later search for to replace within our container. By putting the outcome in a .env.local file, we tell our build system to use this file over the regular .env file.

We can now use Nginx's sub_filter functionality in combination with envsubst. We'll use the sub_filter functionality within our configuration file to tell the nginx configuration "if this string is present in any of our JS files, replace it with this new value". We'll use the sed tool again to make it possible to dynamically create this configuration file from the keys we have in our manifold project.

```
location ~* ^.+\.js$ {
    LOCATION_SUB_FILTER
    sub_filter_once off;
    sub_filter_types *;
}
```

If we put this in our nginx.conf.sample file, we can use sed to replace LOCATION_SUB_FILTER to configure our environment variables.

First, we'll make sure we format our environment variables into the sub_filter format.

```
export NGINX_SUB_FILTER=$(cat .env | grep '=' | sort | sed -e 's/REACT_APP_\([a-zA-Z_]*\)=\(.*\)/sub_filter\ \"NGINX_REPLACE_\1\" \"$$\{\1\}\";/')
```

Now, we can put this in place in our nginx.conf file.

```
cat nginx.conf.sample | sed -e "s|LOCATION_SUB_FILTER|$(echo $NGINX_SUB_FILTER)|" | sed 's|}";\\ |}";\n\t\t|g'
```

This will generate a config file that has a bunch of sub_filter items set up.

```
location ~* ^.+\.js$ {
    sub_filter "NGINX_REPLACE_SEGMENT_KEY" "${SEGMENT_KEY}";
    sub_filter "NGINX_REPLACE_STRIPE_KEY" "${STRIPE_KEY}";
    sub_filter_once off;
    sub_filter_types *;
}
```

We can now use the envsubst tool to start nginx with the correct keys in place. By doing so, envsubst will look at the provided file and see where we want to replace environment variables. If it finds matches, it'll replace it what is provided in the environment. This allows us to inject ENVs into our container which will be loaded into our static files. This also opens up the possibility to add some extra configuration to our nginx.conf.sample file.

```
server {
    listen ${PORT};
    server_name ${SERVER_NAME};
    root /var/www/;
    index index.html;

    location ~* ^.+\.js$ {
        LOCATION_SUB_FILTER
        sub_filter_once off;
        sub_filter_types *;
    }

    location / {
        try_files $uri /index.html;
    }
}
```

To do so, we need to start our docker container by pre-running envsubst. Our Dockerfile will reflect this change.

```dockerfile
FROM nginx:stable

COPY --from=build /app/build/ /var/www

RUN rm /etc/nginx/conf.d/default.conf
COPY ./nginx.conf /etc/nginx/conf.d/default.conf.template

# This is a hack around the envsubst nginx config. Because we have `$uri` set
# up, it would replace this as well. Now we just reset it to its original value.
ENV uri \$uri

# Default configuration
ENV PORT 80
ENV SERVER_NAME _

CMD ["sh", "-c", "envsubst < /etc/nginx/conf.d/default.conf.template > /etc/nginx/conf.d/default.conf && nginx -g 'daemon off;'"]
```

Your container is now environment agnostic, meaning that it doesn't know environment specific details. This means that you can now use a different STRIPE_KEY or SEGMENT_KEY for your test, staging, and production environment without having to recompile your files and create a new Docker image.

### Why use Nginx to replace the values?

One question that might arise is why use nginx to do the string replacement and not replace it within the file itself using envsubst?

We've decided on this approach due to 2 specific reasons. The first reason is that when inlining the ENV as an environment variable on build time causes issues. The dotenv package we use replaces $, { and } values with "". This causes our replaced value to result in just the key value.

This can be overcome by replacing those values within the container with a prefixed $. This is where the second issue arises. In our codebase, we have snippets like $t and $e. These are not environment variables and should thus not be replaced, however, envsubst will replace these. To not have to set up exceptions for every variable like this — and possible more in the future — we've decided to let nginx handle this.

## Running your container

Having everything baked into this image makes it easy now to run your project anywhere. Be it locally, on CI or on your production environments. To illustrate, we'll use docker-compose to start the container and use manifold-cli to inject the credentials.

First, we'll have to create a docker-compose.yml file which represents how we want to run our service.

```yaml
version: '3'

services:
  dashboard:
    image: manifoldco/demo-app
    ports:
      - 3001:3001
    environment:
      - PORT=${PORT:-3001}
      - STRIPE_KEY=${STRIPE_KEY}
      - SEGMENT_KEY=${SEGMENT_KEY}
```

We can now run our container by using running manifold run -p frontend-application docker-compose up. This will inject all the credentials set up in our frontend-application project on Manifold into our container.

## Development and Deployment

Now that you have a container that is ready for production, 2 questions remain. What to do for development and how can we deploy this container?

For development we still recommend using yarn start as this gives you the benefit of automatic code reloads versus having to wait for your application to be minimised every time. You could potentially run this within a Docker container as well, as [illustrated here](https://gist.github.com/przbadu/4a62a5fc5f117cda1ed5dc5409bd4ac1).

Deploying your application heavily depends on where you host your applications and falls out of scope for this blogpost. There are however several guides available that show you how to deploy your container to [Kubernetes](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/) or [Heroku](https://devcenter.heroku.com/articles/container-registry-and-runtime) for example.
