# Refactoring a Monolithic Data Model to Microservice Architectures in MongoDB

As the lead architect at [Hybrid Web Agency](https://hybridwebagency.com/), one of the most demanding projects I've ever undertaken was the transition of an outdated monolithic ERP system into a cutting-edge microservices architecture. The aging monolith had become a burden to maintain, hindering my client's ability to innovate effectively.

Fortunately, my extensive experience, acquired through years of providing customized [Software Development Services in Arlington, TX](https://hybridwebagency.com/arlington-tx/best-software-development-company/), allowed me to approach this challenge with confidence. I understand the obstacles organizations face when dealing with rigid and hard-to-adapt systems. This article outlines the systematic approach we took to restructure the data model and decompose the monolithic system at both the code and database levels.

Throughout this migration process, my team and I unearthed lesser-known best practices for designing domain entities within a document database like MongoDB. These practices are essential for supporting fully independent microservices. If you've ever wondered how to prepare your database architecture for cloud readiness while preserving historical data, this article will provide valuable insights.

By the end of this article, you'll have a clear roadmap for migrating your legacy systems to a modern architecture. I'll share practical insights to help you avoid common pitfalls and accelerate the delivery of new features to your customers. Let's dive in!

## Advantages of a Microservices Architecture

Transitioning to a microservices architecture offers numerous benefits over a monolithic approach. It allows for independent service deployment, enabling rapid feature development and release without disrupting the entire application.

Furthermore, different programming languages and frameworks can be used for various services, allowing you to choose the best technologies for each specific domain. For instance, you can leverage Python's machine learning libraries for a recommendation engine, while building a user interface with React.

This separation of concerns empowers specialized teams to work autonomously on individual services. It also enables quick validation of ideas through prototypes, reducing the risk associated with a full monolithic rewrite. New team members can efficiently contribute to services aligned with their expertise.

In a landscape where technologies constantly evolve, microservices mitigate the risk of these shifts. Replacements involve only small, isolated parts of the system, rather than an entire monolithic overhaul. Well-defined interfaces simplify the process of migrating components to new implementations.

#### Independent Scaling

Efficient management of computational resources is achieved by allowing services to scale independently in response to demand. For instance, during holidays, only order processing services may require additional servers to handle increased orders, without affecting the overall system.

```
ordersAPI.scale(replicas: 5);
```

Horizontal scaling at the service level leads to significant cost savings by optimizing resource allocation. Idle support microservices, such as user profiles, don't require overprovisioning to handle traffic that doesn't impact them.

## Analyzing the Data Model
### Understanding Entity Relationships

The initial step in migrating to a microservices architecture is analyzing how entities interact within the existing monolithic data model. We conducted a thorough examination of each collection in the MongoDB database to identify areas where domains naturally clustered and transaction boundaries were established.

Entities like Users, Products, and Orders formed the core of these bounded contexts. The relationships between these fundamental entities served as candidates for service decomposition. For example, we noticed that Orders featured foreign keys linking them to Users, representing the customers, and Products, signifying the purchased items.

To gain a deeper understanding of these cross-dependencies, we visually mapped interconnected fields by examining sample documents. This process revealed that legacy code had merged data relevant to different business capabilities, such as shipping addresses repeating user profiles rather than maintaining lightweight references.

Our analysis of these relationships exposed instances of problematic tight coupling between modules, causing cascading updates. Normalizing redundant data eliminated obstacles to the independent development of user profiles and shipping namespaces.

We employed database tools to explore these connections. Using MongoDB Compass, we diagrammed relationships through $lookup pipelines and ran aggregate queries to count references between entities. This approach helped us identify key breakpoints for separating logic into coherent services.

Understanding these relationships not only delineated domain boundaries but also ensured that services offered well-defined interfaces. These clear contracts made it possible for autonomous teams to develop and deploy modules incrementally, akin to micro Frontends, without hindering each other.

### Identifying Transactional Boundaries

In addition to understanding relationships, we delved into transactions within the existing codebase to comprehend how business processes flowed. This exercise helped us identify areas where data modifications needed to occur within a single service to maintain data consistency and integrity.

For example, within the realm of order processing, we recognized that any updates related to the order itself, payments, inventory levels, and shipment notifications needed to take place within a single service to ensure data consistency. This insight played a pivotal role in defining the boundary of our Order Management service.

Analyzing both relationships and transactions provided crucial insights that guided the restructuring of the data model and logic into independently deployable microservices with well-defined interfaces.

## Refactoring for Microservices

### Standardizing Data Schemas

To support independent services that may deploy to different data stores when necessary, we standardized schemas to eliminate redundancy and include only the essential data required by each service.

For instance, the original Orders schema contained the entire User object. We carried out a significant refactoring to transform it into a lightweight reference:

```python
# Before
Orders: {
  user: {
    name: 'John',
    address: '123 Main St...' 
  }
  #...
}

# After  
Orders: {
  userId: 1234
  #...  
}
```

In a similar vein, we extracted product details from the Orders collection and assigned them to their independent collections, allowing these entities to evolve independently over time.

### Embracing Domain-Driven Design

We harnessed the power of bounded contexts from Domain-Driven Design to logically segregate services, such as Order Fulfillment and User Profiles. These interfaces abstracted data access:

```
interface UserRepository {
  getUser(id): User;
  updateProfile(user: User): void;
}

class MongoUserRepository implements UserRepository {

  users = db.collection('users');

  async getUser(id) {
    return this.users.findOne({_id: id}); 
  }

  //...
}
```

### Evolving Data Access Patterns

To align with the demands of the new architecture, queries and commands underwent substantial refactoring. Previously, services accessed data directly using calls like `db.collection.find()`. We introduced data access libraries to introduce an abstraction layer:

```python
# order.service.ts

constructor(private ordersRepo: OrderRepository) {}

getOrders() {
  return this ordersRepo.find();
}

# mongodb.repo.ts

@Injectable()
class MongoOrderRepository implements OrderRepository {

  constructor(private mongoClient: MongoClient) {}

  find() {
   return this.mongoClient.db.collection('orders').find(); 
  }

}
```

This approach granted us the flexibility to migrate databases without necessitating alterations to consumer code.

## Deploying Microservices

### Independent Scaling

In the realm of microservices, autoscaling is executed at the individual service level rather than on an application-wide basis. We implemented scaling logic through Docker Swarm and Kubernetes.

Deployment manifests specified scaling policies contingent on CPU and memory usage:

```yaml
# docker-compose.yml

services:
  orders:
    image: orders
    deploy:
     

 resources:
        limits:
          cpus: '0.50'
          memory: 512M
      restart_policy: any
      placement:
        constraints: [node.role == worker]  
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      rollback_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      scaler:
        min_replicas: 2    
        max_replicas: 6
```

Scaling tiers delivered reserved capacity and elastic overflow management. The Orders service autonomously monitored itself and spawned or terminated containers as needed:

```python
# orders.js

const cpusUsed = process.cpuUsage();

if(cpusUsed > 80) {
  swarm.scaleService('orders', 1);
}

if(cpusUsed < 20) {
  swarm.scaleService('orders', -1);  
}
```

Load balancers, including Nginx and Traefik, routed traffic to scaled replica sets. This approach optimized resource utilization, enhanced throughput, reduced latency, and cut costs.

### Building Resilience

We implemented a range of resiliency techniques, encompassing retry policies, timeouts, and circuit breakers. Rate limiting and throttling guarded against cascading failures. The Platform service took charge of transient issues related to dependent services.

A combination of in-house solutions and open-source libraries, such as Polly, Hystrix, and Resilience4j, bolstered our defenses. Centralized logging using Elasticsearch allowed us to trace errors across the distributed applications.

### Strengthening Reliability

Ensuring the reliability of microservices involved deploying various techniques to prevent single points of failure. Our focus was on automating responses to transient errors and fortifying protection against overload.

We leveraged the Resilience4J library to introduce circuit breakers for graceful fault handling:

```python
# OrdersService.java

@Slf4j
@Service
public class OrdersService {

  @CircuitBreaker(name="ordersService", fallbackMethod="fallback")
  public List<Order> getOrders() {
    return orderRepository.getOrders();
  }

  public List<Order> fallback(Exception e) {
    log.error("Circuit open, returning empty orders");
    return Collections.emptyList();
  }
}
```

Rate limiting was employed to prevent services from becoming overwhelmed during periods of high stress:

```python
# RateLimiterConfig.java

@Configuration
public class RateLimiterConfig {

  @Bean
  public RateLimiter ordersLimiter() {
    return RateLimiter.create(maxOrdersPerSecond); 
  }

}
```

Timeouts were introduced to terminate long-running calls:

```python
# ProductClient.java

@HystrixCommand(fallbackMethod="getProductFallback",
               commandProperties={
                 @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds", value="1000")  
               })
public Product getProduct(Long id) {
  return productService.getProduct(id);
}
```

Retry logic was enforced via policies defined at the client level:

```python
# RetryConfig.java

@Configuration
public class RetryConfig {

  @Bean
  public RetryTemplate ordersRetryTemplate() {
    SimpleRetryPolicy policy was introduced to provide a more reliable experience, and policies were defined at the client level:
    policy.setMaxAttempts(3);
    return a RetryTemplate(policy);
  }

} 
```

These measures ensured uniform responses and prevented cascading failures across services.

## In Conclusion

The journey of transforming our monolithic ERP system into a microservices architecture has been an immense learning experience. This endeavor transcends the realm of technical migration, signifying a profound organizational transformation that empowers our client to serve the evolving needs of their customers more effectively.

By dismantling tightly interwoven layers and establishing distinct boundaries between domains, our development team has unlocked a new realm of agility. Features can now be developed and deployed independently based on business priorities rather than being constrained by architectural limitations. This newfound ability to experiment and iterate swiftly will keep our application ahead of evolving market demands.

In parallel, our operations team now boasts complete visibility and control over each system component. Anomalous behavior can be detected early through enhanced monitoring of individual services. Scaling and failovers have transitioned from manual to automated processes. This heightened resilience serves as a reliable foundation for our client's continued growth.

While the benefits of microservices are evident, embarking on such a migration is far from straightforward. Our approach to analyze relationships, define interfaces, and introduce abstractions, rather than opting for a crude 'rip and replace' strategy, has resulted in a flexible architecture capable of evolving in line with customer needs.

Above all, I am grateful for the opportunity this project provided to work closely with a client on their digital transformation. Sharing our experiences and insights here is a way to pay it forward, enabling others to learn from our successes and mistakes. My hope is that it empowers more businesses to embark on the journey of modernization, with the ultimate rewards being enhanced customer experiences.

## References 

- MongoDB Documentation - Official documentation on data modeling, queries, deployment, and more: [Read more](https://docs.mongodb.com/)

- Microservices Pattern - Martin Fowler's seminal overview of microservices architecture principles: [Read more](https://martinfowler.com/articles/microservices.html)

- Domain-Driven Design - Eric Evans' book introducing DDD concepts for structuring services around business domains: [Read more](https://domainlanguage.com/ddd/)

- Twelve-Factor App Methodology - Best practices for building software-as-a-service apps readily deployable to the cloud: [Read more](https://12factor.net/)

- Container Journal - In-depth articles on containerization, orchestration, cloud platforms: [Read more](https://containerjournal.com/)
- Docker Documentation - Comprehensive guides for building, deploying, and managing containerized apps: [Read more](https://docs.docker.com/)

- Kubernetes Documentation - Kubernetes, the leading container orchestrator, and its architecture: [Read more](https://kubernetes.io/docs/home/)
