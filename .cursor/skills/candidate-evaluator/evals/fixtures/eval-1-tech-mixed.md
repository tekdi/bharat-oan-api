# Test Fixture 1 — Tech Candidate, Mixed Signals
# Expected outcome: Proceed with Caveats (strong technical depth, ownership concern)

## CV

**Priya Nair**
Senior Software Engineer | 7 years experience
priya.nair@email.com | Bangalore

**Experience**

*TechServe Solutions — Senior Engineer (2021–present)*
- Led architecture of a multi-tenant SaaS platform serving 50+ enterprise clients
- Designed and owned the data ingestion pipeline processing 2M events/day
- Mentored team of 4 junior engineers; drove adoption of microservices architecture across the org

*Infosys — Software Engineer (2019–2021)*
- Built RESTful APIs for a government e-procurement portal (state of Karnataka)
- Worked on a team delivering digital identity verification module

*TCS — Associate Engineer (2017–2019)*
- Full-stack development on internal tools

**Skills:** Python, Go, Kubernetes, PostgreSQL, Kafka, AWS, system design

---

## Interview Transcript (Round 2 — Technical Evaluation)

**Interviewer A:** Walk me through a piece of work you had real ownership of in the last year.

**Priya:** Sure. At TechServe I've been leading the architecture of our multi-tenant platform. We moved from a monolith to microservices over about 18 months. I was responsible for the overall design — the service boundaries, the data isolation strategy, the API gateway layer.

**Interviewer A:** What were the hardest decisions in that migration?

**Priya:** Honestly the hardest part was getting buy-in from the team. People were comfortable with the monolith. I had to do a lot of convincing.

**Interviewer A:** From a technical standpoint — what was the hardest design decision?

**Priya:** Data isolation was tricky. We had to decide between schema-per-tenant and row-level isolation. We went with schema-per-tenant.

**Interviewer A:** Why that over row-level?

**Priya:** It was cleaner. Easier to reason about. And the ops team preferred it.

**Interviewer A:** What were the downsides of that choice?

**Priya:** Schema management overhead. Running migrations across hundreds of schemas gets complex. We wrote tooling for it.

**Interviewer A:** Who designed that tooling?

**Priya:** That was more our DevOps engineer, Suresh. He built most of it. I specced the requirements.

**Interviewer B (design problem — government grievance portal):** Before you start — what do you want to know?

**Priya:** Who are the users? What's the expected volume? Is this web-only or mobile too?

**Interviewer B:** Citizens and government officers. Volume — say 10,000 grievances/day. Web and mobile.

**Priya:** OK so I'd start with... three main entities: Citizen, Grievance, Officer. The grievance gets submitted, routed to the right department, tracked through stages.

**Interviewer B:** How would routing work?

**Priya:** Probably a rules engine. Each grievance type maps to a department. We'd configure the mappings.

**Interviewer B:** What happens when an officer is on leave and a grievance is unassigned for 3 days?

**Priya:** That's an escalation flow. We'd set a threshold — say 48 hours — and auto-escalate to the supervisor.

**Interviewer B:** We've just been told the system needs to work in 6 languages. How does that change your design?

**Priya:** I'd add a localisation layer. Store content keys, pull translations from a config or a service. Standard i18n approach.

**Interviewer B:** What breaks first in this system at 10x the volume you assumed?

**Priya:** Probably the database writes — if grievances are coming in at 100k/day you'd want to think about write queuing. Maybe put Kafka in front of the ingestion layer.

**Interviewer A (probe):** Tell me about a time you made a technical call that turned out to be wrong.

**Priya:** Hmm. I don't know if there's been a major one. Our architecture decisions have generally held up well. There were some small things but nothing that significantly set us back.

**Interviewer A:** One more — how do you typically stay current with what's changing in the field?

**Priya:** I read engineering blogs, follow some people on LinkedIn. I try to attend meetups when I can.

**Their questions:**
- "What's the team size I'd be joining?"
- "What's the tech stack currently?"
- "What does the appraisal cycle look like?"
