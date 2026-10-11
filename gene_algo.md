Solving the 3 AM WAN Meltdown: How We Use Genetic Algorithms and Digital Twins for IS-IS Traffic Engineering

By the Network Infrastructure & Automation Engineering Team
Reading Time: 9 minutes | Tags: #TrafficEngineering #Networking #GeneticAlgorithms #DistributedSystems #Dijkstra

![Image](https://images.unsplash.com/photo-1544197150-b99a580bb7a8?auto=format&fit=crop&w=1400&q=80)

    Figure 1: High-capacity IP backbones require continuous, coordinated metric balancing to prevent transient congestion.

1. The 3:00 AM Dilemma: The Multi-Link "Balloon Effect"

Every wide-area network (WAN) backbone engineer knows this scenario:

It’s 3:00 AM. A high-bandwidth fiber run between Singapore and Tokyo spikes to 94.69% utilization, sounding alarms across the NOC.

The instinctual fix is simple: log into the Singapore core router, bump the IS-IS link metric, and divert traffic. But five minutes later, a cascade begins:
code Code

[Singapore -> Tokyo] Metric: 7439 -> 8500  (Congestion relieved: drops to 52%)
       │
       ▼ (Traffic diverts via transit path)
[Manila -> Tokyo]    Utilization spikes to 133% ⚠️ CRITICAL OVERLOAD
       │
       ▼ (Engineer bumps Manila metric to 9000)
[Hong Kong -> Tokyo] Utilization spikes to 105% ⚠️ CRITICAL OVERLOAD

This is the "Balloon Effect" of IP routing: squeeze traffic out of one corridor, and it bulges out destructively in another.
code Code

┌─────────────────────────┐
                       │   EBR1.SNG3 (Singapore) │
                       └────────────┬────────────┘
                                    │
                  Baseline: 94.7%   │   Naive Metric Bump (+1000)
                     [CONGESTED]    │   Diverts traffic to Manila!
                                    ▼
       ┌────────────────────────────┴────────────────────────────┐
       │                                                         │
       ▼                                                         ▼
┌───────────────┐     Severe Spillover: 133%      ┌─────────────────────────┐
│ EBR1.HKG4     │ ◄────────────────────────────── │ EBR1.MNA4 (Manila)      │
│ (Hong Kong)   │                                 └────────────┬────────────┘
└───────┬───────┘                                              │
        │                                                      │
        └──────────────────────────┬───────────────────────────┘
                                   │
                                   ▼
                       ┌─────────────────────────┐
                       │   EBR2.TOK4 (Tokyo)     │
                       └─────────────────────────┘

Because wide-area IP networks run shortest-path interior gateway protocols (IGP) like IS-IS or OSPF, link adjustments cannot be evaluated in isolation.

To solve this, we built an evolutionary optimization engine inside our TE-SIM (Traffic Engineering Simulator) that computes optimal multi-link metric vectors in seconds—guaranteeing zero secondary overloads.
2. Why Machine Learning & Gradients Fail on IP Networks

If this is an optimization problem, why can't we use standard machine learning, gradient descent, or linear programming?

In 2000 and 2002, researchers Bernard Fortz and Mikkel Thorup (AT&T Labs) mathematically proved that the Weight Setting Problem (WSP) for shortest-path routing is NP-hard. It possesses two fatal characteristics that defeat standard optimizers:
A. Dijkstra is a Non-Differentiable Step Function

Gradient descent relies on smooth derivatives (

        
∂y∂x
∂x∂y​

      

). However, Dijkstra’s Shortest Path First (SPF) algorithm operates on discrete integer steps:
code Code

Link Utilization (%)
  100% ┼                               ┌─────────────────────
       │                               │ (Instant 40 Gbps Jump)
       │                               │
   60% ┼───────────────────────────────┘
       │  (Zero traffic moves)
    0% ┼─────────────────────────────────────────────────────
       7438           7439           7440           7441
                           Metric Setting

    Increasing a metric from 7438 to 7439 yields 0 Gbps of movement because the path is still the shortest.

    Increasing it from 7439 to 7440 causes the path cost to exceed an alternate path: an entire 40 Gbps aggregate trunk instantly jumps.

There is no slope (

        
dydx=0
dxdy​=0

      

almost everywhere, and

        
∞
∞

      

at the cliff). Standard optimizers stall out immediately.
B. Discrete Search Space Explosion

Even if we limit metric tuning to just a small corridor of 5 links with 5,000 allowable metric values each, the search space is:

        
Search Space=5,0005=3.125×1018 permutations
Search Space=5,0005=3.125×1018 permutations

      

Simulating SPF across 97,000+ demands for billions of combinations would take weeks. We need a heuristic that converges on a robust global optimum in seconds.
3. The Solution: Genetic Algorithms + Digital Twins

A Genetic Algorithm (GA) uses principles of natural evolution—chromosomes, fitness evaluation, elitism, and mutation—to iteratively evolve optimal solutions.
hub Diagram
Mermaid diagram
The Chromosome Encoding

Instead of tuning the entire network (1,300+ circuits), TE-SIM scopes the problem to a targeted 5-gene chromosome covering the bottleneck and its adjacent spillover paths:
code Code

┌──────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│    Gene 0    │    Gene 1    │    Gene 2    │    Gene 3    │    Gene 4    │
│   SNG-TOK    │   MNA-TOK    │   HKG-MNA    │   SNG-HKG    │   SNG-MNA    │
│ (Congested)  │  (Spillover) │  (Balance)   │  (Alt-Path)  │  (Alt-Path)  │
├──────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│    11,529    │    9,646     │    1,939     │    3,164     │    2,858     │
└──────────────┴──────────────┴──────────────┴──────────────┴──────────────┘

The Fitness (Loss) Function

In operational networking, an algorithm that improves one link by blowing out another is useless. We enforce this with an asymmetrical penalty:

        
Loss=Peak Network Utilization (%)+(Count of Links Over 80%×1,000)
Loss=Peak Network Utilization (%)+(Count of Links Over 80%×1,000)

      

    If any link exceeds 80%, the candidate gets an immediate +1,000 penalty per violation.

    If all links stay under 80%, the score is simply the network-wide peak utilization. The lower the score, the better the candidate.

4. Deep-Dive: Step-by-Step Numeric Walkthrough

Let’s trace an actual run of the algorithm through two generations using real network parameters.
code Code

Corridor Constraints:
• Threshold Limit: 80.0%
• Candidate Pool: Population of 3 candidates
• Demand Volume: Singapore -> Tokyo aggregate demand = 60 Gbps

Generation 0: Initialization

Three candidate chromosomes are initialized. Candidate 1 is the baseline configuration; Candidates 2 and 3 are sampled within our corridor bounds.
code Code

Candidate A (Baseline):  [ SNG-TOK: 7439,  MNA-TOK: 5150 ]
Candidate B (Naive Bump):[ SNG-TOK: 12500, MNA-TOK: 5150 ]
Candidate C (Coordinated):[ SNG-TOK: 11200, MNA-TOK: 9100 ]

Step 1: Simulate Traffic via Dijkstra

Our digital twin simulates the 60 Gbps demand matrix across the topology:

    Candidate A (Baseline):

        Direct SNG-TOK metric is low (7439). All 60 Gbps takes direct link.

        Direct link utilization: 94.7% ⚠️ (1 violation)

        Manila link utilization: 24.0%

    Candidate B (Naive Bump):

        SNG-TOK metric raised to 12500. Direct path becomes too expensive.

        45 Gbps diverts to Singapore

                
        →
        →

              

        Manila

                
        →
        →

              

        Tokyo.

        Direct link utilization drops to 42.0%.

        Manila-Tokyo link utilization hits 92.5% ⚠️ (1 violation)

    Candidate C (Balanced Coordinated):

        Both metrics scaled upward proportionally. Traffic splits evenly across transit paths and ECMP links.

        Direct link utilization: 76.2% (0 violations)

        Manila link utilization: 73.5% (0 violations)

Step 2: Calculate Fitness & Loss

        
Loss=Peak Util+(Violations×1000)
Loss=Peak Util+(Violations×1000)

      

Candidate	Peak Util	Violations	Loss Calculation	Total Score	Status
Candidate A	94.7%	1	

        
94.7+(1×1000)
94.7+(1×1000)

      

	1094.7	Failed
Candidate B	92.5%	1	

        
92.5+(1×1000)
92.5+(1×1000)

      

	1092.5	Failed
Candidate C	76.2%	0	

        
76.2+(0×1000)
76.2+(0×1000)

      

	76.2	Fittest (Winner)
Generation 1: Selection & Gaussian Mutation

Now the evolutionary cycle produces the next generation:

    Elitism: Candidate C (Score 76.2) and Candidate B (Score 1092.5) survive. Candidate A is eliminated.

    Mutation: TE-SIM generates an offspring from Candidate C by applying zero-mean Gaussian noise (

            
    σ=300
    σ=300

          

    ):

        
Child Metrici=Parent Metrici+N(0,σ)
Child Metrici​=Parent Metrici​+N(0,σ)

      

code Code

Parent Gene (Candidate C): [ SNG-TOK: 11,200 | MNA-TOK: 9,100 ]

Sampled Noise:
  • Gene 0 Noise: +329
  • Gene 1 Noise: +546

Child Candidate D:          [ SNG-TOK: 11,529 | MNA-TOK: 9,646 ]

Step 3: Evaluate Generation 1

We run the digital twin on our new child:
code Code

Simulating Candidate D:
  • SNG-TOK Utilization: 76.4%
  • MNA-TOK Utilization: 47.9%
  • Peak Network Util:   76.4%
  • Violations:          0
  • Loss Score:          76.4  <-- Optimal Headroom Achieved!

Within two generations, the algorithm isolated a metric combination that shed 18.3% of traffic off the congested link while preventing the Manila transit link from blowing past its ceiling.
5. Algorithmic Trade-offs: Why GA?

How does this compare to other industry traffic-engineering approaches?
Approach	Convergence Time	Native IS-IS Compatible?	Enforces Hard Limits (

        
<80%
<80%

      

)?	Operational Blast Radius
Genetic Algorithm (TE-SIM)	5–12 seconds	Yes	Yes	Bounded (3–5 links)
Local / Tabu Search	

        
<2
<2

      

seconds	Yes	Partially	Lowest (1–2 links)
Linear Programming (Multi-Commodity)	Sub-second	No (Requires arbitrary flow splitting)	Yes	Global
Integer Linear Programming (ILP)	Hours / Days	Yes	Yes	Unpredictable
Segment Routing (SR-TE / RSVP)	Instantaneous	N/A (Path-based)	Yes	Zero IGP impact (Path Tunnel)

    Key Architecture Takeaway: Linear Programming (LP) is common in academic literature, but it cannot be deployed directly on native IS-IS IP backbones. LP assumes routers can split traffic into arbitrary decimal fractions. Native IS-IS routers strictly adhere to equal-cost multipath (ECMP) and shortest paths. GA works because it optimizes directly within native router constraints.

6. The Production 3-Tier Architecture

In production, you don't run a cold GA search for every incident. We deploy a 3-Tier Triage Pipeline to ensure real-time response:
code Code

[ Congestion Alert: Utilization > 80% ]
                                             │
                                             ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ TIER 1: Bounded Local Neighborhood Search (< 1 sec)                                    │
│ • Tests 1-link and 2-link metric deltas in the immediate corridor.                     │
│ • Successfully resolves ~85% of minor congestion events without regional changes.      │
└────────────────────────────────────────────────────────────────────────────────────────┘
                                             │
                                  Did Tier 1 clear the alert?
                                             │
                             YES ────────────┴──────────── NO
                              │                             │
                              ▼                             ▼
                   [ Execute CLI Remediate ]     ┌───────────────────────────────────────┐
                                                 │ TIER 2: Genetic Algorithm (5 - 12 sec)│
                                                 │ • Scales search to 5–10 regional genes│
                                                 │ • Finds multi-hop bypass paths        │
                                                 └───────────────────────────────────────┘
                                                                    │
                                                                    ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ TIER 3: Proactive Offline Digital Twin (Cached Execution)                              │
│ • Runs continuously against current topology snapshots (`network.db`).                 │
│ • Pre-computes contingency remediations for common N-1 fiber cut scenarios.           │
│ • Mean-Time-To-Remediation (MTTR) when incidents occur: < 200 ms.                      │
└────────────────────────────────────────────────────────────────────────────────────────┘

7. Real-World Execution & Router Configuration

Here is an actual trace log from TE-SIM executing against our production topology:
code Text

[AI-GA] Loading topology: 1,299 circuits, 97,412 active demand pairs (44.7 Tbps)
[AI-GA] Initializing population=6, max_generations=3, mutation_rate=0.4
[AI-GA] Gen 0: Best Loss = 1094.69 (1 Violations, Peak = 94.69%)
[AI-GA] Gen 1: Best Loss = 78.10   (0 Violations, Peak = 78.10%)
[AI-GA] Gen 2: Best Loss = 77.00   (0 Violations, Peak = 77.00%)
[AI-GA] Gen 3: Best Loss = 77.00   (0 Violations, Peak = 77.00%)
[AI-GA] >> Optimal convergence reached in 11.49 seconds.

Generated Production Configuration

Once the optimizer completes, TE-SIM generates vendor-specific, atomic configuration changes ready for automated pipeline review:
Juniper Junos OS
code Junos

/* Step 1: Relieve primary Singapore to Tokyo congestion */
set protocols isis interface ae3.3 level 2 metric 11529
annotate protocols isis interface ae3.3 "TE-SIM: Relieve SNG-TOK utilization to 76.4%"

/* Step 2: Establish Manila transit headroom to absorb spillover */
set protocols isis interface ae6.6 level 2 metric 9646
annotate protocols isis interface ae6.6 "TE-SIM: Maintain Manila spillover headroom <= 48%"

commit check
commit comment "AUTO-TE: Relieve SNG-TOK congestion via GA-optimized metric vector"

Cisco IOS-XR
code Cisco

! TE-SIM Automated Remediation Vector
router isis 100
 interface Bundle-Ether3.3
  description ** Core Link to TOK4 **
  metric 11529 2
 !
 interface Bundle-Ether6.6
  description ** Transit Spillover Link to TOK4 **
  metric 9646 2
 !
!
commit comment "AUTO-TE: Relieve SNG-TOK congestion via GA-optimized metric vector"

8. Summary & Key Takeaways

    Isolated link tuning is an anti-pattern: Tweaking metrics on a single link in an interconnected WAN almost always creates downstream spillover ("The Balloon Effect").

    Dijkstra requires heuristic optimization: Because SPF routing behaves like a non-differentiable step function, gradient-based ML fails. Evolutionary Genetic Algorithms efficiently balance multi-link tradeoffs.

    Guardrails matter more than raw speed: A fitness function must penalize threshold violations heavily. An algorithm that lowers average network utilization while leaving a single link at 105% is unacceptable in production.

    Hybrid is best: Combine fast local search for simple incidents with GA for complex, multi-hop bypass scenarios.

Discussion & Questions

How does your team handle WAN congestion events? Do you rely on manual metrics, automated SR-TE controllers, or offline network planning models? Let's discuss in the comments below.
