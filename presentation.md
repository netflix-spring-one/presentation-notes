# Why does the audience care? (Jon)

* Make a Spring Boot app *cloud-ready* even if you are not yet *cloud-native*
* Spring Cloud is a misnomer.  Everything we will show you can be stood up in a private DC.
* TODO get McGarr's slide on Cloud as a **utility**
* Spring Cloud Netflix makes it easy to start up relatively complex pieces with little configuration

---

# Simple Spring Boot App (Taylor)

* (Should be on workshop branch, git checkout tags/start)
* Slide [Collection<SpringBootApplication>]
* Orient audience to the code
* Start Eureka from `./gradlew bootRun > out.log`
* Start Membership from `./gradlew bootRun > out.log`
* Start Recommendations from IDE
* Execute `curl http://localhost:8001/api/recommendations/jschneider`
* Execute `curl http://localhost:8001/api/recommendations/twicksell`
* Execute `curl http://localhost:8001/api/recommendations/unknown`

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
* Add `@Primary` on RestTemplate bean - we don't want Ribbon just yet

* Mention that Eureka is fault tolerant.  We deploy at least one per AZ per region.

# Respond to Application Event when No Longer In Discovery (Taylor)

* (git checkout tags/eureka)
* Add dependency: `compile 'com.netflix.spring:spring-cloud-netflix-contrib:0.3.0'`
* Add to Recommendations:
```
  @EventListener
  public void onEurekaStatusDown(EurekaStatusChangedEvent event) {
      if(event.getStatus() == InstanceInfo.InstanceStatus.DOWN || event.getStatus() == InstanceInfo.InstanceStatus.OUT_OF_SERVICE) {
          System.out.println("Stop listening to queues and such...");
      }
  }
```
* Run `curl -X PUT http://localhost:9000/eureka/apps/recommendations/theone/status?value=OUT_OF_SERVICE` (can take about 15 secs)
* [Eureka REST Operations](https://github.com/Netflix/eureka/wiki/Eureka-REST-operations)

---

# Ribbon RestTemplate (Jon)

* (git checkout tags/eureka-listener)
* Add `members.ribbon.DeploymentContextBasedVipAddresses: membership` to application.yml
* Remove RestTemplate bean
* Replace Hard-coded Link with Ribbon client address
* Restart Recommendations
* Show both instances are up in Eureka server at http://localhost:9000

* No longer tied to load balancer.  No need to register route with load balancer.
* Smarter load distribution?  e.g.
    - zone avoidance rule,
    - availability filter rule (filters those in circuit breaker trip state)
    - keep track of response times and give preferential treatment to those instances that respond the fastest
* Discuss benefit of stacks -- segmenting traffic by responsibility
    - set eureka.instance.virtualHostName (default is application name), could set Membership to membership-batch or membership-api
* All Ribbon get requests will be automatically retried if the request to the first node fails, PUTS/POSTS can be configured to do that
    - Max number of retries on the same server (excluding the first try) `sample-client.ribbon.MaxAutoRetries=1`
    - Max number of next servers to retry (excluding the first server) `sample-client.ribbon.MaxAutoRetriesNextServer=1`
    - Whether all operations can be retried for this client `sample-client.ribbon.OkToRetryOnAllOperations=true`

---

# Feign Client (Jon)

* (git checkout tags/ribbon)
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
* Remove Autowired RestTemplate from RecommendationsController
* Add `@Autowired MembershipRepository membershipRepository`
* Change RestTemplate call to `membershipRepository.findMember(user)`

* Not used internally - talk to Adrian Cole?

---

# Hystrix (Jon)

* Show Hystrix slide (Frank Underwood)
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
* Mention that SEMAPHORE strategy will cause timeoutInMilliseconds to NOT work! Hanging threads will remain hung but circuit breaker will be tripped.

---

# Hystrix dashboard (Jon)

* Start HystrixDashboard
* In browser, navigate to http://localhost:9001/hystrix
* In text box, enter http://localhost:8001/hystrix.stream
* Execute JMeter run against Recommendations

---

# Turbine (Jon)

* (git checkout tags/hystrix)
* Add `eureka.instance.appGroupName: sample` to application.yml
* Remove `eureka.instance.hostname: theone` -- because Turbine is going to try to connect to this as a resolvable host
* Start Turbine from IDE
* In hystrix dashboard, enter: http://localhost:9002/turbine.stream?cluster=SAMPLE

---

# Spectator Metrics (Taylor)

* (git checkout tags/turbine)
* Add `@Autowired RestTemplate restTemplate;` back to RecommendationsController
* Change membership call to `restTemplate.getForObject("http://members/api/member/{user}", Member.class, user);`
* In browser, show current actuator metrics: http://localhost:8001/metrics
* Tagged vs. Hierarchical structure.  Canonical example: how to get latency of all HTTP 200s in a hierarchical structure
consisting of `{uri}.200`.
* Add dependencies:
```
compile 'org.springframework.boot:spring-boot-starter-aop'
compile 'com.netflix.spectator:spectator-api:0.30.0'
compile 'com.netflix.spectator:spectator-reg-servo:0.30.0'
compile 'com.netflix.spectator:spectator-ext-sandbox:0.30.0'
```
* Add `autoconfigure.exclude: org.springframework.cloud.netflix.servo.ServoMetricsAutoConfiguration`
* Execute JMeter script
* Show /metrics endpoint

* What should not be a tag?  Example: customerId because of the combinatorial explosion of tags.
* Add a bad tag and see what happens in /metrics

* Walkthrough of underlying code

* Tell the superbowl thundering herd anecdote while we wait for spectator metrics to appear

---

# Atlas (Taylor)

* Start atlas with ./startAtlas.sh
* Add dependency `compile 'com.netflix.servo:servo-atlas:0.11.2'`
* Restart Membership
* Execute JMeter script
* List tags http://localhost:7101/api/v1/tags
* Retrieve a single png
```
http://localhost:7101/api/v1/graph?q=name,rest,:eq,:avg
http://localhost:7101/api/v1/graph?q=name,rest,:eq,statistic,count,:eq,:and,:avg
http://localhost:7101/api/v1/graph?q=name,rest,:eq,statistic,count,:eq,:and,:avg,(,bucket,),:by
http://localhost:7101/api/v1/graph?q=name,rest,:eq,statistic,count,:eq,:and,:avg,(,bucket,),:by,:pct,:stack

http://localhost:7101/api/v1/graph?q=metric,Queue.*,:re,:sum,(,name,appName,),:by
```
* Demonstrate really basic stack language
* Demonstrate dashboard generation
* Requirements for standing up a production-ready Atlas

---

# What about Zuul (Jon)

* Our premise has been that client-side load balancing is an architectural alternative to the typical load-balancer-per-server model.  Ribbon+Eureka folds load balancing into the service call itself.
* So why Zuul, a router distinct from the operation of any particular service?
