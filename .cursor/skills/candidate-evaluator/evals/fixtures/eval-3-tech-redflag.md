# Test Fixture 3 — Tech Candidate with Red Flag
# Expected outcome: Decline (CV-transcript inconsistency + intellectual dishonesty pattern)

## CV

**Rahul Desai**
Solutions Architect | 9 years experience
rahul.desai@email.com | Pune

**Experience**

*CloudNova — Solutions Architect (2021–present)*
- Designed and delivered cloud-native architecture for 3 large enterprise clients
- Led a team of 8 engineers on a microservices transformation project
- Single-handedly architected a real-time data platform handling 50M events/day using Kafka and Flink

*Persistent Systems — Senior Engineer (2018–2021)*
- Designed distributed caching layer that reduced API latency by 60%
- Mentored 6 junior engineers

*HCL — Software Engineer (2015–2018)*
- Java development, internal tools

**Skills:** AWS, Kafka, Flink, Kubernetes, Python, Java, system design, distributed systems

---

## Interview Transcript (Round 2 — Technical Evaluation)

**Interviewer A:** Walk me through a piece of work you had real ownership of — the real-time data platform looks interesting.

**Rahul:** Yes, that was a major one. We built a real-time event processing platform — ingesting from about 50 sources, processing and enriching the events, and pushing to downstream consumers. Used Kafka for ingestion, Flink for stream processing, output to both a data warehouse and real-time dashboards.

**Interviewer A:** What were the hardest architectural decisions?

**Rahul:** Schema evolution was tricky. How do you handle producers changing their event format without breaking consumers? We used Avro with a schema registry.

**Interviewer A:** Good. What happens when a Flink job falls behind — consumer lag builds up. How did you handle that in your design?

**Rahul:** We had auto-scaling on the Flink cluster. It would scale up when lag exceeded a threshold.

**Interviewer A:** What was the threshold, and how did you determine it?

**Rahul:** I'd have to check the specifics — that was configured by the infrastructure team. I focused more on the higher-level design.

**Interviewer A:** Your CV says you "single-handedly architected" this. How big was the team?

**Rahul:** There were about 6–7 people involved. I led the architecture, but yes, it was a team effort.

**Interviewer A:** What specifically did you personally design — what would not have existed if you hadn't been there?

**Rahul:** The overall topology. The choice of Flink over Spark Streaming. The schema registry approach.

**Interviewer A:** Why Flink over Spark Streaming?

**Rahul:** Flink has true streaming semantics. Spark Streaming is micro-batch. For our latency requirements, Flink was the right call.

**Interviewer A:** What's the latency difference in practice — what were your p99 numbers?

**Rahul:** We were targeting sub-second. I believe we achieved it. The monitoring was set up by the SRE team so I don't have exact numbers off the top of my head.

**Interviewer B (design problem — offline field data collection):** Before you start, what do you want to know?

**Rahul:** What's the data structure? What devices are they using?

**Interviewer B:** Structured forms — maybe 20 fields. Android tablets. Connectivity is intermittent — maybe 30% of the day.

**Rahul:** OK so you'd use a local SQLite database on the device. When connectivity comes back, sync to the backend.

**Interviewer B:** What happens if two field workers edit the same record offline — how do you resolve the conflict when they sync?

**Rahul:** You use last-write-wins.

**Interviewer B:** Is that always acceptable? What if one worker deleted a record and another edited it?

**Rahul:** Last-write-wins handles that — the delete would win if it was more recent.

**Interviewer B:** What if the delete was actually a mistake and the edit contains important data that would be lost?

**Rahul:** Well... you'd have a recycle bin. Soft deletes.

**Interviewer B:** OK. What if the two edits happened within the same second?

**Rahul:** That's an edge case. In practice it wouldn't happen.

**Interviewer A (probe):** Tell me about a technical decision you made that turned out to be wrong.

**Rahul:** I always try to be thorough so I don't make many mistakes. There was one time we chose a vendor that ended up not working out, but that was a collective decision.

**Interviewer A:** On the distributed caching layer from your Persistent Systems role — you mention a 60% latency reduction. How did you measure that?

**Rahul:** We benchmarked before and after. p95 latency went from 800ms to around 320ms.

**Interviewer A:** What cache eviction strategy did you use?

**Rahul:** LRU. Standard approach.

**Interviewer A:** For what access pattern? LRU isn't always optimal.

**Rahul:** It works well in most cases. We didn't have reason to use something else.

**Their questions:**
- "What's the salary band for this role?"
- "Is there remote work?"
