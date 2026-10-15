# Spring Components & Routing

## Core Stereotypes (`@Component`, `@Controller`, `@Service`, `@Repository`)
- **`@Component`:** The base stereotype annotation indicating a class is a Spring-managed bean[cite: 5].
- **`@Controller`:** Marks a class as a Spring MVC controller (Presentation Layer) handling HTTP requests and returning views (HTML/JSP)[cite: 5].
- **`@Service`:** Marks the Business Logic layer[cite: 5]. Functionally, it does absolutely nothing special over `@Component`, but it clarifies human intent for Domain-Driven Design and allows easy targeting for Aspect-Oriented Programming (AOP)[cite: 5].
- **`@Repository`:** Marks the Data Access Layer (DAO)[cite: 5]. Beyond `@Component`, it actively translates database persistence exceptions into Spring's unified `DataAccessException` hierarchy[cite: 5].

## `@Controller` vs. `@RestController`
- **`@Controller` (Traditional MVC):** Assumes traditional web development. When a method returns a String, Spring passes it to the `ViewResolver` to find and render a physical HTML template (like Thymeleaf)[cite: 5].
- **`@RestController` (Modern APIs):** A meta-annotation combining `@Controller` and `@ResponseBody`[cite: 5]. It completely bypasses the `ViewResolver` and passes the returned Java object to an `HttpMessageConverter` (like Jackson) to serialize into raw JSON[cite: 5].

## Routing (`@RequestMapping` vs. `@GetMapping`)
- **`@RequestMapping`:** The original, generic routing annotation capable of handling any HTTP method[cite: 5].
- **Composed Annotations:** `@GetMapping`, `@PostMapping`, etc., were introduced in Spring 4.3 as shortcuts that pre-configure the exact HTTP method, eliminating boilerplate[cite: 5].
- **Architectural Best Practice:** Use `@RequestMapping` exclusively at the Class level to define the shared base URL (e.g., `/api/v1/orders`), and use specific annotations like `@GetMapping` at the Method level to keep endpoints clean and strictly aligned with REST semantics[cite: 5].

---

# Advanced Annotations & Configurations

## Lombok `@UtilityClass`
- **Purpose:** Eliminates the boilerplate required to safely structure generic, stateless helper classes in Java[cite: 5].
- **Mechanics:** It automatically makes the class `final`, makes all methods `static`, and generates a private constructor that throws an `UnsupportedOperationException` to strictly prevent instantiation or inheritance[cite: 5].

## Transactions (`@Transactional`)
- **Core Concept:** Guarantees ACID properties (specifically Atomicity) by wrapping the service method in an AOP Proxy[cite: 5]. The Proxy starts a database transaction before the method, commits upon successful completion, and rolls back if an unchecked exception (`RuntimeException`) is thrown[cite: 5]. *(Note: It does not roll back for checked exceptions unless explicitly configured[cite: 5]).*
- **Transaction Boundaries:** The start and end point of the transaction, which senior engineers strictly place at the Service Layer to orchestrate multiple repository calls successfully[cite: 5].
- **Propagation Levels:**
  - **`REQUIRED` (Default):** If a transaction already exists, join it; if not, create a new one[cite: 5]. *Risk:* If a nested method throws an exception, the entire combined transaction rolls back[cite: 5].
  - **`REQUIRES_NEW`:** Always pauses the existing transaction and starts a completely new, separate transaction[cite: 5]. Used heavily for Audit Logging so failures don't prevent the log from saving[cite: 5].

## CORS (`@CrossOrigin` vs. Global Config)
- **The Problem:** Modern browsers enforce the Same-Origin Policy, blocking responses when a frontend (e.g., `localhost:3000`) calls an API on a different origin (`localhost:8080`)[cite: 5].
- **Pre-Flight Requests (`OPTIONS`):** For complex requests (like POSTing JSON), browsers send an invisible `OPTIONS` request first to verify the server accepts the origin before sending the actual data[cite: 5].
- **`@CrossOrigin`:** The quick fix placed on a Controller[cite: 5]. (Avoid in production as it creates a maintenance nightmare when URLs change[cite: 5]).
- **Global Configuration (Production Standard):** Configure globally via a `WebMvcConfigurer` bean (`addCorsMappings()`)[cite: 5]. 
  - *Architect Trap:* If your application uses Spring Security, the MVC CORS config is silently bypassed[cite: 5]. You must configure CORS directly within the `SecurityFilterChain`[cite: 5].

---

# Data & Service Layers

## JPA vs. Hibernate ORM
- **JPA (Java Persistence API):** A standard specification (`jakarta.persistence`) that defines interfaces (`EntityManager`) and annotations (`@Entity`, `@Id`) but contains no execution logic[cite: 5].
- **Hibernate:** A concrete ORM framework that implements the JPA specification and provides the actual execution engine, plus extra vendor-specific features[cite: 5].
- **Relationship:** You code against JPA abstractions for portability, while Hibernate acts as the underlying provider performing the physical work behind the scenes[cite: 5].

## Content Negotiation (XML vs. JSON)
- By default, Spring uses `HttpMessageConverters` (Jackson) to serialize/deserialize JSON[cite: 5].
- **Accepting XML:** Add the `jackson-dataformat-xml` dependency, annotate the model with `@XmlRootElement`, and explicitly set `consumes = MediaType.APPLICATION_XML_VALUE` on the endpoint[cite: 5].
- **Efficient Conversion:** To convert XML payloads directly to JSON without relying on brittle lightweight libraries, use Jackson's `XmlMapper` to read the XML into a generic `JsonNode` tree, then immediately use `ObjectMapper` to write the tree out as a JSON string[cite: 5].

## Internationalization (i18n) & Error Messages
- **Architecture:** The client specifies the language via the `Accept-Language` HTTP header[cite: 5]. Spring's `AcceptHeaderLocaleResolver` extracts it and binds the `Locale` to a `ThreadLocal` context (`LocaleContextHolder`)[cite: 5].
- **Resolution:** When an error occurs, the Global Exception Handler (`@RestControllerAdvice`) queries a `MessageSource` (linked to `messages_{locale}.properties` resource bundles) using the current Locale to return translated JSON[cite: 5].
- **Thread Boundary Gotcha:** Because Locale is stored in `ThreadLocal`, asynchronous tasks (`@Async`) or reactive streams will lose the Locale context unless it is manually extracted and passed to the worker thread[cite: 5].

## Microservice Resilience (Circuit Breakers)
- **The Rule:** Never hard-code static error responses or wait endlessly for dead downstream services, which exhausts thread pools and causes cascading failures[cite: 5].
- **The Solution:** Use a Circuit Breaker (like Resilience4j). If a downstream API fails, the circuit "opens" and blocks outgoing requests instantly[cite: 5].
  - *Strategy A (Fail Fast):* If data is mandatory (e.g., payment processing), instantly return a `503 Service Unavailable` with a structured JSON error explaining the offline dependency[cite: 5].
  - *Strategy B (Graceful Degradation):* If data is optional (e.g., recommendations), use a Fallback Method to return a safe default, an empty list, or stale cached data with a `200 OK` so the UI remains functional[cite: 5].

## Hiding JSON Fields (DTO Serialization)
- **The DTO Pattern (Best Practice):** Never return Database Entities directly to the client[cite: 5]. Map the Entity to a Response DTO that physically lacks the sensitive fields (like passwords), mathematically guaranteeing it cannot leak[cite: 5].
- **Jackson Hack (`WRITE_ONLY`):** If sharing a single DTO for requests and responses, use `@JsonProperty(access = JsonProperty.Access.WRITE_ONLY)` on the password field[cite: 5]. This allows the client to send it in a POST payload, but Jackson will never include it in outgoing JSON responses[cite: 5].

## Large File Uploads
- **The Memory Trap:** Capturing a large video via `MultipartFile.getBytes()` directly into a byte array will instantly crash the JVM with an `OutOfMemoryError`[cite: 5].
- **App Layer Fix:** Configure Tomcat limits (`spring.servlet.multipart.max-file-size`) and stream the `InputStream` directly to disk without holding it in memory[cite: 5].
- **System Design Fix (Pre-Signed URLs):** Proxying massive files through the backend wastes CPU and bandwidth[cite: 5]. The API should authenticate the user, generate a secure AWS S3 Pre-Signed URL, and send it to the client[cite: 5]. The client then uploads the file directly to S3, bypassing the Spring backend entirely[cite: 5].

---
