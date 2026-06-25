# Writing Skills – Submission

---

## Q1: Is there any situation that describes an unconventional route taken in your experience?

### Situation
In recharge service platform, I was assigned a new requirement to store each user specific data for future analysis. The initial approach was to get these bills from external service, then validate and format the data, and store it in database synchronously.

### Task
My responsibility was to develop this feature without any impact on performance and scalability of the service.

### Action
While analyzing the solution, I identified that additional processing and validating the bill would increase latency of this critical service and create a scalability bottleneck.

Further analyzing, I recommended a different approach using Cassandra database, as it is suitable for high write throughput which aligned with our requirements. I also proposed an asynchronous event-driven approach using Kafka which can decouple the formatting and validation of the data. This reduced the response time of the service and made it lightweight.

### Result
This solution fulfilled the business requirements to store bills from external sources for further analysis without impacting the response time of the request service.

Looking back, it was not just implementing a solution but improving the service for a more scalable and maintainable approach. This enabled better future growth of the platform.

---

## Q2: Do you recall a situation where you intensely and relentlessly focused to complete a task that helped you grow?

### Situation
During a cloud migration initiative, the task was to migrate an existing service to a new cloud environment. The service consisted of Kotlin and Java 8 code with a strict deadline.

### Task
My responsibility was to migrate the service to a new cloud environment with no business impact.

### Action
As I evaluated, the new environment was standardized on Java 21 and Spring Boot 3. If we only migrated the service, it would require additional maintenance and repeated solutioning for every enhancement.

After analyzing the risks and future efforts, I proposed not just migration but upgrading the codebase from Kotlin and Java 8 to Java 21 and Spring Boot 3, aligned with other services. I focused on delivery within the deadline while resolving framework migration challenges and ensuring no compatibility issues through iterative testing.

### Result
The project was completed within the assigned timeline. The service was aligned with the modern tech stack, reducing future maintenance and separate solutioning for enhancements.

This task helped me understand the importance of looking beyond immediate solutions and balancing short-term delivery with long-term maintainability.

---

## Q3: Experience of joining a new team or complex environment

I joined a large microservices environment with significant service-to-service communication, business flows, and architecture complexity.

To avoid being overwhelmed, I followed a layered approach. First, I understood the system at a high level. When documentation was limited, I created diagrams for service-to-service communication.

Next, I learned service by service, focusing only on those required for my task. I traced requests through services and databases to understand the end-to-end flow. Once I had clarity, I discussed with teammates to validate and fill knowledge gaps.

This helped me understand the system and start working on feature development and deployment. Later, I was able to resolve production issues as well.

This experience taught me that becoming effective in a complex environment is more about understanding business flow step by step rather than trying to learn everything upfront.
