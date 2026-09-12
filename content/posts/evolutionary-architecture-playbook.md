---
title: "Building Evolutionary Architectures: Book Review"
date: 2026-09-12
draft: false
tags: ["architecture", "evolutionary-architecture", "fitness-functions", "microservices", "conways-law", "legacy", "migration"]
categories: ["architecture", "engineering"]
summary: "Chapter-by-chapter review of 'Building Evolutionary Architectures' by Neal Ford, Rebecca Parsons and Patrick Kua: the problem each chapter solves, the key ideas and a diagram. Fitness functions, the architectural quantum, Expand/Contract, the Inverse Conway Maneuver, antipatterns and legacy migration strategies."
ShowToc: true
---

A review of **"Building Evolutionary Architectures"** by Neal Ford, Rebecca Parsons and Patrick Kua. One section per chapter: the problem it solves, the key ideas, and a diagram.

## Core thesis: three pillars

An evolutionary architecture supports **guided incremental change across multiple dimensions**. The operative word is *guided*: the system does not drift, it changes under the control of automated checks.

The three pillars overlap, and they only work together:

| Pillar | What it gives you |
|---|---|
| Incremental change | iterative build and deployment |
| Fitness functions | protection of architectural characteristics |
| Appropriate coupling | architectural quanta and boundaries |

<div class="svg-diagram">
<svg viewBox="0 0 720 430" role="img" aria-label="Venn diagram of the three pillars: incremental change, fitness functions, appropriate coupling">
  <circle cx="255" cy="170" r="118" fill="#4ade8010" stroke="#60a5fa" stroke-width="1.5"/>
  <circle cx="465" cy="170" r="118" fill="#4ade8010" stroke="#60a5fa" stroke-width="1.5"/>
  <circle cx="360" cy="305" r="118" fill="#4ade8010" stroke="#60a5fa" stroke-width="1.5"/>
  <circle cx="360" cy="215" r="34" fill="#4ade8038"/>
  <text x="238" y="132" text-anchor="middle" fill="#e5e7eb" font-size="15" font-weight="600">Incremental</text>
  <text x="238" y="152" text-anchor="middle" fill="#e5e7eb" font-size="15" font-weight="600">change</text>
  <text x="238" y="175" text-anchor="middle" fill="#8b90a0" font-size="11">iterative build</text>
  <text x="238" y="190" text-anchor="middle" fill="#8b90a0" font-size="11">and deploy</text>
  <text x="484" y="132" text-anchor="middle" fill="#e5e7eb" font-size="15" font-weight="600">Fitness</text>
  <text x="484" y="152" text-anchor="middle" fill="#e5e7eb" font-size="15" font-weight="600">functions</text>
  <text x="484" y="175" text-anchor="middle" fill="#8b90a0" font-size="11">protect architectural</text>
  <text x="484" y="190" text-anchor="middle" fill="#8b90a0" font-size="11">characteristics</text>
  <text x="360" y="348" text-anchor="middle" fill="#e5e7eb" font-size="15" font-weight="600">Appropriate coupling</text>
  <text x="360" y="371" text-anchor="middle" fill="#8b90a0" font-size="11">architectural quanta</text>
  <text x="360" y="386" text-anchor="middle" fill="#8b90a0" font-size="11">and boundaries</text>
</svg>
</div>

## Chapter 1. Guarding against decay through multiple dimensions

**The problem.** Traditional long-term planning does not work. Systems decay (bit rot) because the ecosystem around them keeps shifting: frameworks, business requirements, scale.

**Key ideas:**

- *evolvability* is the new primary architectural characteristic;
- it is a meta-characteristic: a shell that protects every other property of the system;
- architecture must treat **time** as a first-class element.

<div class="svg-diagram">
<svg viewBox="0 0 720 390" role="img" aria-label="Architectural dimensions travelling through time inside an evolvability boundary">
  <rect x="18" y="46" width="684" height="322" rx="26" fill="#4ade8008" stroke="#4ade80" stroke-width="1.5"/>
  <text x="360" y="76" text-anchor="middle" fill="#4ade80" font-size="14" font-weight="600">Evolvability (fitness functions)</text>
  <text x="648" y="106" text-anchor="middle" fill="#8b90a0" font-size="12">time →</text>
  <line x1="196" y1="120" x2="668" y2="120" stroke="#4ade80" stroke-width="2" marker-end="url(#ea)"/>
  <text x="182" y="125" text-anchor="end" fill="#e5e7eb" font-size="13">Auditability</text>
  <line x1="196" y1="156" x2="668" y2="156" stroke="#f87171" stroke-width="2" marker-end="url(#er)"/>
  <text x="182" y="161" text-anchor="end" fill="#e5e7eb" font-size="13">Performance</text>
  <line x1="196" y1="192" x2="668" y2="192" stroke="#4ade80" stroke-width="2" marker-end="url(#ea)"/>
  <text x="182" y="197" text-anchor="end" fill="#e5e7eb" font-size="13">Security</text>
  <line x1="196" y1="228" x2="668" y2="228" stroke="#60a5fa" stroke-width="2" marker-end="url(#eb)"/>
  <text x="182" y="233" text-anchor="end" fill="#e5e7eb" font-size="13">Requirements</text>
  <line x1="196" y1="264" x2="668" y2="264" stroke="#fbbf24" stroke-width="2" marker-end="url(#ey)"/>
  <text x="182" y="269" text-anchor="end" fill="#e5e7eb" font-size="13">Data</text>
  <line x1="196" y1="300" x2="668" y2="300" stroke="#60a5fa" stroke-width="2" marker-end="url(#eb)"/>
  <text x="182" y="305" text-anchor="end" fill="#e5e7eb" font-size="13">Legality</text>
  <line x1="196" y1="336" x2="668" y2="336" stroke="#fbbf24" stroke-width="2" marker-end="url(#ey)"/>
  <text x="182" y="341" text-anchor="end" fill="#e5e7eb" font-size="13">Scalability</text>
  <defs>
    <marker id="ea" markerWidth="9" markerHeight="9" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3 z" fill="#4ade80"/></marker>
    <marker id="eb" markerWidth="9" markerHeight="9" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3 z" fill="#60a5fa"/></marker>
    <marker id="ey" markerWidth="9" markerHeight="9" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3 z" fill="#fbbf24"/></marker>
    <marker id="er" markerWidth="9" markerHeight="9" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3 z" fill="#f87171"/></marker>
  </defs>
</svg>
</div>

## Chapter 2. Fitness functions as an immune system

**The problem.** How do you objectively measure and protect critical characteristics — speed, security, resilience — when hundreds of developers change the code every day?

**Key ideas:**

- an objective assessment of system integrity; the term comes from evolutionary computing;
- categorised along two axes: atomic vs holistic, triggered vs continuous;
- identify the metric for every *-ility* up front, not after the fact.

<div class="svg-diagram">
<svg viewBox="0 0 720 400" role="img" aria-label="Systemwide fitness function composed of unit tests, contract tests, integration tests, monitoring and process metrics">
  <circle cx="360" cy="200" r="74" fill="#60a5fa22" stroke="#4ade80" stroke-width="2"/>
  <text x="360" y="190" text-anchor="middle" fill="#e5e7eb" font-size="13" font-weight="600">Systemwide</text>
  <text x="360" y="208" text-anchor="middle" fill="#e5e7eb" font-size="13" font-weight="600">Fitness</text>
  <text x="360" y="226" text-anchor="middle" fill="#e5e7eb" font-size="13" font-weight="600">Function</text>
  <rect x="278" y="24" width="164" height="46" rx="8" fill="#101114" stroke="#262a31"/>
  <text x="360" y="52" text-anchor="middle" fill="#e5e7eb" font-size="13">Process metrics</text>
  <line x1="360" y1="126" x2="360" y2="76" stroke="#8b90a0" stroke-width="1.6" marker-end="url(#fg)"/>
  <rect x="24" y="120" width="150" height="46" rx="8" fill="#101114" stroke="#262a31"/>
  <text x="99" y="148" text-anchor="middle" fill="#e5e7eb" font-size="13">Unit tests</text>
  <line x1="292" y1="172" x2="180" y2="150" stroke="#8b90a0" stroke-width="1.6" marker-end="url(#fg)"/>
  <rect x="546" y="120" width="150" height="46" rx="8" fill="#101114" stroke="#262a31"/>
  <text x="621" y="148" text-anchor="middle" fill="#e5e7eb" font-size="13">Monitoring</text>
  <line x1="428" y1="172" x2="540" y2="150" stroke="#8b90a0" stroke-width="1.6" marker-end="url(#fg)"/>
  <rect x="24" y="300" width="150" height="46" rx="8" fill="#101114" stroke="#262a31"/>
  <text x="99" y="328" text-anchor="middle" fill="#e5e7eb" font-size="13">Contract tests</text>
  <line x1="296" y1="240" x2="180" y2="302" stroke="#8b90a0" stroke-width="1.6" marker-end="url(#fg)"/>
  <rect x="546" y="300" width="150" height="46" rx="8" fill="#101114" stroke="#262a31"/>
  <text x="621" y="322" text-anchor="middle" fill="#e5e7eb" font-size="13">Integration</text>
  <text x="621" y="338" text-anchor="middle" fill="#e5e7eb" font-size="13">tests</text>
  <line x1="424" y1="240" x2="540" y2="302" stroke="#8b90a0" stroke-width="1.6" marker-end="url(#fg)"/>
  <defs>
    <marker id="fg" markerWidth="9" markerHeight="9" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3 z" fill="#8b90a0"/></marker>
  </defs>
</svg>
</div>

## Chapter 3. Engineering incremental change

**The problem.** Large changes are expensive and carry fatal risk. The result is a business that fears deployment, and release paralysis.

**Key ideas:**

1. Small increments shrink the blast radius of a mistake.
2. Hypothesis-driven development.
3. Full automation of the routine through Continuous Delivery.

<div class="svg-diagram">
<svg viewBox="0 0 960 250" role="img" aria-label="Deployment pipeline from commit to production with fitness function gates between stages">
  <rect x="14" y="74" width="104" height="56" rx="8" fill="#101114" stroke="#60a5fa"/>
  <text x="66" y="108" text-anchor="middle" fill="#e5e7eb" font-size="13">Commit</text>
  <line x1="122" y1="102" x2="146" y2="102" stroke="#8b90a0" stroke-width="1.8" marker-end="url(#pg)"/>
  <rect x="150" y="74" width="104" height="56" rx="8" fill="#101114" stroke="#60a5fa"/>
  <text x="202" y="108" text-anchor="middle" fill="#e5e7eb" font-size="13">Build</text>
  <line x1="258" y1="102" x2="282" y2="102" stroke="#8b90a0" stroke-width="1.8" marker-end="url(#pg)"/>
  <rect x="286" y="68" width="48" height="68" rx="8" fill="#4ade8018" stroke="#4ade80"/>
  <text x="310" y="110" text-anchor="middle" fill="#4ade80" font-size="22">✓</text>
  <text x="310" y="156" text-anchor="middle" fill="#8b90a0" font-size="10">gate</text>
  <line x1="338" y1="102" x2="362" y2="102" stroke="#8b90a0" stroke-width="1.8" marker-end="url(#pg)"/>
  <rect x="366" y="74" width="124" height="56" rx="8" fill="#101114" stroke="#60a5fa"/>
  <text x="428" y="108" text-anchor="middle" fill="#e5e7eb" font-size="13">Integration</text>
  <line x1="494" y1="102" x2="518" y2="102" stroke="#8b90a0" stroke-width="1.8" marker-end="url(#pg)"/>
  <rect x="522" y="68" width="48" height="68" rx="8" fill="#4ade8018" stroke="#4ade80"/>
  <text x="546" y="110" text-anchor="middle" fill="#4ade80" font-size="22">✓</text>
  <text x="546" y="156" text-anchor="middle" fill="#8b90a0" font-size="10">gate</text>
  <line x1="574" y1="102" x2="598" y2="102" stroke="#8b90a0" stroke-width="1.8" marker-end="url(#pg)"/>
  <rect x="602" y="74" width="104" height="56" rx="8" fill="#101114" stroke="#60a5fa"/>
  <text x="654" y="108" text-anchor="middle" fill="#e5e7eb" font-size="13">Deploy</text>
  <line x1="710" y1="102" x2="734" y2="102" stroke="#8b90a0" stroke-width="1.8" marker-end="url(#pg)"/>
  <rect x="738" y="68" width="48" height="68" rx="8" fill="#f8717118" stroke="#f87171"/>
  <text x="762" y="110" text-anchor="middle" fill="#f87171" font-size="22">✕</text>
  <text x="762" y="156" text-anchor="middle" fill="#f87171" font-size="10">gate</text>
  <line x1="790" y1="102" x2="814" y2="102" stroke="#262a31" stroke-width="1.8" stroke-dasharray="4 4"/>
  <rect x="818" y="74" width="126" height="56" rx="8" fill="#101114" stroke="#262a31"/>
  <text x="881" y="108" text-anchor="middle" fill="#8b90a0" font-size="13">Production</text>
  <text x="762" y="196" text-anchor="middle" fill="#f87171" font-size="12">faulty build stops here</text>
  <text x="480" y="36" text-anchor="middle" fill="#8b90a0" font-size="12">every stage transition is guarded by a fitness function gate</text>
  <defs>
    <marker id="pg" markerWidth="9" markerHeight="9" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3 z" fill="#8b90a0"/></marker>
  </defs>
</svg>
</div>

## Chapter 4. Anatomy of the architectural quantum

**The problem.** Accidental coupling — a change in one place breaks the whole system through hidden dependencies.

**Key ideas:**

- an architectural quantum is the smallest independently deployable unit with high functional cohesion inside;
- a quantum **always** includes its database: transactional boundaries are the strongest form of coupling.

> **If the database is shared, it is no longer a quantum.**

<div class="svg-diagram">
<svg viewBox="0 0 720 330" role="img" aria-label="Architectural quantum containing code, external library and its own database">
  <rect x="46" y="40" width="628" height="196" rx="16" fill="#60a5fa0d" stroke="#4ade80" stroke-width="2"/>
  <text x="360" y="70" text-anchor="middle" fill="#4ade80" font-size="14" font-weight="600">Architectural quantum</text>
  <rect x="80" y="104" width="164" height="72" rx="10" fill="#101114" stroke="#60a5fa"/>
  <text x="162" y="136" text-anchor="middle" fill="#e5e7eb" font-size="13">Code</text>
  <text x="162" y="155" text-anchor="middle" fill="#8b90a0" font-size="12">(module)</text>
  <rect x="278" y="104" width="164" height="72" rx="10" fill="#101114" stroke="#60a5fa"/>
  <text x="360" y="136" text-anchor="middle" fill="#e5e7eb" font-size="13">External</text>
  <text x="360" y="155" text-anchor="middle" fill="#e5e7eb" font-size="13">library</text>
  <rect x="476" y="104" width="164" height="72" rx="10" fill="#4ade8014" stroke="#4ade80"/>
  <text x="558" y="136" text-anchor="middle" fill="#e5e7eb" font-size="13">Database</text>
  <text x="558" y="155" text-anchor="middle" fill="#8b90a0" font-size="12">(data)</text>
  <line x1="244" y1="140" x2="272" y2="140" stroke="#8b90a0" stroke-width="1.6"/>
  <line x1="442" y1="140" x2="470" y2="140" stroke="#4ade80" stroke-width="4"/>
  <text x="360" y="206" text-anchor="middle" fill="#8b90a0" font-size="11">transactional boundary = strongest coupling</text>
  <text x="360" y="284" text-anchor="middle" fill="#f87171" font-size="14" font-weight="600">Shared database → no longer a quantum</text>
</svg>
</div>

## Chapter 4. Diagnostic matrix of styles

**The problem.** Picking the wrong architectural template — one that resists change and does not match the pace of the business (Big Ball of Mud, for example).

**Key idea.** Different styles carry a different built-in quantum size. The smaller the quantum, the faster and cheaper the change — but the higher the cost of running the infrastructure.

| Characteristic | Monolith | SOA | Microservices | Serverless |
|---|---|---|---|---|
| Quantum size | Large | Medium | Small | Micro |
| Coupling level | High | High | Low | Low |
| Incremental change | Hard | Hard | Easy | Easy |
| Fitness functions | Hard | Medium | Easy | Easy |

## Chapter 5. Evolving data on a live system

**The problem.** The illusion that "database schemas live forever". The database becomes a bottleneck and a point of rigid integration that blocks refactoring of the code.

**Key ideas:**

- data must evolve in lockstep with code;
- abandon Shared Database Integration;
- use the Expand/Contract pattern to migrate without downtime.

<div class="svg-diagram">
<svg viewBox="0 0 760 210" role="img" aria-label="Expand, contract migration pattern in four steps">
  <circle cx="86" cy="86" r="28" fill="#4ade8018" stroke="#4ade80" stroke-width="2"/>
  <text x="86" y="93" text-anchor="middle" fill="#4ade80" font-size="17" font-weight="600">1</text>
  <text x="86" y="146" text-anchor="middle" fill="#e5e7eb" font-size="13">Expand</text>
  <text x="86" y="167" text-anchor="middle" fill="#8b90a0" font-size="11">add the new shape</text>
  <line x1="122" y1="86" x2="200" y2="86" stroke="#8b90a0" stroke-width="1.8" marker-end="url(#xg)"/>
  <circle cx="246" cy="86" r="28" fill="#4ade8018" stroke="#4ade80" stroke-width="2"/>
  <text x="246" y="93" text-anchor="middle" fill="#4ade80" font-size="17" font-weight="600">2</text>
  <text x="246" y="146" text-anchor="middle" fill="#e5e7eb" font-size="13">Triggers</text>
  <text x="246" y="167" text-anchor="middle" fill="#8b90a0" font-size="11">two-way sync</text>
  <line x1="282" y1="86" x2="360" y2="86" stroke="#8b90a0" stroke-width="1.8" marker-end="url(#xg)"/>
  <circle cx="436" cy="86" r="28" fill="#4ade8018" stroke="#4ade80" stroke-width="2"/>
  <text x="436" y="93" text-anchor="middle" fill="#4ade80" font-size="17" font-weight="600">3</text>
  <text x="436" y="146" text-anchor="middle" fill="#e5e7eb" font-size="13">Migrate</text>
  <text x="436" y="167" text-anchor="middle" fill="#8b90a0" font-size="11">move readers/writers</text>
  <line x1="472" y1="86" x2="550" y2="86" stroke="#8b90a0" stroke-width="1.8" marker-end="url(#xg)"/>
  <circle cx="626" cy="86" r="28" fill="#4ade8018" stroke="#4ade80" stroke-width="2"/>
  <text x="626" y="93" text-anchor="middle" fill="#4ade80" font-size="17" font-weight="600">4</text>
  <text x="626" y="146" text-anchor="middle" fill="#e5e7eb" font-size="13">Contract</text>
  <text x="626" y="167" text-anchor="middle" fill="#8b90a0" font-size="11">drop the old shape</text>
  <defs>
    <marker id="xg" markerWidth="9" markerHeight="9" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3 z" fill="#8b90a0"/></marker>
  </defs>
</svg>
</div>

## Chapter 6. The mechanics of building evolution

**The problem.** There is no clear, repeatable algorithm for designing a new system or refactoring a huge legacy codebase.

**Key ideas:**

- a strict three-step algorithm turns evolutionary architecture from an abstraction into a practice: identify the dimensions → define the fitness functions → automate them in the pipeline;
- isolating the old code is critical to the survival of the new — an anticorruption layer sits between legacy and the new services.

<div class="svg-diagram">
<svg viewBox="0 0 720 280" role="img" aria-label="Anticorruption layer between a tangled legacy system and new microservices">
  <rect x="20" y="52" width="216" height="182" rx="12" fill="#101114" stroke="#f87171"/>
  <text x="128" y="38" text-anchor="middle" fill="#f87171" font-size="13" font-weight="600">Legacy system</text>
  <path d="M44,98 C104,72 150,138 212,106" fill="none" stroke="#8b90a0" stroke-width="1.6"/>
  <path d="M44,136 C112,178 148,80 212,148" fill="none" stroke="#8b90a0" stroke-width="1.6"/>
  <path d="M44,182 C98,128 158,206 212,178" fill="none" stroke="#8b90a0" stroke-width="1.6"/>
  <path d="M44,210 C120,196 140,110 212,214" fill="none" stroke="#8b90a0" stroke-width="1.6"/>
  <line x1="240" y1="143" x2="326" y2="143" stroke="#8b90a0" stroke-width="1.8" marker-end="url(#ag)"/>
  <rect x="336" y="60" width="20" height="166" rx="6" fill="#4ade8030" stroke="#4ade80" stroke-width="2"/>
  <text x="346" y="44" text-anchor="middle" fill="#4ade80" font-size="13" font-weight="600">Anticorruption layer</text>
  <line x1="364" y1="143" x2="444" y2="143" stroke="#4ade80" stroke-width="1.8" marker-end="url(#agr)"/>
  <text x="600" y="38" text-anchor="middle" fill="#60a5fa" font-size="13" font-weight="600">New microservices</text>
  <rect x="456" y="66" width="108" height="48" rx="8" fill="#101114" stroke="#60a5fa"/>
  <rect x="596" y="66" width="108" height="48" rx="8" fill="#101114" stroke="#60a5fa"/>
  <rect x="456" y="172" width="108" height="48" rx="8" fill="#101114" stroke="#60a5fa"/>
  <rect x="596" y="172" width="108" height="48" rx="8" fill="#101114" stroke="#60a5fa"/>
  <line x1="564" y1="90" x2="596" y2="90" stroke="#60a5fa" stroke-width="1.4"/>
  <line x1="564" y1="196" x2="596" y2="196" stroke="#60a5fa" stroke-width="1.4"/>
  <line x1="510" y1="114" x2="510" y2="172" stroke="#60a5fa" stroke-width="1.4"/>
  <line x1="650" y1="114" x2="650" y2="172" stroke="#60a5fa" stroke-width="1.4"/>
  <defs>
    <marker id="ag" markerWidth="9" markerHeight="9" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3 z" fill="#8b90a0"/></marker>
    <marker id="agr" markerWidth="9" markerHeight="9" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3 z" fill="#4ade80"/></marker>
  </defs>
</svg>
</div>

## Chapter 6. Guiding principles and routing

**The problem.** Making "irreversible" architectural decisions that turn into technical debt and block further development.

**Key ideas:**

- make decisions reversible;
- evolvability > predictability;
- build *sacrificial* architectures you will not regret throwing away.

<div class="svg-diagram">
<svg viewBox="0 0 720 280" role="img" aria-label="Dynamic routing sending eighty percent of traffic to V1 and twenty percent to a sacrificial V2">
  <rect x="16" y="110" width="136" height="60" rx="10" fill="#101114" stroke="#262a31"/>
  <text x="84" y="146" text-anchor="middle" fill="#e5e7eb" font-size="13">User traffic</text>
  <line x1="156" y1="140" x2="222" y2="140" stroke="#8b90a0" stroke-width="1.8" marker-end="url(#rg)"/>
  <rect x="228" y="104" width="168" height="72" rx="10" fill="#60a5fa18" stroke="#60a5fa" stroke-width="2"/>
  <text x="312" y="136" text-anchor="middle" fill="#e5e7eb" font-size="13">Proxy /</text>
  <text x="312" y="155" text-anchor="middle" fill="#e5e7eb" font-size="13">Discovery</text>
  <path d="M400,124 C462,124 468,72 524,72" fill="none" stroke="#60a5fa" stroke-width="2" marker-end="url(#rb)"/>
  <text x="470" y="60" text-anchor="middle" fill="#60a5fa" font-size="12">80% traffic</text>
  <rect x="532" y="42" width="172" height="62" rx="10" fill="#101114" stroke="#60a5fa"/>
  <text x="618" y="68" text-anchor="middle" fill="#e5e7eb" font-size="13" font-weight="600">V1</text>
  <text x="618" y="88" text-anchor="middle" fill="#8b90a0" font-size="11">current service</text>
  <path d="M400,158 C462,158 468,214 524,214" fill="none" stroke="#4ade80" stroke-width="2" stroke-dasharray="6 5" marker-end="url(#rgr)"/>
  <text x="470" y="240" text-anchor="middle" fill="#4ade80" font-size="12">20% traffic</text>
  <rect x="532" y="184" width="172" height="62" rx="10" fill="#4ade8012" stroke="#4ade80" stroke-dasharray="6 5"/>
  <text x="618" y="210" text-anchor="middle" fill="#e5e7eb" font-size="13" font-weight="600">V2</text>
  <text x="618" y="230" text-anchor="middle" fill="#8b90a0" font-size="11">sacrificial service</text>
  <defs>
    <marker id="rg" markerWidth="9" markerHeight="9" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3 z" fill="#8b90a0"/></marker>
    <marker id="rb" markerWidth="9" markerHeight="9" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3 z" fill="#60a5fa"/></marker>
    <marker id="rgr" markerWidth="9" markerHeight="9" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3 z" fill="#4ade80"/></marker>
  </defs>
</svg>
</div>

## Chapter 7. Threat board: antipatterns and traps

**The problem.** Hidden systemic dysfunctions kill flexibility at the root, even when the team sits on modern frameworks.

**Key idea.** Recognise and destroy toxic patterns before they take root in the engineering culture.

| Antipattern | What it is |
|---|---|
| Vendor King | total dependency on a proprietary vendor. Fix: introduce an abstraction layer |
| Leaky Abstractions | infrastructure implementation details seep into business logic |
| Resume-Driven Development | hype technology adopted for the architect's CV, needlessly complicating the quantum |

## Chapter 8. The Inverse Conway Maneuver

**The problem.** Coordination hell. The company structure conflicts with the target architecture, so every release needs sign-off across departments.

**Key ideas:**

- Conway's Law: architecture inevitably mirrors the communication structure of the organisation;
- Inverse Conway Maneuver: shape the teams exactly the way you want the target architecture to look.

<div class="svg-diagram">
<svg viewBox="0 0 760 380" role="img" aria-label="Siloed layer teams with tangled communication versus cross-functional product teams">
  <text x="180" y="36" text-anchor="middle" fill="#f87171" font-size="14" font-weight="600">Silos</text>
  <rect x="30" y="62" width="300" height="46" rx="8" fill="#101114" stroke="#60a5fa"/>
  <text x="180" y="91" text-anchor="middle" fill="#e5e7eb" font-size="13">UI team</text>
  <rect x="30" y="184" width="300" height="46" rx="8" fill="#101114" stroke="#60a5fa"/>
  <text x="180" y="213" text-anchor="middle" fill="#e5e7eb" font-size="13">Backend team</text>
  <rect x="30" y="306" width="300" height="46" rx="8" fill="#101114" stroke="#60a5fa"/>
  <text x="180" y="335" text-anchor="middle" fill="#e5e7eb" font-size="13">DBA team</text>
  <path d="M70,108 C100,140 250,152 300,184" fill="none" stroke="#f87171" stroke-width="1.4"/>
  <path d="M300,108 C250,150 100,140 60,184" fill="none" stroke="#f87171" stroke-width="1.4"/>
  <path d="M160,108 C200,142 120,150 220,184" fill="none" stroke="#fbbf24" stroke-width="1.4"/>
  <path d="M70,230 C110,264 250,272 300,306" fill="none" stroke="#f87171" stroke-width="1.4"/>
  <path d="M300,230 C250,270 100,266 60,306" fill="none" stroke="#f87171" stroke-width="1.4"/>
  <path d="M220,230 C160,268 240,276 140,306" fill="none" stroke="#fbbf24" stroke-width="1.4"/>
  <line x1="368" y1="40" x2="368" y2="356" stroke="#262a31" stroke-width="1.5"/>
  <text x="566" y="36" text-anchor="middle" fill="#4ade80" font-size="14" font-weight="600">Cross-functional</text>
  <rect x="404" y="56" width="104" height="300" rx="12" fill="#4ade800a" stroke="#4ade80"/>
  <text x="456" y="80" text-anchor="middle" fill="#4ade80" font-size="12">Product A</text>
  <rect x="418" y="96" width="76" height="40" rx="6" fill="#101114" stroke="#60a5fa"/>
  <text x="456" y="121" text-anchor="middle" fill="#e5e7eb" font-size="12">UI</text>
  <line x1="456" y1="136" x2="456" y2="186" stroke="#4ade80" stroke-width="1.6" marker-end="url(#cg)"/>
  <rect x="418" y="192" width="76" height="40" rx="6" fill="#101114" stroke="#60a5fa"/>
  <text x="456" y="217" text-anchor="middle" fill="#e5e7eb" font-size="11">Backend</text>
  <line x1="456" y1="232" x2="456" y2="282" stroke="#4ade80" stroke-width="1.6" marker-end="url(#cg)"/>
  <rect x="418" y="288" width="76" height="40" rx="6" fill="#101114" stroke="#60a5fa"/>
  <text x="456" y="313" text-anchor="middle" fill="#e5e7eb" font-size="12">DBA</text>
  <rect x="518" y="56" width="104" height="300" rx="12" fill="#4ade800a" stroke="#4ade80"/>
  <text x="570" y="80" text-anchor="middle" fill="#4ade80" font-size="12">Product B</text>
  <rect x="532" y="96" width="76" height="40" rx="6" fill="#101114" stroke="#60a5fa"/>
  <text x="570" y="121" text-anchor="middle" fill="#e5e7eb" font-size="12">UI</text>
  <line x1="570" y1="136" x2="570" y2="186" stroke="#4ade80" stroke-width="1.6" marker-end="url(#cg)"/>
  <rect x="532" y="192" width="76" height="40" rx="6" fill="#101114" stroke="#60a5fa"/>
  <text x="570" y="217" text-anchor="middle" fill="#e5e7eb" font-size="11">Backend</text>
  <line x1="570" y1="232" x2="570" y2="282" stroke="#4ade80" stroke-width="1.6" marker-end="url(#cg)"/>
  <rect x="532" y="288" width="76" height="40" rx="6" fill="#101114" stroke="#60a5fa"/>
  <text x="570" y="313" text-anchor="middle" fill="#e5e7eb" font-size="12">DBA</text>
  <rect x="632" y="56" width="104" height="300" rx="12" fill="#4ade800a" stroke="#4ade80"/>
  <text x="684" y="80" text-anchor="middle" fill="#4ade80" font-size="12">Product C</text>
  <rect x="646" y="96" width="76" height="40" rx="6" fill="#101114" stroke="#60a5fa"/>
  <text x="684" y="121" text-anchor="middle" fill="#e5e7eb" font-size="12">UI</text>
  <line x1="684" y1="136" x2="684" y2="186" stroke="#4ade80" stroke-width="1.6" marker-end="url(#cg)"/>
  <rect x="646" y="192" width="76" height="40" rx="6" fill="#101114" stroke="#60a5fa"/>
  <text x="684" y="217" text-anchor="middle" fill="#e5e7eb" font-size="11">Backend</text>
  <line x1="684" y1="232" x2="684" y2="282" stroke="#4ade80" stroke-width="1.6" marker-end="url(#cg)"/>
  <rect x="646" y="288" width="76" height="40" rx="6" fill="#101114" stroke="#60a5fa"/>
  <text x="684" y="313" text-anchor="middle" fill="#e5e7eb" font-size="12">DBA</text>
  <defs>
    <marker id="cg" markerWidth="9" markerHeight="9" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3 z" fill="#4ade80"/></marker>
  </defs>
</svg>
</div>

## Chapter 8. The economics of evolution: talking to the business

**The problem.** How do you justify to management — to the CFO — the money spent on engineering practices, refactoring and fitness functions?

**Key ideas:**

- a radical reduction in release risk;
- dramatically faster time-to-market;
- an unblocked culture of experimentation (kaizen).

<div class="svg-diagram">
<svg viewBox="0 0 700 340" role="img" aria-label="Cost per quantum falls while benefit rises; the crossing point is the sweet spot for investment">
  <line x1="76" y1="46" x2="76" y2="286" stroke="#8b90a0" stroke-width="1.5" marker-start="url(#sgu)"/>
  <line x1="76" y1="286" x2="646" y2="286" stroke="#8b90a0" stroke-width="1.5" marker-end="url(#sg)"/>
  <text x="360" y="316" text-anchor="middle" fill="#8b90a0" font-size="12"># of quanta</text>
  <text x="30" y="166" text-anchor="middle" fill="#8b90a0" font-size="12" transform="rotate(-90 30 166)">cost / benefit</text>
  <path d="M96,66 C200,196 300,238 626,264" fill="none" stroke="#f87171" stroke-width="2.5"/>
  <text x="596" y="248" text-anchor="end" fill="#f87171" font-size="12">cost / quantum</text>
  <path d="M96,268 C380,254 470,160 626,66" fill="none" stroke="#4ade80" stroke-width="2.5"/>
  <text x="600" y="58" text-anchor="end" fill="#4ade80" font-size="12">benefit</text>
  <circle cx="316" cy="236" r="9" fill="#0d0e12" stroke="#fbbf24" stroke-width="2.5"/>
  <line x1="316" y1="226" x2="316" y2="150" stroke="#fbbf24" stroke-width="1.3" stroke-dasharray="4 4"/>
  <text x="316" y="138" text-anchor="middle" fill="#fbbf24" font-size="13" font-weight="600">Sweet spot</text>
  <text x="316" y="120" text-anchor="middle" fill="#8b90a0" font-size="11">optimal investment</text>
  <defs>
    <marker id="sg" markerWidth="9" markerHeight="9" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3 z" fill="#8b90a0"/></marker>
    <marker id="sgu" markerWidth="9" markerHeight="9" refX="8" refY="3" orient="auto"><path d="M8,0 L8,6 L0,3 z" fill="#8b90a0"/></marker>
  </defs>
</svg>
</div>

## Migration strategies: where to start

**The problem.** Analysis paralysis when facing a huge, historically tangled legacy monolith.

**Key idea.** Choosing the right starting point is critical: it must either cut risk quickly or demonstrate maximum value to the business.

| Strategy | What it is |
|---|---|
| Low-Hanging Fruit | quick wins: easy to extract, low value |
| Highest-Value | maximum business benefit: harder, but proves ROI |
| Infrastructure-first | automate the pipelines before anything else |
| Testing-first | cover the legacy with fitness functions before refactoring |

## Final synthesis

**The problem.** Letting go of the "we will build this once, and build it to last" mindset.

**Key idea.** Evolutionary architecture is not a destination — it is a built-in survival mechanism. Three loops are braided into one: teams (Conway's Law), architecture (fitness functions and quanta), pipelines (incremental change).

<div class="svg-diagram">
<svg viewBox="0 0 720 300" role="img" aria-label="Teams, architecture and pipelines form one continuous reinforcing loop">
  <rect x="66" y="48" width="220" height="76" rx="12" fill="#60a5fa14" stroke="#60a5fa" stroke-width="2"/>
  <text x="176" y="80" text-anchor="middle" fill="#e5e7eb" font-size="14" font-weight="600">Teams</text>
  <text x="176" y="102" text-anchor="middle" fill="#8b90a0" font-size="11">Conway's Law</text>
  <rect x="424" y="48" width="230" height="76" rx="12" fill="#4ade8014" stroke="#4ade80" stroke-width="2"/>
  <text x="539" y="80" text-anchor="middle" fill="#e5e7eb" font-size="14" font-weight="600">Architecture</text>
  <text x="539" y="102" text-anchor="middle" fill="#8b90a0" font-size="11">fitness functions &amp; quanta</text>
  <rect x="245" y="192" width="230" height="76" rx="12" fill="#fbbf2414" stroke="#fbbf24" stroke-width="2"/>
  <text x="360" y="224" text-anchor="middle" fill="#e5e7eb" font-size="14" font-weight="600">Pipelines</text>
  <text x="360" y="246" text-anchor="middle" fill="#8b90a0" font-size="11">incremental change</text>
  <line x1="290" y1="86" x2="418" y2="86" stroke="#8b90a0" stroke-width="1.8" marker-end="url(#yg)"/>
  <path d="M560,128 C580,178 500,214 481,224" fill="none" stroke="#8b90a0" stroke-width="1.8" marker-end="url(#yg)"/>
  <path d="M239,224 C220,214 140,178 160,128" fill="none" stroke="#8b90a0" stroke-width="1.8" marker-end="url(#yg)"/>
  <defs>
    <marker id="yg" markerWidth="9" markerHeight="9" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L8,3 z" fill="#8b90a0"/></marker>
  </defs>
</svg>
</div>

> Stop building to last. Start building for change.
