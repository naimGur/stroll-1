# STROLL TASK - 1

The system design can have many improvements from optimizing the data read to distributing it to the user over a network. I wanted to narrow the problem by overviewing the system from scalability and data integrity perspectives. Extracted two main flows which must be handled well to serve the amount of request mentioned on the task.

1. **Question Assignment Access:** The process when user gets the question assigned to their cycle.
2. **Cycle Management and Scheduling:** Making the question assignments when a new cycle comes.

---

Here is the system diagram for the service overview

![](<stroll question assignment.png>)

There are the CRUD API services to configure and read the Regions, Assignments, Cycles and Qeustions. Then there is a Scheduler service to run in the shortest possible cycle length period.

---

The approaches below are my architectural approaches to both flows.

1. **Question Assignment Access:** One significant load is when users request to access their assigned questions. The simple logic when user wants to read the question would be to calculate the question by the current cycle (time based operation), and the country (location based operation). These processes inside a service which is under a heavy load is computationally expensive.
I would solve this issue by caching the question assignment by adding a scheduler service to the system. This scheduler service runs in the period of shortest possible cycle length to avoid the runs in vain.
What this service does is to pre-calculate the question assignment and populate the cache. And when a user requests their question, the Question service in the CRUD API first checks the Distributed Cache rather than computing the assignment from the database. Only if there's a cache miss does the system fall back to calculating the assignment from the database tables This significantly reduces the load on the database and improves response times since the Question service can retrieve pre-computed assignments directly from the cache.

2. **Cycle Management and Scheduling:** A complexity is here to manage different cycles across the regions. In the diagram, this responsibility is the Scheduler service's. How this service runs is, it interacts with the Cycle service in the CRUD API to determine cycle configuration, the Region service to handle region requirements such as regional timezones, and also interact with the Assignment service to write the question assignment to the DB.
This process shall be handled by a centralized coordinator to have single point of truth in such complex job. When scheduler runs, it first queries the Cycles table in the Database to determine which regions need updates. It then works with the Assignment service to pre-calculate the next set of questions for each region. These new assignments are stored in both the Assignments table in the Database and the Question Assignments in the Distributed Cache. The Scheduler ensures that all these updates are synchronized across regions by coordinating the timing of cache updates and database writes. This orchestrated approach allows the system to handle complex cycle configurations while maintaining consistency across all regions.
