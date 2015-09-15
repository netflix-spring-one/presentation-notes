# Why does the audience care? (Jon)

* Make a Spring Boot app *cloud-ready* even if you are not yet *cloud-native*
* Spring Cloud is a misnomer.  Everything we will show you can be stood up in a private DC.
* TODO get McGarr's slide on Cloud as a **utility**
* Spring Cloud Netflix makes it easy to start up relatively complex pieces with little configuration

---

# Simple Spring Boot App (Jon)

* Slide [Collection<SpringBootApplication>]
* Orient audience to the code
* Start Eureka from `./gradlew bootRun > out.log`
* Start Membership from `./gradlew bootRun > out.log`
* Start Recommendations from IDE
* Execute `curl http://localhost:8081/api/recommendations/jschneider`
* Execute `curl http://localhost:8081/api/recommendations/twicksell`
* Execute `curl http://localhost:8081/api/recommendations/unknown`

* Recommendations contains a hard-coded link to a DNS name for membership
* With hard-coded service links, multiple instances of a service need to run behind a load balancer (e.g. ELB) and
the load balancer needs to have a route registered with it.
* Some production ready features

---

# Add Eureka (Jon)

* Add `compile 'org.springframework.cloud:spring-cloud-starter-eureka'`
* Add `eureka.client.serviceUrl.defaultZone: http://localhost:9000/eureka/` to application.yml (TRAILING SLASH IS IMPORTANT!)
* Add `eureka.instance.hostname: theone`
* Add `@EnableEurekaClient`
* Remove RestTemplate bean
* Restart Recommendations
* Show both instances are up in Eureka server at http://localhost:9000

* Mention that Eureka is fault tolerant.  We deploy at least one per AZ per region.

---

# Respond to Application Event when No Longer In Discovery (Taylor)

* Demo executing REST request to drop an instance from discovery
* Add to Recommendations:
```
@EventListener
    public void onEurekaStatusDown(EurekaStatusChangedEvent event) {
        if(event.getStatus() == InstanceInfo.InstanceStatus.DOWN || event.getStatus() == InstanceInfo.InstanceStatus.OUT_OF_SERVICE) {
            System.out.println("Stop listening to queues and such...");
        }
    }
```
* Run `curl -X PUT http://localhost:9000/eureka/apps/recommendations/theone/status?value=OUT_OF_SERVICE`
* [Eureka REST Operations](https://github.com/Netflix/eureka/wiki/Eureka-REST-operations)

## RESULT
* Stop listening to queues, etc.

---

# Ribbon RestTemplate (Taylor)

* Add `membership.ribbon.DeploymentContextBasedVipAddresses: membership` to application.yml
* Replace Hard-coded Link with VIP addresses
* No longer tied to load balancer.  No need to register route with load balancer.
* Smarter load distribution?  e.g.
    - zone avoidance rule,
    - availability filter rule (filters those in circuit breaker trip state)
    - keep track of response times and give preferential treatment to those instances that respond the fastest
* Discuss benefit of stacks -- segmenting traffic by responsibility
    - set eureka.instance.virtualHostName (default is application name), could set Membership to membership-batch or membership-api

---

# Feign Client (Jon)

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

# Hystrix (Jon)

* Add `compile 'org.springframework.cloud:spring-cloud-starter-hystrix'`
* Add `@EnableHystrix`
* Add `Set<Movie> familyRecommendations = Sets.newHashSet(new Movie("hook"), new Movie("the sandlot"));`
* Add `@HystrixCommand(fallbackMethod = "recommendationFallback")` METHOD MUST BE PUBLIC!!
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
* Add: `RequestContextHolder.currentRequestAttributes()` -- relies on ThreadLocal and will fail
* Start Recommendations and demonstrate how `currentRequestAttributes()` fails on ThreadLocal
* Add: `@HystrixProperty(name = "execution.isolation.strategy", value = "SEMAPHORE"),` -- @Transactional now works, ThreadLocal now works

---

# Hystrix dashboard (Jon)

* Start HystrixDashboard
* In browser, navigate to http://localhost:9001/hystrix
* In text box, enter http://localhost:8001/hystrix.stream
* Execute JMeter run against Recommendations

---

# Turbine (Jon)

* Start Turbine from IDE
* In hystrix dashboard, enter: http://localhost:9002/turbine.stream?cluster=SAMPLE

---

# Spectator Metrics (Taylor)

* In browser, show current actuator metrics: http://localhost:8001/metrics
* Tagged vs. Hierarchical structure.  Canonical example: how to get latency of all HTTP 200s in a hierarchical structure
consisting of `{uri}.200`.
* Add dependencies:
```
compile 'com.netflix.spectator:spectator-api:0.30.0'
compile 'com.netflix.spectator:spectator-reg-servo:0.30.0'
compile 'com.netflix.spectator:spectator-ext-sandbox:0.30.0'
```
* Show /metrics endpoint after executing a series of REST requests.

* What should not be a tag?  Example: customerId because of the combinatorial explosion of tags.
* Add a bad tag and see what happens in /metrics

* Walkthrough of underlying code

* Tell the superbowl thundering herd anecdote while we wait for spectator metrics to appear

---

# Atlas (Taylor)

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

# What about Zuul (Jon)

* Our premise has been that client-side load balancing is an architectural alternative to the typical load-balancer-per-server model.  Ribbon+Eureka folds load balancing into the service call itself.
* So why Zuul, a router distinct from the operation of any particular service?
