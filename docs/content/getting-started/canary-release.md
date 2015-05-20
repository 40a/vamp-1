---
title: 2. Doing a canary release
type: documentation
weight: 30
menu:
    main:
      parent: getting-started
    
---
    
# 2. Doing a canary release

In the [previous part](/documentation/getting-started/deploying/) of our tutorial we deployed our app sava 1.0. If you haven't
walked through that part, please do so first. Now let's say we have a new version of this great application that we want to canary release into production. We have it containerised as `magneticio/sava:1.1.0` and ready to go.

## Step 1: Prepping our blueprint

Vamp allows you to do canary releases using blueprints. Take a look at the YAML example below. It is quite similar to the blueprint we initially used to deploy sava 1.0.0. However, there are two big differences.

1. The `services` key holds a list of breeds: one for v1.0.0 and one for v1.1.0 of our app. [Breeds](/documentation/reference/breeds/) are Vamp's way of describing static artifacts that can be used in blueprints.
2. We've added the `routing` key which holds the weight of each service as a percentage of all requests. 

Notice we assigned 50% to our current version 1.0.0 and 50% to the new version 1.1.0 We could also start with a 100% to 0% split, a 99% to 1% split or whatever combination you want as long as all percentages add up to 100% in total.
{{% copyable %}}

```yaml
---
name: sava:1.0

endpoints:
  sava.port: 9050

clusters:

  sava:
    services: # services is now a list of breeds
      -
        breed:
          name: sava:1.0.0
          deployable: magneticio/sava:1.0.0
          ports:
            port: 80/http
        scale:
          cpu: 0.5       
          memory: 512  
          instances: 1          
        routing: 
          weight: 50  # weight in percentage           
      -
        breed:
          name: sava:1.1.0 # a new version of our service
          deployable: magneticio/sava:1.1.0
          ports:
            port: 80/http
        scale:
          cpu: 0.5       
          memory: 512  
          instances: 1  
        routing: 
          weight: 50            
```{{% /copyable %}}

{{% alert info %}}
**Note**: There is nothing stopping you from deploying three or more versions and distributing the weight
among them. Just make sure that when doing a straight threeway split you give one service 34% as 33+33+34=100.
{{% /alert %}}


## Step 2: Deploying the new version next to the old one

It is our goal to update the already running deployment with the new blueprint. Vamp will figure out that v1.0.0
is already there and just add v1.1.0 while setting the correct routing between these services.

We update a running deployment getting its name (the UUID) from `/api/v1/deployments` and `PUT`-ing the blueprint to that resource, e.g: `/api/v1/deployments/e1c99ca3-dc1f-4577-aa1b-27f37dba0325`

Vamp should respond with a `202 Accepted` and start executing your command. When finished deploying, you can
start refreshing your browser at the correct endpoint, e.g. `http://10.26.184.254:9050/`.  

![](/img/screenshots/monolith_canary1.png)
The application should switch between responding with a 1.0 page and a 1.1 page. This actually works best in the "Incognito" or "Anonymous" mode of your browser because of html/css/js cache busting issues.

## Step 3: Using filters to target specific groups

Using percentages to divide traffic between versions is already quite powerful, but also very simplistic.
What if, for instance, you want to specifically target a group of users? Or a specific channel of requests
from an internal service? Vamp allows you to do this right from the blueprint DSL. 

Let's start simple: We will allow only Chrome users to access v1.1.0 of our application by inserting this routing scheme:

```yaml
---
routing:
  weight: 0
  filters:
    - condition: User-Agent = Chrome
```

Notice two things:

1. We dialed back the weight to 0%. This is important and might feel counter intuitive, but Vamp first
checks filters and then weight. This means we explicitly do not send 'just some percentage of traffic' to this service but only traffic that matches the filter.
2. We inserted a list of conditions (with only one condition for now)

Our full blueprint now looks as follows:  

{{% copyable %}}

```yaml
---
name: sava:1.0

endpoints:
  sava.port: 9050

clusters:

  sava:
    services: # services is now a list of breeds
      -
        breed:
          name: sava:1.0.0
          deployable: magneticio/sava:1.0.0
          ports:
            port: 80/http
        scale:
          cpu: 0.5       
          memory: 512  
          instances: 1              
        routing: 
          weight: 100
      -    
        breed:
          name: sava:1.1.0
          deployable: magneticio/sava:1.1.0
          ports:
            port: 80/http
        scale:
          cpu: 0.5       
          memory: 512  
          instances: 1              
        routing: 
          weight: 0
          filters:
            - condition: User-Agent = Chrome                   
```
{{% /copyable %}}

Again, use a `PUT` request to the right deployment. As we are not actually deploying anything but just reconfiguring routes, the update should be almost instantaneous. You can fire up a Chrome browser and
a Safari browser and check the results. A hard refresh might be necessary because of your browser's 
caching routine.

![](/img/screenshots/screencap_canary1.gif)

## Step 4: Learning a bit more about filters.

Our browser example is easily testable on a laptop, but of course a bit contrived. Luckily you can 
create much more powerful filters quite easily. Checking Headers, Cookies, Hosts etc. is all possible.
Under the hood, Vamp uses [Haproxy's ACL's](http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#7.1) and you can use the exact ACL definition right in the blueprint in the `condition` field of a filter.

However, ACL's can be somewhat opaque and cryptic. That's why Vamp has a set of convenient "short codes"
to address common use cases. Currently, we support the following, but we will be expanding on this in the future:

```
User-Agent = *string*
Host = *string*
Cookie *cookie name* Contains *string*
Has Cookie *cookie name*
Misses Cookie *cookie name*
Header *header name* Contains *string*
Has Header *header name*
Misses Header *header name*
```

Vamp is also quite flexible when it comes to the exact syntax. This means the following are all equivalent:

```
hdr_sub(user-agent) Android   # straight ACL
user-agent=Android            # lower case, no white space
User-Agent=Android            # upper case, no white space
user-agent = Android          # lower case, white space
```

Having multiple conditions in a filter is perfectly possible. In this case all filters are implicitly
"OR"-ed together, as in "if the first filter doesn't match, proceed to the next". For example, the following filter would first check whether the string "Chrome" exists in the User-Agent header of a
request. If that doesn't result in a match, it would check whether the request has the header 
"X-VAMP-TUTORIAL". So any request matching either condition would go to this service.

```yaml
---
routing:
  weight: 0
  filters:
    - condition: User-Agent = Chrome
    - condition: Has Header X-VAMP-TUTORIAL
```

Using a tool like [httpie](https://github.com/jakubroztocil/httpie) makes testing this a breeze.

    http GET http://10.26.184.254:9050/ X-VAMP-TUTORIAL:stuff

![](/img/screenshots/screencap_canary2.gif)    

Cool stuff. But we are dealing here with single, monolithic applications. Where are the microservices? We will be chopping up this monolith into services and deploy them with Vamp in [the third part of our tutorial →](/documentation/getting-started/splitting-services/)
