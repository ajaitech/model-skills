# Apache Commons Collections official sources

Match the resolved major/minor version and JDK first. Commons Collections 3.x and 4.x packages and APIs are not interchangeable.

| Need | Official URL | Crawl context |
|---|---|---|
| Project overview | https://commons.apache.org/proper/commons-collections/ | Confirm current capabilities, supported Java level, and the correct major line. |
| User guide | https://commons.apache.org/proper/commons-collections/userguide.html | Select the collection family and semantics before choosing a type. |
| API overview | https://commons.apache.org/proper/commons-collections/apidocs/ | Verify packages, types, inheritance, generics, thread-safety, and deprecations. |
| All members index | https://commons.apache.org/proper/commons-collections/apidocs/index-all.html | Locate an exact class, method, field, or constructor; then open its owning Javadoc page. |

Prefer the JDK collection API when it already meets the requirement. When Commons is justified, reuse existing project abstractions and test null handling, ordering, equality, mutation, iterator behavior, concurrency assumptions, serialization boundaries, and asymptotic cost.
