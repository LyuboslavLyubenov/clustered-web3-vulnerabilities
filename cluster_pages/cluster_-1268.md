# Cluster -1268

**Rank:** #400  
**Count:** 12  

## Label
All entries involve input manipulation leading to resource exhaustion—either via excessive data size, unbounded query complexity, or malformed requests—resulting in denial-of-service through computational or gas overhead.

## Cluster Information
- **Total Findings:** 12

## Examples

### Example 1

**Auto Label:** All entries involve input manipulation leading to resource exhaustion—either via excessive data size, unbounded query complexity, or malformed requests—resulting in denial-of-service through computational or gas overhead.  

**Original Text Preview:**

##### Description

Several GraphQL configurations could allow users to perform Denial-of-Service (Dos) attacks at the application level.

The following GraphQL issues have been identified:

* Alias Overloading
* Field Duplication
* Introspection-based Circular Query
* Introspection Query

**Alias overloading - Alias Overloading with 100+ aliases is allowed**

GraphQL's alias feature allows a client to perform the same operation multiple times in the same HTTP request. Increasing the number of aliases in the query had an impact on the response length and time from the server, thus opened a possibility of Denial-of-Service condition.

**Field duplication - Queries are allowed with 500 of the same repeated field**

GraphQL allowed duplicate or repetitive fields in the query. An end-user could make the server process the same field again and again `n` number of times. Increasing the number of same fields in the query had an impact on the response time from the server and thus opened a possibility of Denial-of-Service (Dos) condition.

**Introspection-based Circular Query**

The more the fragments are referencing each other in a query and the more the server response length and time increased. This opened a possibility of a Denial-of-Service (Dos) condition.

**Introspection Query**

The GraphQL introspection could be called by unauthenticated users.

##### Proof of Concept

1. Use a tool such as [graphql-cop](https://github.com/dolevf/graphql-cop) to identify GraphQL vulnerabilities. Run all the queries through a proxy using the flag `--proxy` to identify the vulnerable queries.

2. Example of a request utilizing Alias Overloading.

![GraphQL query with alias overloading.](https://halbornmainframe.com/proxy/audits/images/66e418f253199e3ba192898b)

3. Example of a request utilizing Field Duplication.

![GraphQL query with field duplication.](https://halbornmainframe.com/proxy/audits/images/66e418f853199e3ba192898e)

4. Example of a request utilizing Introspection-based Circular Query.

![GraphQL query with Introspection-based circular query.](https://halbornmainframe.com/proxy/audits/images/66e4190053199e3ba1928991)

5. Example of an Introspection Query.

![GraphQL introspection query.](https://halbornmainframe.com/proxy/audits/images/66e418e753199e3ba1928988)

##### Score

Impact: 4  
Likelihood: 2

##### Recommendation

**Resolve the Alias Overloading Vulnerability**

The alias functionality cannot be directly disabled. Instead, it is advisable to implement a validation process to ensure that the query doesn’t contain more aliases than the chosen maximum.

The following open source project may help to resolve the issue: [GraphQL No Alias Directive Validation](https://github.com/ivandotv/graphql-no-alias?tab=readme-ov-file#graphql-no-alias-directive-validation)

```
No alias directive for graphql mutation and query types. It can limit the amount of alias fields that can be used for queries and mutations, preventing batch attacks.
```

**Resolve the Field Duplication Vulnerability**

To prevent this attack, GraphQL servers should validate the incoming queries and reject any that contain duplicate fields. Additionally, servers could limit the complexity of queries by setting a limit on the response time, to prevent excessive resource consumption.

**Resolve the Introspection-based Circular Query Vulnerability**

Several possibilities exist to remediate this issue: either limit the maximum depth of the introspection query or limit the maximum elapsed time to execute a GraphQL query. Alternatively, disable introspection.

**Resolve the Introspection Query Vulnerability**

Disable the introspection query, or consider implementing authentication mechanisms to prevent unauthenticated users from retrieving it.

##### Remediation

**RISK ACCEPTED:** The **Fuelet team** accepted the risk derived from this issue.

##### References

- [OWASP: GraphQL Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/GraphQL\_Cheat\_Sheet.html)

- [OWASP: Testing GraphQL](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web\_Application\_Security\_Testing/12-API\_Testing/01-Testing\_GraphQL)

---
### Example 2

**Auto Label:** All entries involve input manipulation leading to resource exhaustion—either via excessive data size, unbounded query complexity, or malformed requests—resulting in denial-of-service through computational or gas overhead.  

**Original Text Preview:**

##### Description

Denial of Service (DoS) vulnerability that arises from the exploitation of SQL Injection (SQLi) vulnerabilities, specifically through the use of the PostgreSQL pg\_sleep function. By injecting malicious SQL queries that include the pg\_sleep function, an attacker can induce significant delays in the processing of database queries, effectively causing a denial of service for legitimate users and applications dependent on the database.

pg\_sleep is a function in PostgreSQL that pauses the execution of a query for a specified amount of time (in seconds). While designed for legitimate uses such as process timing and testing, in the hands of an attacker, it becomes a tool for inducing deliberate delays in database response times.

The exploitation involves an attacker identifying a SQL injection point within an application that interacts with a PostgreSQL database. The attacker then crafts a query incorporating the pg\_sleep function, causing the database to pause for a significant amount of time.

  

```
'; SELECT pg_sleep(10); --
```

  

This would cause the database to halt processing for 10 seconds, if injected into a vulnerable SQL query. Repeated or concurrent exploitation of this vulnerability can overwhelm the database, resulting in a DoS condition where legitimate requests cannot be processed in a timely manner.

Furthermore, there was not found any rate limit mechanisms (we have the PoC of sending +300req/min) which chained with this pg\_sleep(20) could stack a high amount of requests with the sleep which will cause a major DoS on the application.

**Note**: A video PoC was shared with the client showing the impact of the vulnerability.

##### Score

Impact: 5  
Likelihood: 5

##### Recommendation

It is recommended to use parameterized queries when dealing with SQL queries that contain user input. Parameterized queries allow the database to understand which parts of the SQL query should be considered as user input, thus solving SQL injection.

##### Remediation

**SOLVED**: The critical issue was solved by implementing proper parameterized queries.

---
### Example 3

**Auto Label:** Unbounded resource consumption via malformed or excessive inputs, leading to denial-of-service through improper validation and lack of limits on input size or frame volume.  

**Original Text Preview:**

##### Description

A vulnerability was discovered with the implementation of the HTTP/2 protocol in Golang prior to 1.21.9 and 1.22.2 versions. The current version used in app chain is 1.21.

An attacker can cause the HTTP/2 endpoint to read arbitrary amounts of header data by sending an excessive number of CONTINUATION frames. This causes excessive CPU consumption of the receiver device since there is no sufficient limitation on the amount of frames. Thus, It could be exploited to cause DoS.

Please note that, many HTTP/2 implementations (including Golang) did not properly limit the amount of CONTINUATION frames within a single stream.

**References:**

<https://kb.cert.org/vuls/id/421644>

<https://www.cve.org/CVERecord?id=CVE-2023-45288>

##### Score

Impact:   
Likelihood:

##### Recommendation

The issue is fixed in both Golang 1.21.9 and 1.22.2. However, If you are intending to use 1.21.X, It is recommended upgrading to 1.21.11 (the latest of 1.21.X) since it has some other security/bug fixes in net/http package.

##### Remediation

**ACKNOWLEDGED:** The **Artela team** acknowledged the issue.

---
