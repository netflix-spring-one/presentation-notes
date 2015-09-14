# Why does the audience care?

* Make a Spring Boot app *cloud-ready* even if you are not yet *cloud-native*
* Spring Cloud is a misnomer.  Everything we will show you can be stood up in a private DC.
* TODO get McGarr's slide on Cloud as a **utility**
* Spring Cloud Netflix makes it easy to start up relatively complex pieces with little configuration

---

# Simple Spring Boot App

* Slide [Collection<SpringBootApplication>]
* Orient audience to the code
* Start Membership from IDE
* Start Recommendations from IDE
* Execute `curl http://localhost:8081/api/recommendations/jschneider`
* Execute `curl http://localhost:8081/api/recommendations/twicksell`
* Execute `curl http://localhost:8081/api/recommendations/unknown`

* Recommendations contains a hard-coded link to a DNS name for membership
* With hard-coded service links, multiple instances of a service need to run behind a load balancer (e.g. ELB) and
the load balancer needs to have a route registered with it.
* Some production ready features

---

# Add Eureka

* Start Eureka with eureka module main method (OBSOLETE)
* Start Eureka with `docker run -d -p 8671:8671 --name eureka netflixspring/sample-eureka` so that other docker containers can link to it

* Add `compile 'org.springframework.cloud:spring-cloud-starter-eureka'`
* Add `eureka.client.serviceUrl.defaultZone: http://localhost:9000/eureka/` to application.yml (TRAILING SLASH IS IMPORTANT!)
* Add `@EnableEurekaClient`
* Remove RestTemplate bean
* Start Recommendations
* Show both instances are up in Eureka server at http://localhost:9000

* Mention that Eureka is fault tolerant.  We deploy at least one per AZ per region

---

# Relaunch Membership With Eureka with Docker

* docker run -d -p 9000:9000 --link eureka --name membership netflixspring/sample-membership

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
* Add: `commandProperties={@HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "5000")}`
* Add: `String mapping = (String) RequestContextHolder.currentRequestAttributes().getAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE, RequestAttributes.SCOPE_REQUEST);`
* Add: `@HystrixProperty(name = "execution.isolation.strategy", value = "SEMAPHORE"),`

* TODO what exactly does it do to thread local and @Transactional?

---

# Hystrix dashboard

* Start HystrixDashboard
* Start Recommendations with newly added Hystrix command
* In browser, navigate to http://localhost:9001/hystrix
* In text box, enter http://localhost:8001/hystrix.stream
* Execute JMeter run against Recommendations

---

# Turbine

* Start Turbine from IDE
* In hystrix dashboard, enter: http://localhost:9002/turbine.stream

---

# Spectator Metrics

* Tagged vs. Hierarchical structure.  Canonical example: how to get latency of all HTTP 200s in a hierarchical structure
consisting of `{uri}.200`.  Would have to first itemize all such `{uri}`.
* Add `@EnableSpectator`
* Show /metrics endpoint after executing a series of REST requests.

* What should not be a tag?  Example: customerId because of the combinatorial explosion of tags.
* Add a bad tag and see what happens in /metrics

* Walkthrough of underlying code

* Tell the superbowl thundering herd anecdote while we wait for spectator metrics to appear

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
