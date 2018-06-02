# Developing an Angular app with multiple services

{{TOC}}

When building an Angular application that consumes multiple services, the initial start can be very tricky. Especially when every service is in early development, but other part of the application need to consume them. The service will change often and development will come with some pain.  
In my actual project we have build some backend services that will live on a Kubernetes cluster at the end. So right now when we make our first steps in the frontend we need some of the services to read from and write to. When having just two or three services around it might be easy to start them from an IDE and than do a simple `ng serve` to get everything up and running.  
But there are some problems with this solution. To get everything working you will need to make some changes in the service configuration. For example, the Angular app will need to make request with CORS allowed on the backend. But these configuration are not used when running in production or staging. The next thing is that every application might need a different tooling and we need to get all that tooling up and running. With some training on the command line this might become easier, but at the end we might not even have the appropriate tooling installed. So there is a lot of extra work to get some else code up and running. So we need to find a solution that wouldn't need much change between development, staging and production.  

## What we need

We needed a solution that is simple and would work nearly out of the box after some initial configuration is done. Every developer should be able to just pull the latest changes and than run the project, with a simple one line command.  
We needed support for existing services which are deployed to the local container repo and services that are still in development.  
Our Angular frontend should be run with `ng serve` and automatically connected to all other services.

## Solution

So here is what I think might work. A reverse proxy, in my case nginx, will be the main entrypoint to the application. All services that are reachable on the local container repo will be pulled via docker-compose. In the nginx configuration all services are registered by name and the reverse proxy will link to them. The configuration will be part of the Angular repo so that it can be shared will all other developers easily.

## Angular

For the demo purpose I just set up a small app, that will reach out some service endpoints.

```
# ng new SampleProject
# npm install
```

This will create the main Angular application for us. Now we will add a new component and a service to our app.

```
# ng g c user
# ng g s user
```

Next we add a call to the user backend service that will request some additional information of a user and calculate a discount for a specific user. Each of these request will be handled by a different backend service.

```
// user.service.ts
getUserInfo(id: number): Observable<UserInfo> {
return this.httpClient
	.get<UserInfo>(`${this.baseUrl}/userinfo/${id}`, {
    observe: 'body'
  })
  .catch(this.handleError);
}
```

The result of that call will be of type `User`, so let's create that next.

```
// user.ts
interface User {
	id: number;
	age: number;
	firstName: string;
	lastName: string;
	discount: number;
}
```

Next will will add a service that will make a call to the discount backend service.

```
# ng g s discount
```

Inside that service we add the code to get the information from the discount backend service.

```
// discount.service.ts
getDiscountForUser(userId: number): Observable<Discount> {
return this.httpClient
	.get<Discount>(`${this.baseUrl}/discount/${id}`, {
    observe: 'body'
  })
  .catch(this.handleError);
}
```

And again we add an interface for the object we want to receive from that call.

```
// discount.ts
interface Discount {
	id: int;
	percentage: number;
}
```

Up to this point this code won't work, because we haven't set up the base url for both services. To prevent some cors configuration, which is needed when our endpoints would be mapped to different domains, we will use a reverse proxy. This will inject the correct route to the endpoints. So in our JS code we will add an endpoint in each of our Angular services that is located on the same server.

```
// user.service.ts
private baseUrl = '/backend/userservice';
```

```
// discount.service.ts
private baseUrl = '/backend/discountservice';
```

To use both of the services we will inject them into our user component and run the requests. The services will get some static demo data and we print the result on the webpage.

```
user: User[];

constructor(private userService: UserService, private discountService: DiscountService) {
}

ngOnInit() {
	this.userService.getUser().subscribe(
      fetchedUsers => {
        this.users = fetchedUsers;
        fetchedUsers.forEach(user => this.discountService.getDiscountForUser(user.id).subscribe(
          discount => user.discount = discount
        ));
      },
      err => console.log(err)
    );
}
```

Inside the html we just put the result on the site.

```
<table class="table">
  <thead>
  <tr>
    <th scope="col">#</th>
    <th scope="col">First Name</th>
    <th scope="col">Last Name</th>
    <th scope="col">Age</th>
    <th scope="col">Discount</th>
  </tr>
  </thead>
  <tbody>
  <tr *ngFor="let user of users; index as i">
    <th scope="row">{{i}}</th>
    <td>{{user.firstName}}</td>
    <td>{{user.lastName}}</td>
    <td>{{user.age}}</td>
    <td>{{user.discount}}</td>
  </tr>
  </tbody>
</table>
```

## Backend Services

For simplicity we will just have two backend services that our Angular app will use. One that is a Kotlin Spring based app and one that is a .Net core based project. The Kotlin Spring backend service will send some static user information and the .net core backend service will do some discount calculation. Nothing fancy, because I want to focus on the _how_ not the _what_.

### Kotlin user service

The reason why I choose Kotlin is, that this language is one of the rising starts right now and I believe we will see a lot more apps coming from there. To get starting with the app we head over to the [Spring Initializer](https://start.spring.io/) website and create a new simple project. We will use Gradle, Kotlin, the latest Spring Boot version and add the Web package. Click _Generate Project_ and this will download a zip file with our template.

#### Endpoint

The first thing we will need is an endpoint for our user service. So let's create that. We will use a typical structure for our app with a controller and a service. I skip the repository, because we are not going to use any database here, just hard coded data.

In all source code example I skip the imports and package/namespaces. Check out the complete solution from my github account.

Just a quick side node. Check the lean syntax of Kotlin! Even as a .NET Developer, this is looking super good!

#### Controller

Our controller will just return an array of users when the root is hit by an get request.

```
@RestController
class UserController(val userService: UserService) {
    @GetMapping(path = ["/"])
    fun getUsers(): ResponseEntity<Array<User>> {
        return userService.getUsers()
    }
}
```

#### User

Thanks to Kotlin data class power we create this awesome lean code block.

```
data class User(
    val firstName: String,
    val lastName: String,
    val age: Int) {
}
```

#### Service

The service will create the http result object with the body that contains our array of User.

```
@Service
class UserService {
    fun getUsers(): ResponseEntity<Array<User>> {
        return ResponseEntity.ok(arrayOf(
                User("Tim", "Klug", 34),
                User("Torben", "Holsten", 35),
                User("Jolanta", "Frederiksen", 33)
        ))
    }
}
```

#### Configuration

The configuration will only contain the application port, which we set to `:80` to serve without any extra port portion in the url.

```
# application.yml
server:
	port: 80
```


#### Dockerfile

Nothing to comment here. This should all be clear.

```
FROM openjdk:8u151-jre-alpine3.7

MAINTAINER 16851587+tim-klug@users.noreply.github.com

COPY /build/libs/userservice-*.jar /src/application.jar

EXPOSE 80

WORKDIR /src

CMD java -jar application.jar
```

#### Creating the container

So that's it for our UserService. We build it and create an image from it. This service will be treated as a almost running service in our development.

For our container we will need a jar package, so let's compile that with gradle.

```
# ./gradlew clean bootJar
```

Next we build our image from the Dockerfile.

```
# docker build -t tim-klug/userservice:latest .
```	

We can spin up a container from that images, to check if everything worked out well.

```
# docker run --rm -d -p 8080:8080 tim-klug/userservice:latest
```

You can open the endpoint `http//localhost:8080/` in your browser or shoot a Postman request at it. The result will be the same. We should get our three users.

```
[
    {
        "firstName": "Tim",
        "lastName": "Klug",
        "age": 34
    },
    {
        "firstName": "Torben",
        "lastName": "Holsten",
        "age": 35
    },
    {
        "firstName": "Jolanta",
        "lastName": "Frederiksen",
        "age": 33
    }
]
```

Ok, now we have our UserService so let's create the service for calculating discounts.

### the CalculationService

For this service I will use C# and build it in Rider. There is no special need to do so. You can choose what ever IDE fits for you.

I create a new `.net core 2` project with the WebApi template. The same can be done in VS or with Yeoman. The template has already a controller template in it. I will recycle that and rename it.

#### the DiscountController

```
// DiscountController.cs
[Route("api/[controller]")]
    public class DiscountController : Controller
    {
        [HttpGet("{id}")]
        public int Get(int id)
        {
            return 23;
        }
    }
```

This controller will return a static percentage for any user id. As said the business logic doesn't matter right now.

So that is all we need to do here. Since we will run this service in debug during development, we will no put it into a container.

Next we will create a simple Angular app for our frontend.

### Reverse Proxy

To put all those parts togehter we will make use of an reverse proxy. In this case I hava chosen Nginx. To enable our
services to get some data routed, we need to create a configuration file that will make all the traffic forward passing
for us.

The first thing we need to do is to configure nginx so that it will serve a website.

```
server {
  listen 80;
  charset utf-8;
  sendfile on;
  root /usr/share/nginx/html;
}
```

The next thing is make nginx work as a revese proxy for our two services. For each service an URI path must be assigned
that will be forwarded to the service. These information must be placed inside the servce node.

```
location / {
  proxy_pass http://host.docker.internal:4200/;
  proxy_http_version 1.1;
  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
}
location /backend/userservice {
  proxy_pass http://usermanager/;
  proxy_http_version 1.1;
  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
}

location /backend/discountservice/ {
  proxy_pass http://host.docker.internal:5000/;
  proxy_redirect     off;
  proxy_set_header   Host $host;
  proxy_set_header   X-Real-IP $remote_addr;
  proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header   X-Forwarded-Host $server_name;}
```

Notice, that the root and the discountservice are refferenced to `http://host.docker.internal`. This will forward the
traffic to the local environment and enable us to debug directly this service. I like to place this file directly in
frontend project folder.

### Docker Compose

The last peace is the Docker Compose file that will tight all the services together and let them interact with eachother. The docker-compose.yml will also be placed directly in the root of the fronted project. This file will contain the configuration of the nginx and the service we want to serve by a container. The combination of which service will be in a container and which are started from an IDE depends on the actual needs.  
I a szenario where services should be able to talk to each other, make shure to give the services a propper naming inside the docker compose file. The names used there should match the names the servies use to reach out for each other.

```
version: '3.6'

services:
 proxy:
   image: nginx
   volumes:
     - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
   ports:
   - "8888:80"
   depends_on:
     - userservice
 userservice:
   image: tim-klug/userservice:latest
   restart: always
```

The targets from the nginx configuration must be avaulable when starting nginx. Therefore the nginx depends on the the service that is running inside a container. To make make nginx use the configuration, it is linked as a volumn to the container.

So the last thing to do is starting the .net app with `dotnet run` form the root of the calculation service or from the ide of choice. But make shure that the correct port is used. Than run the `docker-compose up` inside the frontend
project. If everything is configured right, log messages should run by during start up. For the .net app to see verbose logging, make shure to enable it. Otherwise nothing will show up. The last thing is to start the Angular with `ng serve` and everything shoud be up and running. Open the briwser on port 8888 (not 4200) to access the app from the reverse proxy.

## Pitfalls

One thing that took some time was the to configure the nginx reverse proxy to forward to the correct endpoint. Check the trailing /, it might be removed during forwarding dependung on the enpoint that is provided. The `proxy_pass` must have a trailing / so that all uri parts are forwarded to the endpoint. Otherise you will end up on the sites root level,  without any parameters in the reuest. As always the behavior depends on the business needs.
