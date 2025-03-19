## **Ways of Shortening URLs** ([Reference](https://github.com/kyungtaek-jonas-lim/jonas-api-master/blob/main/src/routes/common/urlRoutes.ts))
There are several common techniques for shortening URLs. Each method has its own advantages and disadvantages.


| **Method** | **Pros** | **Cons** | **Description** |
|-----------|----------|----------|--------------|
| **ShortID (`shortid`)** | Simple and fast | Slightly predictable, deprecated | Generates a random alphanumeric string for short URLs. **Base62 encoding** is applied to ensure URL safety. |
| **SHA-256 Hashing** | Same input → same output, prevents duplication | Slight chance of collision when shortened | Uses SHA-256 hashing and extracts the first 8 characters for a shortened URL. **Base62 encoding** is applied to make the hash URL-friendly. |
| **Sequential ID + Random Suffix** | Unique and harder to predict | Slightly longer URLs | Uses a sequential ID with a random suffix to improve uniqueness. **Base62 encoding** is applied to shorten the ID. |
| **UUID (v4)** | Universally unique | Too long for URL shortening | Generates a UUID and truncates it to create a shortened URL. **Base62 encoding** is used to shorten and format the UUID. |
| **ULID** | Time-ordered and unique | Slightly longer than other methods | Generates a ULID (Universally Unique Lexicographically Sortable Identifier). **Base62 encoding** is applied to ensure shorter, URL-safe output. |
| **Original URL Base62 Encoding** | Simple, no need for ID generation | Results in longer URLs, inefficient for large URLs | Directly encodes the original URL using Base62, but often results in longer URLs compared to other methods. Rarely used in typical URL shorteners. |

&nbsp;
---

**Recommendation:**

> All ID generation methods typically apply **Base62 encoding** to make the generated IDs shorter and URL-safe. **Base62** is not a standalone URL shortening method but an encoding mechanism applied to numeric or alphanumeric IDs. Directly encoding the original URL using Base62 is generally inefficient due to longer results. Among the ID generation techniques, **Sequential ID + Random Suffix** combined with **Base62 encoding** is a common and efficient approach for ensuring both uniqueness and readability in URL shortening services.