# Implementing a new feature in a fastifyjs plugin
## Case study: Implement ThrottleStream in @fastify/fastify-rate-limit
--------------------------------------------------------------------------------
Implementing a new feature is a daily task for an average developer. Issues
arise when implementing a new feature, which should be used in backend
applications. While a new feature in a backend application is critically
inspected by your colleagues and maybe bugs are reported by the use of your
backend application, implementing a new feature in a widely used community
project has the critical eyes of other maintainers and the necessity to supply
high quality code. Developers, who use the community project, build over time
trust in the code quality and that trust can be shattered if you supply bad
code.

I want to show my approach for implementing a new feature. Maybe you disagree on
some aspects and I am happy to discuss them with you.

### The Feature Request: Limit the download speed of requests

On 29th of April user Uriel29 created an issue in @fastify/fastify-rate-limit,
regarding the possibilty to limit the download speed of requests. This feature
was not implemented yet in fastify-rate-limit and mcollina labeled the issue as
`enhancement` and highly appreciated if that solution would be implemented.

Shortly after a user asked for more information and showed interest in
implementing the feature. I don't want to blame the user. Simple tasks are
marked with `good-first-issue`. But it is not a simple task.

Describing a potentail approach for the solution takes more time than
implementing it. 

