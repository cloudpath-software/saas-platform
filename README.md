# Cloudpath saas platform
![Demo](example.gif?raw=true)
## Overview

Self hostable, secure and fast saas platform. Provides static site hosting, bucket file management and more is planned to be added.

## Installation

Setting up a local instance does require [docker](https://www.docker.com/). Install it on your machine if you haven't already.

Use the provided docker-compose.yml configuration to quickly spin up a local instance using the following command.

### Steps
 
- Clone this repo
- Create a .env file and copy the contents of the .env.example file. Replace variables with your credentials.
- Run the command below


```bash
docker-compose -f docker-compose.yml up
```

This command will setup a few backend services described below.

## Architecture

| Service name                       | Platform (version)                      | Is native image?     | Runtime jdk                                                                                                                                                                                       |
| ----------------------     | :------------------------: | :----------: | :----------------------------------------------------------:                                                                                                                                |
| `Api gateway`                  | `Spring Boot` (3.1.3)  | `true`       | graalvm-ce:ol7-java17-22.3.3                                                                               |
| `Public spa bff`               |  `Spring Boot` (3.1.3)         | `false`       | eclipse-temurin:17-jdk-focal |
| `Internal spa bff`               |  `Spring Boot` (3.1.3)         | `false`       | eclipse-temurin:17-jdk-focal|
| `Public oauth2`               |  `Spring Boot` (3.1.3)         | `true`       | graalvm-ce:ol7-java17-22.3.3|
| `Authorization service`               |  `Spring Boot` (3.1.3)         | `true`       | graalvm-ce:ol7-java17-22.3.3 |
| `Core service`               |  `Spring Boot` (3.1.3)         | `true`       | graalvm-ce:ol7-java17-22.3.3 |
| `Internal service`               |  `Spring Boot` (3.1.3)         | `true`       | graalvm-ce:ol7-java17-22.3.3 |
| `Admin service`               |  `Spring Boot` (3.1.3)         | `true`       | graalvm-ce:ol7-java17-22.3.3 |
| `Start service`               |  `Spring Boot` (3.1.3)         | `true`       | graalvm-ce:ol7-java17-22.3.3 |
| `Hosting service`               |  `Spring Boot` (3.1.3)         | `true`       | graalvm-ce:ol7-java17-22.3.3 |
| `Content delivery (cdn) service`               |  `Spring Boot` (3.1.3)         | `true`       | graalvm-ce:ol7-java17-22.3.3 |
| `Telemetry service`               |  `Spring Boot` (3.1.3)         | `true`       | graalvm-ce:ol7-java17-22.3.3 |

## What is a native image?

You can learn more about it [here](https://www.baeldung.com/spring-native-intro). In short, it's a technology to build java code to a standalone executable. At the expense of longer build times, the main pros are it's extremely fast startup time (typically fewer than 100ms) and lower memory consumption. 

The size of a native image executable is large, but future updates will seek to reduce them as much as possible.

## In depth services description

Overall this is a microservice architecture. It followed current best practices with regards to security and overall structure.

The internal process communication is done using client side load balancing. A mix of rest and event communication is used dependencing if the event can be handled asynchronously.

### External dependencies

| Name                 | Docker image                           |  Required  | Notes                                                   |
| -------------------- | :------------------------------------: | :--------: | :-----------------------------------------------------: |
| `Consul`             | `consul:1.15.4`                        |  yes  | Discovery service, provides loadbalancing and visibility on the health of services. |
| `RabbitMQ`           | `rabbitmq:3.12.3-management-alpine`    |  yes  | Lightweight messaging broker. Uses the Advanced Message Queuing Protocol (amqp) |
| `Redis`              | `redis:7.0-alpine`                     |  yes  | Extremely fast in memory key value database. Used as a cache.  |
| `Jaeger`             | `jaegertracing/all-in-one:latest`      |  no   | Provides distributed tracing. Visibility on the communicaiton between services  |
| `Grafana`            | `grafana/grafana:10.1.1`               |  no   | Provides services metrics dashboards. Easily display collected data in a nice dashboard.  |
| `prometheus`         | `prom/prometheus`                      |  no   | Data collector service. Periodically collects metrics (jvm memory, rabbitmq active connections, etc). Used by grafana.  |
| `Postgres`           | `postgres:15.4`                        |  yes  | Sql database. Used by all services.  |

### Api gateway

Public entry point. The api gateway is exposed by mapping the internal `443` docker port to the `443` host port. Has a few predefined routes.

| Domain routes                    | Referenced service     | Publicly exposed | Notes                                      |
| -------------------------------- | :-------:              | :--------------: | :----------------------------------------: |
| `api`.domain.extension           | `public-spa-bff`       | yes  | Main load balanced api endpoint.                     |
| `internal-api`.domain.extension  | `internal-spa-bff`     | no   | Used by internal web apps. Not exposed to the public                  |
| `oauth2`.domain.extension        | `Public oauth2 service`| yes  | Access public oauth2 resources (ex: jwk set uri)     |
| `cdn`.domain.extension           | `cdn service`          | yes  | Access public assets. (Ex: html files, images, etc.)     |
| `accounts`.domain.extension      | `authorization service`| yes  | Proxies authorization requests. Required for client side cors same origin requirements           |
| `telemetry`.domain.extension     | `telemetry service`    | yes  | Access telemery endpoints (Ex: site vitals endpoint)      |
| `{subdomain}`.domain.extension     | `hosting service`    | yes  | Public site files endpoint, used to host sites.      |

## Public services

### Public spa bff

This is a single page application (web) bff service. It expects to receive a session id through by the client cookies and retrieves the client's latest access token. If the access token is expired, an attempt to refresh it using the refresh token will be attempted.

After a valid access token is retrieved, the request is internally forwarded to the resource server with the bearer token.

`This service essentially allows to keep the client's access and refresh token within the backend. Storing the access and refresh token on the client is not recommended.
`

### Public OAuth2

Authorization server built on top of [spring authorization server](https://github.com/spring-projects/spring-authorization-server).

It handles signing jwt's with your private key & provides a jwk set uri at `.well-known/oauth-authorization-server`

## Internal services

### Internal spa bff

Similar to the public spa bff, only not intended to be publicly exposed. Will contain certain filters to check internal api access control. Admin stuff.

## Resource services

### Authorization

This service is an extension to the public oauth2 service. Handles authorization related crud user operations, session management, user credentials, etc. 

It was seperated to not expose this service to the public directly.

### Core

Manages core logic that other services depend on. Ex: organization reference, user workspaces & reference to the active user workspace.

### Internal

Intended to not be publically exposed. It manages internal setup and configuration logic.

### Admin

Similar to the internal service. Endpoints requires admin level access.

### Start

Used by the `start.domain.extension` web app. Manages providing visibility on available options, if theres updates & `eventually visibility on community products`.

### Hosting

Static web app hosting service. Manages crud website logic, domain management, etc. `The site hosting api is heavily inspired by Netlify's api`. Communicates with redis to cache site deploy files. Prevents unecessary db transactions and improves site load time.

### Content delivery (cdn)

Essentially a file system. Crud api to create folders, files, include a file in a specific folder, etc. This service is used by multiple services (ex: hosting, telemetry services)

### Telemetry

Provides an application agnostic api for telemetry related logic. At the moment, web apps are supported (javascript tracking url & script), but eventually, support for mobile app's can be added.

`The web app tracking script does not use cookies.`

#### Endpoints

| Path                   | Description                                             |
| ---------------------- | :-----------------------------------------------: |
| `{scheme}://telemetry.{domain}.{extension}/telemetry.js`    |  Retrieves the cdn hosted tracking script.                  |
| `{scheme}://telemetry.{domain}.{extension}/insight/vitals`  |  Send client telemetry data (user agent, path, etc.) | 

# FAQ

### Interested in being a beta tester? 

Send an email at <fabrizio.rodin-miron@cloudpath.app>. If you are willing to provide feedback, this will provide you with a reduced price for future paid features.

### Roadmap

At the moment, i'm working on this project alone. So no offical roadmap. Given interest, i'll look into making a more structured roadmap.

But in a overview, this is my goal, adding

-  A localisation service `(will be included for free)`. To automate translating web & mobile apps.

- Billing abstraction `(will be an opt in paid feature)`. Already built, just not ready yet. (managing all the webhook events is a pain!) Essentially adds an abstraction layer to supported payment providers (stripe, braintree, etc) to allow easily switching from one provider to another. Its possible to have customer 1 using stripe, and customer 2 using braintree for example. `The main problem this is attempting to solve is let's say that stripe increases their fees, and some cheaper alternative is available, swithing should be easy and painless. Providing us leverage to negociate assuming transaction volume is significant enough.` This will include tax management, generating and persisting invoice pdf's, creating pricing tables, processing one time payments or scheduling recurring subscription payments.

- Expanding on the community product showcasing, possibly a beta testing solution. 

This all depends on feedback and market needs tho. So we will see.

### Why was this built?

This was built as internal tools to have a great microservices architecture as a foundation to build saas products on top of... My mind set is I only have to build it all once and reuse it.

But I know that the odds of my products being successful in this market are low, so maybe the tech community can also built on top of it.

### Will it be open source?

Im open to the idea, I would need to make some connections with people that are willing to help manage it. I'm working alone on this at the moment, so it's easier if its closed source. Hiding my terrible commit messages! If your interested in managing open source code, message me at <fabrizio.rodin-miron@cloudpath.app>.