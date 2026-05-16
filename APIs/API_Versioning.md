# 🔄 API Versioning: Managing Change Without Breaking Clients

> *"Your API is a contract. Once clients depend on it, you can't just change it — you'll break their applications. API versioning is how you evolve your API while keeping existing integrations working."*

**⏱️ Estimated Time**: 15 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [RESTful APIs](./RESTful.md), [API Design Guidelines](./API_Design_Guidelines.md)

---

## 🤔 Why Version APIs?

```
THE PROBLEM:
  v1: GET /users/123 → { "name": "Alice", "email": "alice@mail.com" }
  
  New requirement: split "name" into "firstName" + "lastName"
  
  If you just change it:
  v2: GET /users/123 → { "firstName": "Alice", "lastName": "Smith", ... }
  
  → 500 mobile apps parsing "name" field just broke! 💥
  
THE SOLUTION: Version your API
  /v1/users/123 → { "name": "Alice Smith" }        (old clients still work)
  /v2/users/123 → { "firstName": "Alice", ... }    (new clients get new format)
```

---

## 🏗️ Versioning Strategies

```
STRATEGY 1: URL PATH VERSIONING (most common)
  GET /api/v1/users/123
  GET /api/v2/users/123
  ✅ Simple, explicit, cacheable
  ❌ URL changes are disruptive, not RESTful purists' choice
  Used by: Twitter, Stripe, Google Maps

STRATEGY 2: QUERY PARAMETER
  GET /api/users/123?version=1
  GET /api/users/123?version=2
  ✅ Optional, easy to default
  ❌ Easy to forget, caching complexity
  Used by: Google Data API, Amazon

STRATEGY 3: CUSTOM HEADER
  GET /api/users/123
  X-API-Version: 1
  ✅ Clean URLs, doesn't pollute path
  ❌ Not visible in browser, harder to test/share
  Used by: Azure, some internal APIs

STRATEGY 4: CONTENT NEGOTIATION (Accept header)
  GET /api/users/123
  Accept: application/vnd.myapp.v2+json
  ✅ Most RESTful, follows HTTP spec
  ❌ Complex, hard to test in browser
  Used by: GitHub API

STRATEGY 5: NO VERSIONING (evolve carefully)
  - Only add fields, never remove or rename
  - Use feature flags or capability negotiation
  ✅ Simplest for small APIs
  ❌ Eventually impossible to maintain
  Used by: Stripe (alongside URL versioning for breaking changes)
```

---

## 💻 Spring Boot Implementation

```java
// Strategy 1: URL Path Versioning
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    @GetMapping("/{id}")
    public UserResponseV1 getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return new UserResponseV1(user.getFullName(), user.getEmail());
    }
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    @GetMapping("/{id}")
    public UserResponseV2 getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return new UserResponseV2(user.getFirstName(), user.getLastName(), user.getEmail());
    }
}

// Strategy 3: Header-based versioning (single controller)
@RestController
@RequestMapping("/api/users")
public class UserController {
    @GetMapping(value = "/{id}", headers = "X-API-Version=1")
    public UserResponseV1 getUserV1(@PathVariable Long id) { ... }
    
    @GetMapping(value = "/{id}", headers = "X-API-Version=2")
    public UserResponseV2 getUserV2(@PathVariable Long id) { ... }
}

// Strategy 4: Content Negotiation
@RestController
@RequestMapping("/api/users")
public class UserController {
    @GetMapping(value = "/{id}", produces = "application/vnd.myapp.v1+json")
    public UserResponseV1 getUserV1(@PathVariable Long id) { ... }
    
    @GetMapping(value = "/{id}", produces = "application/vnd.myapp.v2+json")
    public UserResponseV2 getUserV2(@PathVariable Long id) { ... }
}
```

---

## 📊 Comparison Table

| Strategy | Visibility | Caching | RESTfulness | Simplicity | Adoption |
|---|---|---|---|---|---|
| URL Path | High | Easy | Low | High | Very High |
| Query Param | Medium | Moderate | Medium | High | Medium |
| Custom Header | Low | Moderate | Medium | Medium | Medium |
| Accept Header | Low | Complex | High | Low | Low |

---

## 🎯 Versioning Best Practices

```
1. DEFAULT VERSION: Always have a default (usually latest or v1)
   If no version specified → don't break → use default

2. DEPRECATION POLICY: Communicate timeline clearly
   "v1 deprecated Jan 2025, sunset July 2025"
   Add Sunset header: Sunset: Sat, 01 Jul 2025 00:00:00 GMT

3. MINIMIZE BREAKING CHANGES:
   ✅ Adding new fields (backward compatible)
   ✅ Adding new endpoints
   ❌ Removing fields (breaking)
   ❌ Renaming fields (breaking)
   ❌ Changing field types (breaking)

4. SUPPORT AT MOST 2-3 VERSIONS simultaneously
   More versions = more maintenance = more bugs
   
5. CHANGELOG: Maintain detailed API changelog
   Document every change between versions
```

---

## ⚠️ Common Pitfalls

1. **Too many versions** — Supporting 5+ versions becomes unmaintainable. Deprecate aggressively with clear timelines. Most teams support current + previous only.

2. **Versioning internal APIs** — Between your own microservices, prefer contract testing + backward-compatible changes over versioning. Versioning adds coupling.

3. **Breaking changes without bumping version** — Even "small" changes (changing a date format, removing a nullable field) can break clients. When in doubt, version.

---

## 📝 Interview Q&A

**Q: How would you migrate clients from API v1 to v2?**
> A: (1) **Announce deprecation** with timeline (6+ months for public APIs). (2) **Add Sunset/Deprecation headers** to v1 responses. (3) **Provide migration guide** documenting all changes. (4) **Monitor v1 usage** — identify who hasn't migrated. (5) **Contact stragglers** directly. (6) **Return 410 Gone** after sunset date. Key principle: never surprise clients with breaking changes.

---

## 🔗 What to Read Next

1. **[APIs/RESTful.md](./RESTful.md)** — REST API design fundamentals
2. **[APIs/GraphQL.md](./GraphQL.md)** — Schema-based alternative (no versioning needed)
3. **[APIs/API_Design_Guidelines.md](./API_Design_Guidelines.md)** — Complete design guide

---

*[← Webhooks](./Webhooks.md) | [Back to Index](../INDEX.md)*
