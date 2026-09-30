# Part 85: Team Topology & CI/CD Organization

## บทนำ

วิธีที่ teams จัดโครงสร้างและ interact กันมีผลโดยตรงต่อคุณภาพและ architecture ของ software ที่พวกเขาสร้าง - นี่คือ Conway's Law ใน action บทนี้จะครอบคลุม Team Topologies framework และวิธีจัดโครงสร้าง teams เพื่อให้ CI/CD work ได้อย่างมีประสิทธิภาพ

## สารบัญ

1. [Conway's Law และ CI/CD](#conways-law)
2. [Team Topologies Framework](#team-topologies)
3. [Stream-Aligned Teams](#stream-aligned)
4. [Platform Teams](#platform-teams)
5. [Enabling Teams](#enabling-teams)
6. [Complicated Subsystem Teams](#complicated-subsystem)
7. [Interaction Modes](#interaction-modes)
8. [CI/CD Organization Design Patterns](#organization-patterns)
9. [Case Studies](#case-studies)
10. [แบบฝึกหัด](#exercises)

---

## 1. Conway's Law และ CI/CD {#conways-law}

### Conway's Law

> "Organizations which design systems are constrained to produce designs which are copies of the communication structures of those organizations."
> — Melvin Conway, 1967

```
ตัวอย่าง Conway's Law ใน action:

ถ้า Organization Structure แบบนี้:
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Frontend │  │ Backend  │  │   DBA    │
│  Team    │  │  Team    │  │  Team    │
└──────────┘  └──────────┘  └──────────┘

Architecture ที่ได้:
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Frontend │←→│ Backend  │←→│Database  │
│  Layer   │  │  Layer   │  │  Layer   │
└──────────┘  └──────────┘  └──────────┘

Consequence สำหรับ CI/CD:
- 3 pipelines แยกกัน
- Integration hell
- Slow deployments
- Blame culture
```

### Inverse Conway Maneuver

แทนที่จะ follow Conway's Law, เราสามารถ design organization เพื่อให้ได้ architecture ที่ต้องการ:

```
Target Architecture (Microservices):
Service A → Service B → Service C
(loosely coupled)

Design Organization accordingly:
Team A (Service A): 5 people
Team B (Service B): 5 people
Team C (Service C): 5 people

Each team owns their pipeline:
Team A: Build → Test → Deploy Service A
Team B: Build → Test → Deploy Service B
Team C: Build → Test → Deploy Service C

Result:
- Faster deployments
- Clear ownership
- Independent releases
```

### Reverse Conway Problems ใน CI/CD

ถ้า team structure ไม่ match CI/CD design:

```
Problem 1: Shared Pipeline Team
Organization: Teams A, B, C share one "DevOps Team"
Result: 
  - Bottleneck ที่ DevOps team
  - Teams ไม่รู้สึกเป็นเจ้าของ pipeline
  - Slow feature delivery

Problem 2: Silo-ed Testing
Organization: QA team แยกจาก Dev team
Result:
  - Testing เกิดหลัง development นาน
  - Integration issues พบช้า
  - CI/CD ไม่ work จริงๆ

Problem 3: Centralized Database Team  
Organization: DBA team ต้องอนุมัติทุก schema change
Result:
  - Database migrations เป็น bottleneck
  - Deployment ต้องรอ DBA approval
  - Slow velocity
```

---

## 2. Team Topologies Framework {#team-topologies}

### 4 Team Types

```
Team Topologies Framework (Matthew Skelton & Manuel Pais):

1. STREAM-ALIGNED TEAM
   - Aligned กับ flow of business/user value
   - End-to-end responsibility
   - "This is the primary team type"
   
2. PLATFORM TEAM
   - ให้ services ที่ stream-aligned teams ใช้
   - ลด cognitive load ของ stream-aligned teams
   
3. ENABLING TEAM
   - ช่วย stream-aligned teams รับมือกับ challenges
   - Temporary assistance, not permanent dependency
   
4. COMPLICATED SUBSYSTEM TEAM
   - ดูแล systems ที่ต้องการ specialist knowledge
   - Rocket science, payment processing, ML models
```

### ความสัมพันธ์ระหว่าง Team Types

```
                    ┌────────────────────────────┐
                    │     ENABLING TEAM           │
                    │  (CI/CD Best Practices)     │
                    │  ← Temporary assistance →   │
                    └────────────────────────────┘
                              │ helps
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
┌─────────────────┐  ┌──────────────┐  ┌──────────────┐
│  STREAM-ALIGNED │  │STREAM-ALIGNED│  │STREAM-ALIGNED│
│    TEAM A       │  │   TEAM B     │  │   TEAM C     │
│  (Payment)      │  │  (Orders)    │  │   (Users)    │
└────────┬────────┘  └──────┬───────┘  └──────┬───────┘
         │                  │                  │
         └──────────────────┼──────────────────┘
                            │ uses (X-as-a-Service)
                            ▼
                ┌────────────────────────┐
                │    PLATFORM TEAM        │
                │  (CI/CD Infrastructure) │
                │  - Build systems        │
                │  - Deploy tools         │
                │  - Monitoring           │
                └────────────────────────┘
```

---

## 3. Stream-Aligned Teams {#stream-aligned}

### ลักษณะของ Stream-Aligned Team ที่ดี

```
Stream-Aligned Team Characteristics:

Ownership:
✅ Own entire software lifecycle
✅ From code to production operation
✅ Including CI/CD pipeline configuration

Size (Two-Pizza Rule):
✅ 5-9 people (optimal)
✅ Small enough to communicate effectively
✅ Large enough to sustain output

Skills:
✅ Development (frontend/backend)
✅ Testing (automated)
✅ Operations (basic)
✅ Security awareness
```

### Stream-Aligned Team's CI/CD Responsibilities

```yaml
# Team's Pipeline Ownership Model

Stream-Aligned Team (Team Payment):
  OWNS:
    - pipeline configuration (.github/workflows/)
    - Test strategy and implementation
    - Deployment configuration
    - Service-specific security policies
    - Observability configuration (dashboards, alerts)
  
  USES (from Platform):
    - Runner infrastructure
    - Container registry
    - Secret management
    - Shared actions/templates
    - Monitoring infrastructure
  
  NOT RESPONSIBLE FOR:
    - Runner scaling
    - Registry maintenance  
    - Vault maintenance
    - Kubernetes cluster operations
```

### Team Cognitive Load

```
Cognitive Load ที่ Stream-Aligned Team ต้องรับมือ:

Intrinsic Cognitive Load (complexity ของ domain):
- Payment processing logic
- Business rules
- Regulatory requirements

Germane Cognitive Load (learning ที่มีคุณค่า):
- Domain expertise
- Software design patterns
- Testing strategies

Extraneous Cognitive Load (ควร minimize):
- How to configure Kubernetes
- How Vault works internally
- How to scale runners
- Certificate management

Platform Team's job = ลด Extraneous Cognitive Load
```

---

## 4. Platform Teams {#platform-teams}

### Platform Team Charter

```
Platform Team Charter:

Mission:
"Enable stream-aligned teams to deliver software 
 quickly, safely, and with confidence"

NOT:
"Provide and maintain CI/CD infrastructure"
(Too tool-focused, not outcome-focused)

Key Responsibilities:
1. Build and operate CI/CD platform
2. Define golden paths
3. Provide self-service capabilities
4. Set standards (collaboratively, not dictatorially)
5. Measure and improve developer experience

Success Metrics:
- Platform adoption rate > 80%
- Developer NPS > 40
- Mean time to deploy (new service): < 1 day
- P95 pipeline duration < 30 min
```

### Platform Team Structure

```
Platform Team (10-15 people):

Product Manager (1):
- Defines roadmap
- Understands developer needs
- Prioritizes features

Platform Engineering (6-8):
- Builds platform features
- Maintains infrastructure
- Security integration

Developer Experience (2-3):
- Documentation
- Tutorials
- Onboarding
- Developer advocacy

SRE/Operations (2):
- Platform reliability
- Incident response
- SLO management
```

### Platform Team Interaction Modes

```
Platform Team ↔ Stream-Aligned Team:

Primary Mode: X-as-a-Service
  - Platform provides tools/services
  - Team uses them self-service
  - Minimal human interaction needed
  - Examples: Runner pool, Registry, Templates

Secondary Mode: Collaboration
  - Used for complex migrations
  - New capability exploration
  - Usually time-bounded (1-2 sprints)

Avoid: Gatekeeper mode
  - Platform team approves every deployment
  - Creates bottleneck
  - Erodes team autonomy
```

---

## 5. Enabling Teams {#enabling-teams}

### เมื่อไหร่ต้องใช้ Enabling Team?

```
Scenarios ที่ต้องการ Enabling Team:

Scenario 1: Adoption of New Technology
"Organization ต้องการ adopt GitOps แต่ teams ไม่มี experience"

Enabling Team Role:
- Train teams (2-3 sprints)
- Create templates และ examples
- Provide consultation
- Fade out เมื่อ teams พร้อม

Scenario 2: Major Platform Migration
"Organization migrate จาก Jenkins ไป GitHub Actions"

Enabling Team Role:
- Understand current state
- Design migration path
- Pilot with 1-2 teams
- Document lessons learned
- Support remaining teams
```

### Enabling Team สำหรับ CI/CD

```
CI/CD Enabling Team Activities:

Phase 1: Assessment (2-4 weeks)
- Audit current state
- Identify patterns and antipatterns
- Define target state
- Create training curriculum

Phase 2: Training (4-8 weeks)
- Conduct workshops
- Create hands-on labs
- Pair with stream-aligned teams
- Create self-service learning resources

Phase 3: Transition (4-8 weeks)
- Migrate pilot teams
- Document patterns
- Create runbooks
- Transfer knowledge to Platform team

Phase 4: Fade out
- Team becomes independent
- Enabling team moves on
- Platform team supports ongoing
```

### Enabling Team Duration

```
ข้อสำคัญ: Enabling Team ต้องมีวันหมดอายุ

Maximum duration: 3-6 months per engagement

ถ้า Enabling Team ต้องอยู่นานกว่านั้น:
- ปัญหาอาจอยู่ที่ Platform ที่ยังยากเกินไป
- หรือ Stream-aligned teams ต้องการการสนับสนุนมากกว่านี้
- ควร reassess และ find root cause
```

---

## 6. Complicated Subsystem Teams {#complicated-subsystem}

### เมื่อไหร่ต้องใช้ Complicated Subsystem Team?

```
ใช้เมื่อ:
1. Component ต้องการ deep specialist knowledge
2. Knowledge ยาก/แพงที่จะ distribute ทุก team
3. ความผิดพลาดมีผลรุนแรงมาก

CI/CD Examples:
- Security scanning engine (custom implementation)
- Performance testing framework ซับซ้อน
- Build cache system ที่ optimize มาก
- Custom deployment orchestrator
```

### Complicated Subsystem Team ใน CI/CD

```
Example: Security Scanning Subsystem Team

Responsibilities:
- Maintain SAST/DAST scanning tools
- Tune rules for company's codebase
- Reduce false positives
- Integrate new vulnerability databases

Interface กับ Platform team:
- Provide scanning service (API/action)
- Define standards for scan results
- Support for complex vulnerabilities

Interface กับ Stream-aligned teams:
- X-as-a-service (simple scan action)
- Escalation for complex findings
```

---

## 7. Interaction Modes {#interaction-modes}

### 3 Interaction Modes

```
Team Topologies Interaction Modes:

1. COLLABORATION
   "Working closely together for a period of time"
   
   ✅ ใช้เมื่อ:
   - ค้นหา solutions ใหม่ๆ
   - Complex problems ที่ต้องการ combined expertise
   - เวลา migrate หรือ adopt new technology
   
   ❌ ไม่ใช้เมื่อ:
   - ต้องการความชัดเจนเรื่อง ownership
   - หรือ problem ถูก solved แล้ว
   
   Duration: Usually time-bounded (1-4 sprints)

2. X-AS-A-SERVICE
   "Consuming/providing something with minimal interaction"
   
   ✅ ใช้เมื่อ:
   - Service ถูก defined ชัดเจน
   - Interface ไม่เปลี่ยนบ่อย
   - Provider มี capacity
   
   ❌ ไม่ใช้เมื่อ:
   - Service ยังอยู่ใน exploration phase
   - หรือ high variability ใน requirements
   
   Duration: Long-term, steady state

3. FACILITATING
   "One team helps/guides another team"
   
   ✅ ใช้เมื่อ:
   - Team ต้องการ upskilling
   - Removing impediments
   
   ❌ ไม่ใช้เมื่อ:
   - Creates permanent dependency
   
   Duration: Time-bounded, until team is unblocked
```

### Interaction Mode สำหรับ CI/CD Context

```
CI/CD Interaction Mode Examples:

Platform Team → Stream-Aligned Team:

Mode: X-as-a-Service (80% of time)
  - Provide: GitHub Actions runners
  - Provide: Container registry
  - Provide: Pipeline templates
  - Provide: Secret management API
  Interface: self-service portal, documentation

Mode: Collaboration (15% of time)
  - Migrate legacy pipeline → new platform
  - Design new golden path
  - Complex performance investigation
  Duration: 1-3 sprints per engagement

Mode: Facilitating (5% of time)
  - Workshop: "Best practices for CI/CD"
  - "Office hours" support sessions
  - Documentation review sessions
```

---

## 8. CI/CD Organization Design Patterns {#organization-patterns}

### Pattern 1: Fully Federated

```
ลักษณะ:
- แต่ละ stream-aligned team ทำ CI/CD เอง
- ไม่มี central Platform team
- ใช้ shared tools แต่ไม่มี shared processes

เหมาะสำหรับ:
- Organizations ขนาดเล็ก (< 5 teams)
- High autonomy culture
- Diverse technology stacks

ข้อเสีย:
- Duplicated work
- Inconsistent practices
- High cognitive load per team
- Security gaps
```

### Pattern 2: Platform-Enabled

```
ลักษณะ:
- Platform team ให้ common services
- Stream-aligned teams use X-as-a-Service
- Clear boundaries between platform and application

เหมาะสำหรับ:
- Organizations ขนาดกลาง-ใหญ่ (10+ teams)
- Need standardization แต่ preserve autonomy

┌──────────────────────────────────────────┐
│         Platform Team                     │
│  Runner Pool │ Registry │ Templates       │
└──────────────────────────────────────────┘
     ↑ uses          ↑ uses          ↑ uses
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Team A  │     │ Team B  │     │ Team C  │
│ Owns own│     │ Owns own│     │ Owns own│
│ pipeline│     │ pipeline│     │ pipeline│
└─────────┘     └─────────┘     └─────────┘
```

### Pattern 3: Regulated Industry

```
ลักษณะ:
- Platform team จัดการ CI/CD สำหรับ compliance
- Security/Compliance checks mandatory
- Change management integration

เหมาะสำหรับ:
- ธนาคาร, healthcare, government
- SOX/PCI/HIPAA regulated companies

┌──────────────────────────────────────────┐
│    Compliance/Security Layer              │
│  (Cannot be bypassed)                    │
└──────────────────────────────────────────┘
           ↑ enforces
┌──────────────────────────────────────────┐
│         Platform Team                     │
│  Golden Path │ Templates │ Guardrails     │
└──────────────────────────────────────────┘
     ↑ uses          ↑ uses
┌─────────┐     ┌─────────┐
│ Team A  │     │ Team B  │
│ Limited │     │ Limited │
│ config  │     │ config  │
└─────────┘     └─────────┘
```

---

## 9. Case Studies {#case-studies}

### Case Study 1: Moving from Centralized DevOps to Team Topologies

**บริบท:**
- E-commerce company, 150 developers, 20 teams
- "DevOps team" 5 คน ดูแล CI/CD ทั้งหมด
- Bottleneck ที่ DevOps team

**ปัญหา:**
```
DevOps Team ไม่สามารถรับมือกับ demand:
- 50+ deployment requests/day
- Average wait time: 4 hours
- Developers frustrated
- DevOps team burned out
- 2 of 5 DevOps engineers quit ใน 6 เดือน
```

**การเปลี่ยนแปลง:**

```
Phase 1 (Month 1-3): Assessment
- Interview ทุก team
- Map current state
- Define target state

Phase 2 (Month 4-6): Platform Team Formation
- สร้าง Platform team (8 คน)
- ย้าย 3 คนจาก DevOps team
- Hire Platform PM, DX Engineer

Phase 3 (Month 7-12): Migration
- สร้าง self-service platform
- Migrate teams ทีละกลุ่ม
- Training และ enabling

Phase 4 (Month 13+): Steady State
- Teams deploy เอง
- Platform team focus on improvement
```

**ผลลัพธ์:**
```
Deployment wait time: 4 hours → 10 minutes
Deployment frequency: 5x/week → 3x/day
Developer satisfaction: 5.5/10 → 8.2/10
DevOps team turnover: 0 (vs 2 the prior year)
Monthly deployments: 100 → 1,500+
```

### Case Study 2: Enabling Team for GitOps Adoption

**บริบท:**
- 10 teams, 80 developers
- Legacy: Ansible + custom scripts
- Target: GitOps with ArgoCD

**Enabling Team Approach:**

```
Duration: 4 months

Month 1: Preparation
- Enabling team (3 people): 2 platform engineers + 1 developer advocate
- Study teams' current workflows
- Design GitOps templates
- Write comprehensive documentation

Month 2: Pilot
- Work with 2 volunteer teams
- Pair programming
- Daily office hours
- Iterate on templates

Month 3-4: Scale
- Migrate remaining 8 teams
- Each team: 1 week of guided migration
- Self-service after migration

Month 4 end: Fade out
- All teams GitOps-enabled
- Documentation complete
- Platform team takes over support
```

**ผลลัพธ์:**
```
Teams migrated: 10/10 (100%)
Average migration time: 4 days
Deployment errors: -60%
Rollback time: 2 hours → 5 minutes
Configuration drift incidents: 0 (vs 12/year before)
```

---

## 10. แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: Conway's Law Analysis

**สถานการณ์:** บริษัท E-commerce มี team structure ดังนี้:
- Frontend Team (8 คน)
- Backend Team (12 คน)
- DBA Team (4 คน)
- QA Team (6 คน)
- DevOps Team (3 คน)

**งาน:**
1. วาด current team communication structure
2. Predict architecture ที่จะได้ตาม Conway's Law
3. ระบุ CI/CD bottlenecks ที่น่าจะเกิดขึ้น
4. ออกแบบ team structure ใหม่ ตาม Team Topologies

### แบบฝึกหัดที่ 2: Platform Team Charter

**งาน:** เขียน Platform Team Charter สำหรับ:
- Organization: Fintech company
- Size: 200 developers, 20 stream-aligned teams
- Challenge: Slow deployments, security issues

**Charter ต้องครอบคลุม:**
1. Mission statement
2. Team responsibilities
3. Success metrics
4. Interaction modes กับ teams
5. Team composition

### แบบฝึกหัดที่ 3: Cognitive Load Assessment

**งาน:** ประเมิน cognitive load ของ typical stream-aligned team

**Activities:**
1. List tasks ที่ team ต้องทำใน 1 sprint
2. Categorize เป็น Intrinsic/Germane/Extraneous
3. ระบุ tasks ที่ Platform team ควรรับไป
4. คำนวณ % time saved ถ้า Platform จัดการ

### แบบฝึกหัดที่ 4: Interaction Mode Planning

**สถานการณ์:** Platform team ต้องการ onboard 5 new teams ไปยัง platform

**งาน:** ออกแบบ interaction plan:
1. Phase ต่างๆ และ interaction mode ที่เหมาะสม
2. Duration ของแต่ละ phase
3. Success criteria สำหรับ transition
4. Support model หลัง onboarding

```
Template:
Week 1-2: Mode = Collaboration
  Activities: ___
  Success criteria: ___

Week 3-4: Mode = Facilitating
  Activities: ___
  Success criteria: ___

Week 5+: Mode = X-as-a-Service
  Activities: ___
  Success criteria: ___
```

---

## สรุป

Team Topology และ CI/CD Organization มีความสัมพันธ์กันอย่างลึกซึ้ง:

1. **Conway's Law is real** - Team structure กำหนด pipeline structure
2. **Stream-Aligned Teams need ownership** - ของ pipeline ของตัวเอง end-to-end
3. **Platform Teams reduce cognitive load** - ไม่ใช่ control
4. **Interaction modes matter** - ใช้ผิด mode → ผล wrong outcomes
5. **Enabling Teams solve problems, then leave** - ไม่ใช่สร้าง permanent dependency

## อ่านเพิ่มเติม

- "Team Topologies" by Matthew Skelton & Manuel Pais
- "Accelerate" by Nicole Forsgren, Jez Humble, Gene Kim
- Team Topologies website: https://teamtopologies.com
- DORA State of DevOps Report: https://dora.dev/research

---

*Part 85 จาก 100 | CI/CD Mastery Course*
