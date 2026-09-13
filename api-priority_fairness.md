# API Priority and Fairness.

Why we need it ??
-> suppose we are in the multi tenant cluster where we need to manage the different type of workloads but there are someof are like the critical application some of are the regular application. So, If both needs to scale up due to request but the critical application has more priority and regular one has the low priority so 


Feature | Api Priority Fairness        |  Pod Priority and Preemption
Scope   | API server request handling  |  Pod scheduling and resource allocation on nodes
Purpose | 