---
layout: post
title: Dynamic json responses with jackson
slug: How to control json responses with jackson on a per-request basis to implement an expand mechanism
---

It is a common pattern in Rest services to have an expand query
parameter with which the client can control which parts of a probably
big response should be returned.  
Atlassian for example uses it for the jira rest api: https://docs.atlassian.com/jira/REST/cloud/ section "Expansion".

At the company I work for we searched for a solution for this problem for java rest services using https://github.com/FasterXML/jackson.  
At the server side we are using spring-mvc.

Our first try was to use https://github.com/monitorjbl/json-view. But we found out that this solution did not work together well with special serializers. E.g. we had a serializer for Date values which was defined on one of our DTOs with ``@JsonSerialize``. This serializer worked well without json-view but was completely ignored when using it. Instead the serialization produced timestamp values.  
After having a look at the json-view code we did not see a workaround or simple solution to solve this.

So we started with another solution.

Some ideas have been taken from the json-view. So, the general mechanism is the following:

1. in a controller method a thread-local variable is set containing the information which fields from the response should be included and excluded. This can be set using a fluent api in a class called ``JsonProfile`` (coming later).
2. we are using an adjusted ``ObjectMapper`` having a ``JsonProfileFilter`` which extends ``SimpleBeanPropertyFilter``.
3. the ``JsonProfileFilter`` gets the thread-local json profile and decides which fields to serialize
4. an interceptor resets the thread-local json profile after request completion
5. to make the ``JsonProfileFilter`` work the DTOs have to be annotated with ``@JsonFilter("jsonProfile")``

Now for some detail code:

The usage of the ``JsonProfile`` in a controller looks like this:

     JsonProfile.get()
        .exclude(ClassA.class, "*")
        .include(ClassA.class, "id")
        .include(ClassA.class, expandedProperties);

where ``expandedProperties`` is set from a request parameter ``expand``.
This says: exclude every field from ClassA except ``id`` and the properties to be expanded. If the parameter ``expand`` is not set, only the ``id`` will be contained in the response.

The complete JsonProfile class is
<script src="https://gist.github.com/aabeling/b27369c68e6f69008c774a9cd02174e9.js"></script>

The jackson framework is initialized with some adjustments.

In a configuration extending ``WebMvcConfigurationSupport``:
<script src="https://gist.github.com/aabeling/441a7819e2070139f2fa2743968d4fe8.js"></script>
where ``objectMapperWithFilter`` is
<script src="https://gist.github.com/aabeling/e0502e7bc59bedab61d475396adfabe1.js"></script>

And the ``JsonProfileFilter`` is

<script src="https://gist.github.com/aabeling/625f3b9a25dc0d770a79f242a5df3834.js">
</script>

The whole solution is not as easy to use than json-view but it works for us. Now, one we made the preparations we just have to put the ``@JsonProfile`` annotation on DTOs where we want to expand and use the ``JsonProfile`` in the controller methods.
