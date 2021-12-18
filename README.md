# Reverse proxy for online store app

This is a reverse proxy used for an online store app designed for the Distributed Systems course at University of Helsinki.

The proxy is running on Heroku at [https://distsys-backend.herokuapp.com](https://distsys-backend.herokuapp.com).

## Service URIs

- Customer service: `/api/customer` (for endpoints, see [here](https://github.com/emmalait/customer-node))
- Cart and order service: `/api/cartorder` (for endpoints, see [here](https://version.helsinki.fi/ese/distributedsystems/-/tree/main/cartorder))
- Product service: `/api/product` (for endpoints, see [here](https://version.helsinki.fi/ese/distributedsystems/-/tree/main/product))

## Deployment

The proxy is deployed on Heroku using Heroku's [nginx buildpack](https://github.com/heroku/heroku-buildpack-nginx).

## Configuration

The configuration (config/nginx.config.erb) is based on Heroku's [example config](https://help.heroku.com/YTWRHLVH/how-do-i-make-my-nginx-proxy-connect-to-a-heroku-app-behind-heroku-ssl).

In order to get the reverse proxy to work with services running on Heroku, the Host header needs to be set as <app-name>.herokuapp.com, e.g. `proxy_set_header Host customer-node.herokuapp.com;`. There is no straightforward way of defining this when using upstreams, so this is implemented by having a separate "local" server instance for each Heroku app, running in a dedicated port. These server instances are then used as upstreams for the proxy, which enables using nginx's load balancing. Load balancing is done in round-robin fashion.

For example, the customer service is provided by two instances of the customer node running on separate Heroku instances:

```
server {
    listen 8001 default_server;
    server_name _;
    location ~ ^/api/customer/(.*)$ {
        proxy_pass https://customer-node.herokuapp.com/customer-api/$1;
        proxy_set_header Host customer-node.herokuapp.com;
    }
}

server {
    listen 8002 default_server;
    server_name _;
    location ~ ^/api/customer/(.*)$ {
        proxy_pass https://customer-node-2.herokuapp.com/customer-api/$1;
        proxy_set_header Host customer-node-2.herokuapp.com;
    }
}
```

An upstream is defined for the customer service, which distributes incoming requests for both servers:

```
upstream customer_service {
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
}
```

The main server then uses the customer_service upstream for all incoming requests to uri /api/customer:

```
location ~ ^/api/customer {
    proxy_pass http://customer_service;
}
```
