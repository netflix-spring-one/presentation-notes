# Why does the audience care?

* Make a Spring Boot app *cloud-ready* even if you are not yet *cloud-native*
* Spring Cloud is a misnomer.  Everything we will show you can be stood up in a private DC.
* TODO get McGarr's slide on Cloud as a **utility**
* Spring Cloud Netflix makes it easy to start up relatively complex pieces with little configuration

---

# Simple Spring Boot App

* Start Membership from IDE

* Recommendations contains a hard-coded link to a DNS name for membership
* With hard-coded service links, multiple instances of a service need to run behind a load balancer (e.g. ELB) and
the load balancer needs to have a route registered with it.
* Some production ready features

---

# Add Eureka

* Start Eureka with eureka module main method (OBSOLETE)
* Start Eureka with `docker run -d -p 8671:8671 --name eureka netflixspring/sample-eureka` so that other docker containers can link to it

* Add `compile 'org.springframework.cloud:spring-cloud-starter-eureka'`
* Add `eureka.client.serviceUrl.defaultZone: http://localhost:8761/eureka/` to application.yml
* Add `@EnableEurekaClient`
* Remove RestTemplate bean
* Start Recommendations
* Execute `curl -x PUT http://localhost:8671/eureka/recommendations/{instanceId}/` to take instance out of service ... TODO fix this
* [Eureka REST Operations](https://github.com/Netflix/eureka/wiki/Eureka-REST-operations)

---

# Relaunch Membership With Eureka with Docker

* docker run -d -p 8080:8080 --link eureka --name membership netflixspring/sample-membership

---

# Respond to Application Event when No Longer In Discovery

* Demo executing REST request to drop an instance from discovery
* [Eureka REST Operations](https://github.com/Netflix/eureka/wiki/Eureka-REST-operations)

## RESULT
* Stop listening to queues, etc.

---

# Ribbon RestTemplate

* Replace Hard-coded Link with VIP addresses
* No longer tied to load balancer.  No need to register route with load balancer.
* Smarter load distribution?  TODO how?
* Discuss benefit of stacks -- segmenting traffic by responsibility

---

# Feign Client

* Somewhat simplifies the repository
* Add `compile 'org.springframework.cloud:spring-cloud-starter-feign'`
* Add `@EnableFeignClients`
* Add:
```java
@FeignClient("membership")
interface MembershipRepository {
    @RequestMapping(method = RequestMethod.GET, value = "/api/member/{user}")
    Member findMember(@PathVariable("user") String user);
}
```

---

# Hystrix

* Add `compile 'org.springframework.cloud:spring-cloud-starter-hystrix'`
* Add `@EnableHystrix`
* Add `Set<Movie> familyRecommendations = Sets.newHashSet(new Movie("hook"), new Movie("the sandlot"));`
* Add `@HystrixCommand(fallbackMethod = "recommendationFallback")` METHOD MUST BE PUBLIC!!
* Add `@EnableHystrixDashboard`
* Add:
```java
/**
 * Should be safe for all audiences
 */
Set<Movie> recommendationFallback(String user) {
    return familyRecommendations;
}
```
* Add: `commandProperties={@HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "100")}`
* Add: `String mapping = (String) RequestContextHolder.currentRequestAttributes().getAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE, RequestAttributes.SCOPE_REQUEST);`
* Add: `@HystrixProperty(name = "execution.isolation.strategy", value = "SEMAPHORE"),`

* TODO what exactly does it do to thread local and @Transactional?

---

# Turbine

* Start Turbine
* Remove `@EnableHystrixDashboard`
* Wire hystrix commands to Turbine
* Demo centralized circuit breaker in Turbine

---

# Spectator Metrics

* Tagged vs. Hierarchical structure.  Canonical example: how to get latency of all HTTP 200s in a hierarchical structure
consisting of `{uri}.200`.  Would have to first itemize all such `{uri}`.
* Add `@EnableSpectator`
* Show /metrics endpoint after executing a series of REST requests.

* What should not be a tag?  Example: customerId because of the combinatorial explosion of tags.
* Add a bad tag and see what happens in /metrics

* Walkthrough of underlying code

---

# Atlas

* Start atlas with ./startAtlas.sh
* Add `@EnableAtlas`
* Restart Membership
* Execute JMeter script
* List tags
* Retrieve a single png
* Demonstrate really basic stack language
* Demonstrate dashboard generation
* Requirements for standing up a production-ready Atlas

---

# Spectator Graphite via servo-graphite

* What does it do to tags?

---

# What about Zuul

* Our premise has been that client-side load balancing is an architectural alternative to the typical load-balancer-per-server model.  Ribbon+Eureka folds load balancing into the service call itself.
* So why Zuul, a router distinct from the operation of any particular service?
