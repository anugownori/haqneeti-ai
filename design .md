# Design Document: HAQNEETI - The Welfare Execution Protocol

## Overview

HAQNEETI is a voice-first execution intelligence system that models welfare schemes as a dependency graph and generates personalized, rejection-aware, step-by-step action plans for citizens. The system does not automate applications—it guides execution by understanding scheme dependencies, document requirements, deadlines, and common rejection patterns.

### Core Insight

India's welfare system fails not because schemes are missing, but because execution is complex. Schemes depend on other schemes, require documents that must match across departments, have order-sensitive applications, deadline windows, and hidden cascade effects. The system behaves like a dependency network but is presented as a static list. HAQNEETI models this complexity explicitly.

### Core Design Principles

1. **Execution Intelligence, Not Automation**: Guide citizens through complex dependencies, don't auto-apply
2. **Graph-Based Modeling**: Model schemes as a dependency graph with REQUIRES, ENABLES, MUST_PRECEDE, CONFLICTS_WITH relationships
3. **Rejection Prevention**: Pre-submission validation to catch document mismatches, expired certificates, missing prerequisites
4. **Voice-First Access**: Works on feature phones via IVR, no smartphone required
5. **Synthetic Testing**: Validate all logic with synthetic citizens before real deployment
6. **Realistic Scope**: Start with 2 states, 40-50 central schemes, 10-15 state schemes

### MVP Target Metrics

- Synthetic citizens for testing: 500-1000 profiles
- Languages: 2-3 initially (Hindi, Marathi, English)
- Scheme coverage: 40-50 central + 10-15 state schemes
- States: 2 states for MVP
- Rejection reduction target: 40% → 15% (realistic, not 8%)
- Demo safety: 100% synthetic, 0% PII

## Architecture

### Three Core Pillars

```
┌─────────────────────────────────────────────────────────────┐
│  PILLAR 1: Welfare Dependency Graph                         │
│  - Nodes: Scheme, Document, Deadline, Office, Rule          │
│  - Edges: REQUIRES, ENABLES, MUST_PRECEDE, CONFLICTS_WITH   │
│  - Captures cascades, order constraints, conflicts          │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  PILLAR 2: Execution Optimizer + Rejection Prevention       │
│  - Identify eligible schemes                                │
│  - Traverse cascade chains                                  │
│  - Resolve order dependencies                               │
│  - Pre-submission validation (name mismatch, expired docs)  │
│  - Week-by-week execution plan with risk flags              │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  PILLAR 3: Voice Execution Companion (DHARMA)               │
│  - Missed call → IVR callback                               │
│  - Extract profile via structured voice prompts            │
│  - Explain schemes in simple language                       │
│  - Guide through execution steps                            │
│  - Works on feature phones                                  │
└─────────────────────────────────────────────────────────────┘
```

### Supporting Components

```
┌─────────────────────────────────────────────────────────────┐
│  Synthetic Citizen Testing                                  │
│  - Generate 500-1000 synthetic profiles                     │
│  - Based on SECC/Census distributions                       │
│  - Validate graph logic and optimizer paths                 │
│  - Zero PII risk for demos                                  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Governance Dashboard (Decision Support)                    │
│  - Rejection pattern summaries                              │
│  - Document mismatch analysis                               │
│  - Cascade dependency visualization                         │
│  - Synthetic scenario testing                               │
└─────────────────────────────────────────────────────────────┘
```


### Technology Stack (MVP-Focused)

**Voice Processing:**
- Bhashini API: Government language platform (primary)
- Twilio/Exotel: IVR and voice call handling
- Simple TTS: Basic text-to-speech for voice responses
- No voice biometrics in MVP

**Dependency Graph:**
- Neo4j Community Edition OR in-memory graph (NetworkX for MVP)
- Simple REST API for graph queries
- No complex reasoning engine in MVP

**Backend:**
- Python FastAPI: Lightweight async API server
- SQLite/PostgreSQL: Simple relational storage
- No distributed task queues in MVP
- No MongoDB in MVP

**Validation Logic:**
- Rule-based validation (no ML in MVP)
- Pattern matching for document verification
- Simple heuristics for rejection risk

**Synthetic Data:**
- Faker + custom generators based on public SECC distributions
- CSV-based demographic data
- No complex ML-based generation

**Optional (Not MVP):**
- SHA-256 hash receipts (local fallback, no blockchain in MVP)
- WhatsApp integration (post-MVP)
- Gazette scraping (manual updates in MVP)

**Infrastructure:**
- Docker for containerization
- Single server deployment (no Kubernetes in MVP)
- Simple monitoring (logs + basic metrics)

## Components and Interfaces

### PILLAR 1: Welfare Dependency Graph

**Purpose:** Model welfare schemes as a structured dependency graph capturing cascades, order constraints, conflicts, and document dependencies.

**Graph Schema:**

```
Nodes:
- Scheme: {id, name, description, eligibility_rules, benefits, authority, processing_days}
- Document: {id, type, issuing_authority, validity_days, obtaining_process}
- Deadline: {id, scheme_id, deadline_date, type}  # APPLICATION, DOCUMENT_SUBMISSION
- Office: {id, name, location, schemes_handled}
- EligibilityRule: {id, scheme_id, rule_type, condition}

Relationships:
- (Scheme)-[REQUIRES]->(Document): Scheme needs document for application
- (Scheme)-[ENABLES]->(Scheme): Completing first scheme unlocks second (cascade)
- (Scheme)-[MUST_PRECEDE]->(Scheme): First scheme must be completed before second
- (Scheme)-[CONFLICTS_WITH]->(Scheme): Schemes are mutually exclusive
- (Scheme)-[HAS_DEADLINE]->(Deadline): Scheme has application/submission deadline
- (Document)-[EXPIRES_AFTER {days}]->(Document): Document validity period
- (Scheme)-[PROCESSED_AT]->(Office): Scheme applications processed at office
```

**Core Operations:**

```python
class WelfareDependencyGraph:
    def __init__(self, graph_backend: str = "networkx"):
        # Use NetworkX for MVP, can swap to Neo4j later
        if graph_backend == "networkx":
            self.graph = nx.DiGraph()
        else:
            self.graph = GraphDatabase.driver(neo4j_uri)
    
    def find_eligible_schemes(self, profile: CitizenProfile) -> List[Scheme]:
        """Find all schemes matching citizen's eligibility"""
        eligible = []
        for scheme in self.get_all_schemes():
            if self.evaluate_eligibility(scheme.eligibility_rules, profile):
                eligible.append(scheme)
        return eligible
    
    def find_cascade_chain(self, scheme_id: str) -> List[Scheme]:
        """Find all schemes enabled by completing given scheme"""
        # Traverse ENABLES edges
        enabled = []
        for edge in self.graph.out_edges(scheme_id):
            if edge.type == "ENABLES":
                enabled.append(self.get_scheme(edge.target))
                # Recursively find cascades
                enabled.extend(self.find_cascade_chain(edge.target))
        return enabled
    
    def check_order_constraints(self, scheme_ids: List[str]) -> List[Tuple[str, str]]:
        """Check if schemes have MUST_PRECEDE constraints"""
        constraints = []
        for s1 in scheme_ids:
            for s2 in scheme_ids:
                if self.has_edge(s1, s2, "MUST_PRECEDE"):
                    constraints.append((s1, s2))
        return constraints
    
    def check_conflicts(self, scheme_ids: List[str]) -> List[Tuple[str, str]]:
        """Identify conflicting schemes"""
        conflicts = []
        for s1 in scheme_ids:
            for s2 in scheme_ids:
                if self.has_edge(s1, s2, "CONFLICTS_WITH"):
                    conflicts.append((s1, s2))
        return conflicts
    
    def get_required_documents(self, scheme_ids: List[str]) -> List[Document]:
        """Get all documents required for a set of schemes"""
        docs = set()
        for scheme_id in scheme_ids:
            for edge in self.graph.out_edges(scheme_id):
                if edge.type == "REQUIRES":
                    docs.add(self.get_document(edge.target))
        return list(docs)
    
    def evaluate_eligibility(self, rules: Dict, profile: CitizenProfile) -> bool:
        """Simple rule evaluation (no complex inference)"""
        for rule_type, condition in rules.items():
            if rule_type == "age_range":
                if not (condition['min'] <= profile.age <= condition['max']):
                    return False
            elif rule_type == "income_level":
                if profile.income_level not in condition['allowed']:
                    return False
            elif rule_type == "caste_category":
                if profile.caste_category not in condition['allowed']:
                    return False
            # Add more rule types as needed
        return True
```

**Interface:**

```python
class WelfareDependencyGraphInterface:
    def find_eligible_schemes(self, profile: CitizenProfile) -> List[Scheme]
    def find_cascade_chain(self, scheme_id: str) -> List[Scheme]
    def check_order_constraints(self, scheme_ids: List[str]) -> List[Tuple[str, str]]
    def check_conflicts(self, scheme_ids: List[str]) -> List[Tuple[str, str]]
    def get_required_documents(self, scheme_ids: List[str]) -> List[Document]
```


### PILLAR 2: Execution Optimizer + Rejection Prevention

**Purpose:** Generate week-by-week execution plans with pre-submission validation to prevent common rejection causes.

**Algorithm:** Dependency-aware pathfinding with validation checks

```python
class ExecutionOptimizer:
    def __init__(self, dependency_graph: WelfareDependencyGraph):
        self.graph = dependency_graph
        self.validators = self.load_validators()
    
    def generate_execution_plan(self, profile: CitizenProfile) -> ExecutionPlan:
        # Step 1: Find eligible schemes
        eligible_schemes = self.graph.find_eligible_schemes(profile)
        
        if not eligible_schemes:
            return ExecutionPlan(schemes=[], message="No eligible schemes found")
        
        # Step 2: Identify cascades
        cascades = {}
        for scheme in eligible_schemes:
            cascades[scheme.id] = self.graph.find_cascade_chain(scheme.id)
        
        # Step 3: Resolve order constraints
        order_constraints = self.graph.check_order_constraints(
            [s.id for s in eligible_schemes]
        )
        
        # Step 4: Detect conflicts
        conflicts = self.graph.check_conflicts([s.id for s in eligible_schemes])
        
        # Step 5: Compute optimal sequence
        sequence = self.compute_optimal_sequence(
            eligible_schemes,
            cascades,
            order_constraints,
            conflicts
        )
        
        # Step 6: Generate week-by-week plan
        weekly_plan = self.generate_weekly_plan(sequence, profile)
        
        # Step 7: Run pre-submission validation
        validation_results = self.run_validation_checks(weekly_plan, profile)
        
        return ExecutionPlan(
            schemes=sequence,
            weekly_steps=weekly_plan,
            validation_results=validation_results,
            total_weeks=len(weekly_plan),
            cascade_opportunities=cascades
        )
    
    def compute_optimal_sequence(self, schemes: List[Scheme], 
                                  cascades: Dict,
                                  order_constraints: List[Tuple],
                                  conflicts: List[Tuple]) -> List[Scheme]:
        """
        Simple heuristic ordering:
        1. Schemes that enable most others come first
        2. Respect MUST_PRECEDE constraints
        3. Exclude conflicting schemes (pick higher benefit)
        4. Sort by processing time (faster first)
        """
        # Remove conflicting schemes
        schemes = self.resolve_conflicts(schemes, conflicts)
        
        # Topological sort based on MUST_PRECEDE
        ordered = self.topological_sort(schemes, order_constraints)
        
        # Prioritize cascade-enabling schemes
        ordered.sort(key=lambda s: len(cascades.get(s.id, [])), reverse=True)
        
        return ordered
    
    def generate_weekly_plan(self, schemes: List[Scheme], 
                            profile: CitizenProfile) -> List[WeeklyStep]:
        """Generate week-by-week execution steps"""
        plan = []
        current_week = 1
        
        for scheme in schemes:
            # Get required documents
            docs = self.graph.get_required_documents([scheme.id])
            
            # Identify missing documents
            missing_docs = [d for d in docs if d.type not in profile.documents_possessed]
            
            # Add document collection steps
            for doc in missing_docs:
                plan.append(WeeklyStep(
                    week=current_week,
                    action=f"Obtain {doc.type}",
                    location=doc.issuing_authority,
                    documents_needed=[],
                    estimated_time_days=7,
                    notes=doc.obtaining_process
                ))
                current_week += 1
            
            # Add scheme application step
            plan.append(WeeklyStep(
                week=current_week,
                action=f"Apply for {scheme.name}",
                location=self.graph.get_processing_office(scheme.id),
                documents_needed=[d.type for d in docs],
                estimated_time_days=scheme.processing_days,
                notes=f"Processing time: {scheme.processing_days} days"
            ))
            current_week += (scheme.processing_days // 7) + 1
        
        return plan
    
    def run_validation_checks(self, plan: List[WeeklyStep], 
                             profile: CitizenProfile) -> List[ValidationResult]:
        """Pre-submission validation to prevent rejections"""
        results = []
        
        # Check 1: Name consistency across documents
        name_check = self.validators['name_consistency'].validate(profile)
        if not name_check.passed:
            results.append(ValidationResult(
                check="Name Consistency",
                status="FAIL",
                severity="HIGH",
                message="Name mismatch detected across documents",
                suggested_action="Get name correction certificate from tehsil office"
            ))
        
        # Check 2: Expired documents
        for doc_type in profile.documents_possessed:
            expiry_check = self.validators['document_expiry'].validate(doc_type, profile)
            if not expiry_check.passed:
                results.append(ValidationResult(
                    check="Document Validity",
                    status="FAIL",
                    severity="MEDIUM",
                    message=f"{doc_type} has expired",
                    suggested_action=f"Renew {doc_type} before applying"
                ))
        
        # Check 3: Bank linkage
        bank_check = self.validators['bank_linkage'].validate(profile)
        if not bank_check.passed:
            results.append(ValidationResult(
                check="Bank Linkage",
                status="FAIL",
                severity="HIGH",
                message="Bank account not linked to Aadhaar",
                suggested_action="Visit bank to link Aadhaar with account"
            ))
        
        # Check 4: Deadline compression
        deadline_check = self.validators['deadline_risk'].validate(plan)
        if not deadline_check.passed:
            results.append(ValidationResult(
                check="Deadline Risk",
                status="WARNING",
                severity="MEDIUM",
                message="Timeline is tight for upcoming deadline",
                suggested_action="Prioritize document collection this week"
            ))
        
        return results
```

**Validation Checks:**

```python
class NameConsistencyValidator:
    def validate(self, profile: CitizenProfile) -> ValidationCheck:
        """Check if name is consistent across documents"""
        # Simple heuristic: check if name fields match
        # In real system, use fuzzy matching
        names = profile.get_names_from_documents()
        if len(set(names)) > 1:
            return ValidationCheck(passed=False, details=names)
        return ValidationCheck(passed=True)

class DocumentExpiryValidator:
    def validate(self, doc_type: str, profile: CitizenProfile) -> ValidationCheck:
        """Check if document is expired"""
        issue_date = profile.get_document_issue_date(doc_type)
        validity = self.get_validity_period(doc_type)
        if (datetime.now() - issue_date).days > validity:
            return ValidationCheck(passed=False, days_expired=(datetime.now() - issue_date).days - validity)
        return ValidationCheck(passed=True)

class BankLinkageValidator:
    def validate(self, profile: CitizenProfile) -> ValidationCheck:
        """Check if bank account is linked to Aadhaar"""
        # In real system, query NPCI API
        # For MVP, check profile attribute
        if not profile.bank_aadhaar_linked:
            return ValidationCheck(passed=False)
        return ValidationCheck(passed=True)

class DeadlineRiskValidator:
    def validate(self, plan: List[WeeklyStep]) -> ValidationCheck:
        """Check if plan can be completed before deadlines"""
        total_weeks = len(plan)
        # Check against known deadlines
        for step in plan:
            if hasattr(step, 'deadline'):
                weeks_until_deadline = (step.deadline - datetime.now()).days // 7
                if total_weeks > weeks_until_deadline:
                    return ValidationCheck(passed=False, weeks_short=total_weeks - weeks_until_deadline)
        return ValidationCheck(passed=True)
```

**Interface:**

```python
class ExecutionOptimizerInterface:
    def generate_execution_plan(self, profile: CitizenProfile) -> ExecutionPlan
    def validate_before_submission(self, profile: CitizenProfile, scheme_id: str) -> List[ValidationResult]
    def recalculate_after_completion(self, profile: CitizenProfile, completed_scheme: str) -> ExecutionPlan
```


### PILLAR 3: Voice Execution Companion (DHARMA)

**Purpose:** Provide voice-first access to execution plans via IVR, working on feature phones with structured prompts.

**Access Flow:**

```
Citizen gives missed call → System calls back → IVR menu → Profile extraction → Plan generation → Step-by-step guidance
```

**Core Implementation:**

```python
class VoiceExecutionCompanion:
    def __init__(self, ivr_client: IVRClient, 
                 optimizer: ExecutionOptimizer,
                 dependency_graph: WelfareDependencyGraph):
        self.ivr = ivr_client
        self.optimizer = optimizer
        self.graph = dependency_graph
        self.sessions = {}  # session_id -> SessionState
    
    def handle_missed_call(self, phone_number: str):
        """Handle missed call and initiate callback"""
        # Create session
        session_id = self.create_session(phone_number)
        
        # Call back user
        self.ivr.initiate_call(
            to=phone_number,
            callback_url=f"/ivr/session/{session_id}"
        )
    
    def handle_ivr_interaction(self, session_id: str, dtmf_input: str = None, 
                               voice_input: bytes = None) -> IVRResponse:
        """Handle IVR interaction (DTMF or voice)"""
        session = self.sessions[session_id]
        
        if session.state == "INITIAL":
            return self.prompt_language_selection(session)
        
        elif session.state == "LANGUAGE_SELECTED":
            return self.prompt_profile_extraction(session)
        
        elif session.state == "EXTRACTING_PROFILE":
            return self.extract_profile_step(session, voice_input)
        
        elif session.state == "PROFILE_COMPLETE":
            return self.generate_and_present_plan(session)
        
        elif session.state == "PLAN_PRESENTED":
            return self.guide_through_steps(session, dtmf_input)
        
        elif session.state == "GUIDING_STEPS":
            return self.continue_guidance(session, dtmf_input)
    
    def prompt_language_selection(self, session: SessionState) -> IVRResponse:
        """Prompt user to select language"""
        return IVRResponse(
            speech="Press 1 for Hindi, 2 for Marathi, 3 for English. " +
                   "हिंदी के लिए 1 दबाएं, मराठी के लिए 2, अंग्रेजी के लिए 3।",
            gather_input=True,
            num_digits=1,
            next_state="LANGUAGE_SELECTED"
        )
    
    def prompt_profile_extraction(self, session: SessionState) -> IVRResponse:
        """Start profile extraction with structured prompts"""
        lang = session.language
        
        if not session.profile.age:
            return IVRResponse(
                speech=self.localize("Please say your age in years.", lang),
                gather_voice=True,
                next_state="EXTRACTING_PROFILE"
            )
        elif not session.profile.family_size:
            return IVRResponse(
                speech=self.localize("How many people are in your family?", lang),
                gather_voice=True,
                next_state="EXTRACTING_PROFILE"
            )
        elif not session.profile.income_level:
            return IVRResponse(
                speech=self.localize(
                    "Do you have a BPL card? Say yes or no.", lang
                ),
                gather_voice=True,
                next_state="EXTRACTING_PROFILE"
            )
        elif not session.profile.location:
            return IVRResponse(
                speech=self.localize("Which district do you live in?", lang),
                gather_voice=True,
                next_state="EXTRACTING_PROFILE"
            )
        else:
            session.state = "PROFILE_COMPLETE"
            return self.generate_and_present_plan(session)
    
    def extract_profile_step(self, session: SessionState, 
                            voice_input: bytes) -> IVRResponse:
        """Extract one piece of profile information"""
        # Transcribe voice input
        transcript = self.transcribe(voice_input, session.language)
        
        # Extract entity based on current question
        if not session.profile.age:
            age = self.extract_age(transcript)
            session.profile.age = age
        elif not session.profile.family_size:
            family_size = self.extract_number(transcript)
            session.profile.family_size = family_size
        elif not session.profile.income_level:
            has_bpl = self.extract_yes_no(transcript)
            session.profile.income_level = "BPL" if has_bpl else "APL"
        elif not session.profile.location:
            district = self.extract_location(transcript)
            session.profile.district = district
        
        # Continue to next question
        return self.prompt_profile_extraction(session)
    
    def generate_and_present_plan(self, session: SessionState) -> IVRResponse:
        """Generate execution plan and present summary"""
        # Generate plan
        plan = self.optimizer.generate_execution_plan(session.profile)
        session.execution_plan = plan
        
        # Present summary
        lang = session.language
        speech = self.localize(
            f"I found {len(plan.schemes)} schemes for you. " +
            f"The complete plan will take {plan.total_weeks} weeks. " +
            f"Press 1 to hear the first step, press 2 to receive SMS summary, press 3 to end call.",
            lang
        )
        
        return IVRResponse(
            speech=speech,
            gather_input=True,
            num_digits=1,
            next_state="PLAN_PRESENTED"
        )
    
    def guide_through_steps(self, session: SessionState, 
                           dtmf_input: str) -> IVRResponse:
        """Guide user through execution steps"""
        if dtmf_input == "1":
            # Start from first step
            session.current_step = 0
            return self.narrate_step(session)
        elif dtmf_input == "2":
            # Send SMS summary
            self.send_sms_summary(session)
            return IVRResponse(
                speech=self.localize("SMS sent. Call again anytime to continue.", session.language),
                hangup=True
            )
        elif dtmf_input == "3":
            return IVRResponse(
                speech=self.localize("Thank you. Call again anytime.", session.language),
                hangup=True
            )
    
    def narrate_step(self, session: SessionState) -> IVRResponse:
        """Narrate current execution step"""
        step = session.execution_plan.weekly_steps[session.current_step]
        lang = session.language
        
        speech = self.localize(
            f"Week {step.week}: {step.action}. " +
            f"Go to {step.location}. " +
            f"Bring these documents: {', '.join(step.documents_needed)}. " +
            f"This will take about {step.estimated_time_days} days. " +
            f"Press 1 for next step, press 2 to repeat, press 3 to end call.",
            lang
        )
        
        return IVRResponse(
            speech=speech,
            gather_input=True,
            num_digits=1,
            next_state="GUIDING_STEPS"
        )
    
    def continue_guidance(self, session: SessionState, 
                         dtmf_input: str) -> IVRResponse:
        """Continue guiding through steps"""
        if dtmf_input == "1":
            # Next step
            session.current_step += 1
            if session.current_step >= len(session.execution_plan.weekly_steps):
                return IVRResponse(
                    speech=self.localize("That's all the steps. Good luck!", session.language),
                    hangup=True
                )
            return self.narrate_step(session)
        elif dtmf_input == "2":
            # Repeat current step
            return self.narrate_step(session)
        elif dtmf_input == "3":
            # End call
            return IVRResponse(
                speech=self.localize("Call again to continue from where you left off.", session.language),
                hangup=True
            )
    
    def send_sms_summary(self, session: SessionState):
        """Send SMS summary of execution plan"""
        plan = session.execution_plan
        lang = session.language
        
        sms = self.localize(f"Your welfare plan ({len(plan.schemes)} schemes):\n", lang)
        for i, step in enumerate(plan.weekly_steps[:5]):  # First 5 steps
            sms += f"{i+1}. Week {step.week}: {step.action}\n"
        
        if len(plan.weekly_steps) > 5:
            sms += f"...and {len(plan.weekly_steps) - 5} more steps. Call for details."
        
        self.ivr.send_sms(session.phone_number, sms)
    
    def transcribe(self, audio: bytes, language: str) -> str:
        """Transcribe voice input using Bhashini"""
        try:
            return self.bhashini.transcribe(audio, language)
        except:
            # Fallback: simple speech recognition
            return self.simple_transcribe(audio)
    
    def extract_age(self, transcript: str) -> int:
        """Extract age from transcript"""
        # Simple regex for numbers
        import re
        numbers = re.findall(r'\d+', transcript)
        if numbers:
            age = int(numbers[0])
            if 18 <= age <= 100:
                return age
        return None
    
    def extract_number(self, transcript: str) -> int:
        """Extract number from transcript"""
        import re
        numbers = re.findall(r'\d+', transcript)
        return int(numbers[0]) if numbers else None
    
    def extract_yes_no(self, transcript: str) -> bool:
        """Extract yes/no from transcript"""
        transcript_lower = transcript.lower()
        yes_words = ['yes', 'haan', 'ha', 'होय', 'हां']
        return any(word in transcript_lower for word in yes_words)
    
    def extract_location(self, transcript: str) -> str:
        """Extract district name from transcript"""
        # Simple matching against known districts
        for district in KNOWN_DISTRICTS:
            if district.lower() in transcript.lower():
                return district
        return None
    
    def localize(self, text: str, language: str) -> str:
        """Translate text to target language"""
        # Use translation dictionary or API
        if language == "hi":
            return self.translate_to_hindi(text)
        elif language == "mr":
            return self.translate_to_marathi(text)
        return text
```

**Interface:**

```python
class VoiceExecutionCompanionInterface:
    def handle_missed_call(self, phone_number: str)
    def handle_ivr_interaction(self, session_id: str, input: str) -> IVRResponse
    def send_sms_summary(self, phone_number: str, plan: ExecutionPlan)
    def resume_session(self, phone_number: str) -> SessionState
```


### Supporting Component: Synthetic Citizen Testing

**Purpose:** Generate synthetic citizen profiles for system validation without PII risk.

**Implementation:**

```python
class SyntheticCitizenGenerator:
    def __init__(self, secc_data_path: str, census_data_path: str):
        self.secc_distributions = self.load_secc_data(secc_data_path)
        self.census_data = self.load_census_data(census_data_path)
    
    def generate_batch(self, count: int, state: str = None, 
                      district: str = None) -> List[CitizenProfile]:
        """Generate batch of synthetic citizens"""
        profiles = []
        
        for i in range(count):
            profile = self.generate_single_profile(state, district)
            profiles.append(profile)
        
        return profiles
    
    def generate_single_profile(self, state: str = None, 
                               district: str = None) -> CitizenProfile:
        """Generate one synthetic citizen"""
        # Select state/district
        if not state:
            state = self.sample_state()
        if not district:
            district = self.sample_district(state)
        
        # Sample demographics from distributions
        age = self.sample_age_distribution(district)
        gender = self.sample_gender_ratio(district)
        caste = self.sample_caste_distribution(district)
        income = self.sample_income_distribution(district, caste)
        education = self.sample_education_level(district, income)
        occupation = self.sample_occupation(district, education, income)
        family_size = self.sample_family_size(district, income)
        
        # Generate document possession based on demographics
        documents = self.generate_document_set(income, education, caste, district)
        
        return CitizenProfile(
            id=f"SYNTH_{uuid.uuid4().hex[:8]}",
            age=age,
            gender=gender,
            caste_category=caste,
            income_level=income,
            family_size=family_size,
            state=state,
            district=district,
            occupation=occupation,
            education_level=education,
            documents_possessed=documents,
            is_synthetic=True,
            bank_aadhaar_linked=random.random() < 0.6  # 60% have linkage
        )
    
    def sample_from_distribution(self, distribution: Dict[str, float]) -> str:
        """Sample from probability distribution"""
        items = list(distribution.keys())
        probabilities = list(distribution.values())
        return random.choices(items, weights=probabilities)[0]
    
    def generate_document_set(self, income: str, education: str, 
                             caste: str, district: str) -> List[str]:
        """Generate realistic document possession"""
        docs = []
        
        # Aadhaar: 95% have it
        if random.random() < 0.95:
            docs.append("Aadhaar")
        
        # Ration card: 80% have it
        if random.random() < 0.80:
            card_type = "BPL_Card" if income == "BPL" else "APL_Card"
            docs.append(card_type)
        
        # Income certificate: varies by income level
        if income in ["BPL", "APL"] and random.random() < 0.50:
            docs.append("Income_Certificate")
        
        # Caste certificate: varies by caste
        if caste in ["SC", "ST", "OBC"] and random.random() < 0.60:
            docs.append("Caste_Certificate")
        
        # Bank account: 70% have it
        if random.random() < 0.70:
            docs.append("Bank_Passbook")
        
        # Education certificates: varies by education level
        if education in ["SECONDARY", "GRADUATE"] and random.random() < 0.70:
            docs.append("Education_Certificate")
        
        return docs
    
    def validate_distribution_match(self, profiles: List[CitizenProfile], 
                                   district: str) -> ValidationReport:
        """Validate that generated profiles match SECC distributions"""
        # Compute actual distributions from profiles
        actual_caste = self.compute_distribution(profiles, 'caste_category')
        actual_income = self.compute_distribution(profiles, 'income_level')
        
        # Get expected distributions from SECC
        expected_caste = self.secc_distributions[district]['caste']
        expected_income = self.secc_distributions[district]['income']
        
        # Chi-square test
        caste_chi2, caste_p = self.chi_square_test(actual_caste, expected_caste)
        income_chi2, income_p = self.chi_square_test(actual_income, expected_income)
        
        return ValidationReport(
            district=district,
            profile_count=len(profiles),
            caste_match=caste_p > 0.05,  # Not significantly different
            income_match=income_p > 0.05,
            caste_chi2=caste_chi2,
            income_chi2=income_chi2
        )
```

**Interface:**

```python
class SyntheticCitizenGeneratorInterface:
    def generate_batch(self, count: int, state: str = None, district: str = None) -> List[CitizenProfile]
    def validate_distribution_match(self, profiles: List[CitizenProfile], district: str) -> ValidationReport
    def export_for_demo(self, profile_id: str) -> CitizenProfile
```

### Supporting Component: Governance Dashboard

**Purpose:** Provide decision support for administrators with rejection pattern analysis and scenario testing.

**Implementation:**

```python
class GovernanceDashboard:
    def __init__(self, dependency_graph: WelfareDependencyGraph,
                 optimizer: ExecutionOptimizer,
                 synthetic_generator: SyntheticCitizenGenerator):
        self.graph = dependency_graph
        self.optimizer = optimizer
        self.synthetic_gen = synthetic_generator
    
    def get_rejection_patterns(self, district: str, 
                              time_period: str = "last_month") -> RejectionAnalysis:
        """Analyze rejection patterns (from pilot data or simulated)"""
        # In MVP, use simulated data
        # In production, query real application database
        
        rejections = self.query_rejections(district, time_period)
        
        # Group by reason
        reasons = {}
        for rejection in rejections:
            reason = rejection.reason
            if reason not in reasons:
                reasons[reason] = 0
            reasons[reason] += 1
        
        # Sort by frequency
        top_reasons = sorted(reasons.items(), key=lambda x: x[1], reverse=True)
        
        return RejectionAnalysis(
            district=district,
            time_period=time_period,
            total_rejections=len(rejections),
            top_reasons=top_reasons[:10],
            rejection_rate=len(rejections) / self.get_total_applications(district, time_period)
        )
    
    def get_document_mismatch_analysis(self, district: str) -> DocumentAnalysis:
        """Analyze common document mismatch types"""
        mismatches = self.query_document_mismatches(district)
        
        mismatch_types = {
            'name_mismatch': 0,
            'expired_document': 0,
            'missing_document': 0,
            'invalid_format': 0
        }
        
        for mismatch in mismatches:
            mismatch_types[mismatch.type] += 1
        
        return DocumentAnalysis(
            district=district,
            total_mismatches=len(mismatches),
            mismatch_breakdown=mismatch_types,
            most_common_mismatch=max(mismatch_types.items(), key=lambda x: x[1])[0]
        )
    
    def visualize_cascade_dependencies(self, scheme_id: str = None) -> CascadeVisualization:
        """Generate cascade dependency visualization"""
        if scheme_id:
            # Show cascades for specific scheme
            cascades = self.graph.find_cascade_chain(scheme_id)
            subgraph = self.graph.get_subgraph([scheme_id] + [s.id for s in cascades])
        else:
            # Show full dependency graph
            subgraph = self.graph.graph
        
        # Convert to visualization format (D3.js compatible)
        nodes = []
        edges = []
        
        for node in subgraph.nodes():
            nodes.append({
                'id': node,
                'label': self.graph.get_scheme(node).name,
                'type': 'scheme'
            })
        
        for edge in subgraph.edges():
            edges.append({
                'source': edge[0],
                'target': edge[1],
                'type': edge[2] if len(edge) > 2 else 'REQUIRES'
            })
        
        return CascadeVisualization(
            nodes=nodes,
            edges=edges,
            layout='hierarchical'
        )
    
    def simulate_new_scheme(self, new_scheme: Scheme, 
                           population_size: int = 1000,
                           district: str = None) -> SimulationResult:
        """Simulate impact of new scheme on synthetic population"""
        # Generate synthetic population
        population = self.synthetic_gen.generate_batch(population_size, district=district)
        
        # Add new scheme to graph (temporarily)
        self.graph.add_scheme_node(new_scheme)
        
        # Count eligible citizens
        eligible_count = 0
        for profile in population:
            if self.graph.evaluate_eligibility(new_scheme.eligibility_rules, profile):
                eligible_count += 1
        
        # Estimate cascades
        cascade_schemes = self.graph.find_cascade_chain(new_scheme.id)
        
        # Remove temporary scheme
        self.graph.remove_scheme_node(new_scheme.id)
        
        return SimulationResult(
            scheme_name=new_scheme.name,
            population_size=population_size,
            eligible_count=eligible_count,
            eligibility_rate=eligible_count / population_size,
            cascade_opportunities=len(cascade_schemes),
            estimated_benefit_value=eligible_count * new_scheme.benefit_value
        )
    
    def get_deadline_risk_clusters(self, district: str) -> DeadlineRiskAnalysis:
        """Identify schemes with deadline compression risk"""
        schemes = self.graph.get_schemes_by_district(district)
        
        risky_schemes = []
        for scheme in schemes:
            deadlines = self.graph.get_deadlines(scheme.id)
            for deadline in deadlines:
                days_until = (deadline.deadline_date - datetime.now()).days
                if days_until < 30:  # Less than 30 days
                    risky_schemes.append({
                        'scheme': scheme.name,
                        'deadline': deadline.deadline_date,
                        'days_remaining': days_until,
                        'risk_level': 'HIGH' if days_until < 14 else 'MEDIUM'
                    })
        
        return DeadlineRiskAnalysis(
            district=district,
            risky_schemes=risky_schemes,
            total_at_risk=len(risky_schemes)
        )
```

**Interface:**

```python
class GovernanceDashboardInterface:
    def get_rejection_patterns(self, district: str) -> RejectionAnalysis
    def get_document_mismatch_analysis(self, district: str) -> DocumentAnalysis
    def visualize_cascade_dependencies(self, scheme_id: str = None) -> CascadeVisualization
    def simulate_new_scheme(self, new_scheme: Scheme, population_size: int) -> SimulationResult
    def get_deadline_risk_clusters(self, district: str) -> DeadlineRiskAnalysis
```
    def __init__(self, model_path: str, historical_data: pd.DataFrame):
        self.models = self.load_ensemble(model_path)
        self.historical_data = historical_data
        self.geo_risk_cache = {}
    
    def assess_risk(self, profile: CitizenProfile, 
                   scheme: Scheme) -> RiskAssessment:
        # Feature engineering
        features = self.extract_features(profile, scheme)
        
        # Ensemble prediction
        risk_scores = [model.predict_proba(features)[0][1] 
                      for model in self.models]
        base_risk = np.mean(risk_scores)
        
        # Add geo-specific risk
        geo_risk = self.compute_geo_risk(profile.tehsil, scheme.id)
        
        # Add temporal risk (seasonal patterns)
        temporal_risk = self.compute_temporal_risk(scheme.id)
        
        # Combined risk
        total_risk = self.combine_risks(base_risk, geo_risk, temporal_risk)
        
        # Identify risk factors
        risk_factors = self.identify_risk_factors(features, scheme)
        
        # Generate mitigation suggestions
        mitigations = self.suggest_mitigations(risk_factors, profile)
        
        return RiskAssessment(
            overall_risk=total_risk,
            risk_level=self.categorize_risk(total_risk),
            risk_factors=risk_factors,
            mitigations=mitigations,
            geo_context=self.get_geo_context(profile.tehsil, scheme.id)
        )
    
    def compute_geo_risk(self, tehsil: str, scheme_id: str) -> float:
        """Calculate location-specific rejection risk"""
        cache_key = f"{tehsil}_{scheme_id}"
        
        if cache_key in self.geo_risk_cache:
            return self.geo_risk_cache[cache_key]
        
        # Query historical data for this location
        local_data = self.historical_data[
            (self.historical_data['tehsil'] == tehsil) &
            (self.historical_data['scheme_id'] == scheme_id)
        ]
        
        if len(local_data) < 10:
            # Insufficient data, use district or state average
            return self.get_broader_geo_risk(tehsil, scheme_id)
        
        rejection_rate = local_data['rejected'].mean()
        
        # Detect spikes (e.g., BPL application surge)
        recent_rate = local_data.tail(100)['rejected'].mean()
        if recent_rate > rejection_rate * 1.5:
            spike_penalty = 0.15
        else:
            spike_penalty = 0
        
        geo_risk = rejection_rate + spike_penalty
        self.geo_risk_cache[cache_key] = geo_risk
        
        return geo_risk
    
    def identify_risk_factors(self, features: np.ndarray, 
                             scheme: Scheme) -> List[RiskFactor]:
        """Use SHAP values to identify top risk contributors"""
        shap_values = self.compute_shap_values(features)
        
        risk_factors = []
        for i, (feature_name, shap_value) in enumerate(
            sorted(zip(self.feature_names, shap_values), 
                   key=lambda x: abs(x[1]), reverse=True)[:5]
        ):
            if shap_value > 0.05:  # Significant positive contribution to risk
                risk_factors.append(RiskFactor(
                    factor=feature_name,
                    impact=shap_value,
                    description=self.explain_factor(feature_name, scheme)
                ))
        
        return risk_factors
    
    def suggest_mitigations(self, risk_factors: List[RiskFactor], 
                           profile: CitizenProfile) -> List[str]:
        """Generate actionable mitigation suggestions"""
        suggestions = []
        
        for factor in risk_factors:
            if "document" in factor.factor.lower():
                doc_type = self.extract_document_type(factor.factor)
                suggestions.append(
                    f"Obtain {doc_type} before applying. "
                    f"Visit {self.get_issuing_office(doc_type, profile.tehsil)}"
                )
            elif "income" in factor.factor.lower():
                suggestions.append(
                    "Ensure income certificate is recent (within 6 months) "
                    "and matches scheme threshold"
                )
            elif "timing" in factor.factor.lower():
                suggestions.append(
                    f"Consider applying in {self.get_optimal_month(factor.scheme_id)} "
                    "when approval rates are higher"
                )
        
        return suggestions
    
    def generate_alert(self, risk: RiskAssessment, language: str) -> Alert:
        """Generate real-time alert for high-risk applications"""
        if risk.overall_risk < 0.3:
            return None
        
        alert_text = self.localize_alert(
            f"⚠️ Rejection risk: {risk.overall_risk*100:.0f}%\n"
            f"Main issues: {', '.join([f.factor for f in risk.risk_factors[:3]])}\n"
            f"Suggestions: {risk.mitigations[0]}",
            language
        )
        
        return Alert(
            severity="HIGH" if risk.overall_risk > 0.5 else "MEDIUM",
            message=alert_text,
            risk_score=risk.overall_risk
        )
```

**Interface:**

```python
class RejectionRiskEngineInterface:
    def assess_application_risk(self, profile: CitizenProfile, 
                               scheme: Scheme) -> RiskAssessment
    def get_geo_statistics(self, tehsil: str, scheme_id: str) -> GeoStats
    def track_risk_reduction(self, baseline: float, current: float) -> RiskMetrics
```


### 5. Predictive Governance Engine

**Purpose:** Monitor policy changes through gazette scraping and simulate impact on citizen populations.

```python
class PredictiveGovernanceEngine:
    def __init__(self, gazette_sources: List[str], 
                 knowledge_graph: KnowledgeGraph):
        self.sources = gazette_sources
        self.kg = knowledge_graph
        self.scraper = GazetteScraper()
        self.nlp_extractor = PolicyExtractor()
    
    def scrape_gazettes(self) -> List[PolicyChange]:
        """Daily scraping of official gazette publications"""
        changes = []
        
        for source in self.sources:
            # Scrape gazette website
            documents = self.scraper.fetch_recent(source, days=1)
            
            for doc in documents:
                # Extract policy changes using NLP
                extracted = self.nlp_extractor.extract_changes(doc)
                
                for change in extracted:
                    changes.append(PolicyChange(
                        source=source,
                        date=doc.publication_date,
                        type=change.type,
                        scheme_id=change.scheme_id,
                        description=change.description,
                        effective_date=change.effective_date,
                        raw_text=change.raw_text
                    ))
        
        return changes
    
    def process_policy_change(self, change: PolicyChange):
        """Update knowledge graph and alert affected citizens"""
        # Update knowledge graph
        self.kg.update_from_gazette(change)
        
        # Identify affected citizens
        affected = self.identify_affected_citizens(change)
        
        # Generate alerts
        for citizen_id in affected:
            alert = self.generate_policy_alert(citizen_id, change)
            self.send_alert(citizen_id, alert)
    
    def simulate_policy_impact(self, policy_change: PolicyChange, 
                              population: List[CitizenProfile]) -> ImpactReport:
        """Collector simulator: what-if analysis on synthetic population"""
        
        # Apply policy change to synthetic population
        before_eligible = [p for p in population 
                          if self.is_eligible(p, policy_change.scheme_id)]
        
        # Simulate new eligibility rules
        after_eligible = [p for p in population 
                         if self.is_eligible_after_change(p, policy_change)]
        
        # Calculate impact metrics
        beneficiary_change = len(after_eligible) - len(before_eligible)
        beneficiary_change_pct = (beneficiary_change / len(before_eligible) * 100 
                                 if before_eligible else 0)
        
        # Estimate rejection rate impact
        before_risk = np.mean([self.risk_engine.assess_risk(p, policy_change.scheme_id).overall_risk 
                              for p in before_eligible[:1000]])
        after_risk = np.mean([self.risk_engine.assess_risk(p, policy_change.scheme_id).overall_risk 
                             for p in after_eligible[:1000]])
        
        # Calculate ROI
        cost_per_beneficiary = policy_change.estimated_cost / max(len(after_eligible), 1)
        benefit_per_beneficiary = self.kg.get_scheme(policy_change.scheme_id).benefit_value
        roi = (benefit_per_beneficiary - cost_per_beneficiary) / cost_per_beneficiary
        
        return ImpactReport(
            policy_change=policy_change,
            beneficiaries_before=len(before_eligible),
            beneficiaries_after=len(after_eligible),
            beneficiary_change=beneficiary_change,
            beneficiary_change_pct=beneficiary_change_pct,
            rejection_risk_before=before_risk,
            rejection_risk_after=after_risk,
            estimated_roi=roi,
            demographic_breakdown=self.analyze_demographics(after_eligible)
        )
    
    def collector_dashboard(self, district: str) -> DashboardData:
        """Generate collector dashboard with policy simulation tools"""
        return DashboardData(
            active_schemes=self.kg.get_schemes_by_district(district),
            recent_policy_changes=self.get_recent_changes(district),
            simulation_tool=self.get_simulation_interface(),
            beneficiary_stats=self.get_beneficiary_stats(district),
            rejection_trends=self.get_rejection_trends(district)
        )
```

**Gazette Scraping Strategy:**

```python
class GazetteScraper:
    def __init__(self):
        self.sources = {
            'central': 'https://egazette.nic.in/',
            'state_gazettes': {
                'maharashtra': 'https://www.maharashtra.gov.in/gazette',
                # ... other states
            }
        }
    
    def fetch_recent(self, source: str, days: int = 1) -> List[Document]:
        """Fetch recent gazette publications"""
        driver = webdriver.Chrome()
        documents = []
        
        try:
            driver.get(source)
            # Navigate to recent publications
            date_filter = driver.find_element(By.ID, "date_filter")
            date_filter.send_keys(self.format_date(days_ago=days))
            
            # Extract PDF links
            pdf_links = driver.find_elements(By.CSS_SELECTOR, "a[href$='.pdf']")
            
            for link in pdf_links:
                pdf_url = link.get_attribute('href')
                pdf_content = self.download_pdf(pdf_url)
                text = self.extract_text_from_pdf(pdf_content)
                
                documents.append(Document(
                    url=pdf_url,
                    publication_date=self.extract_date(link.text),
                    content=text
                ))
        finally:
            driver.quit()
        
        return documents
```

**Interface:**

```python
class PredictiveGovernanceInterface:
    def scrape_and_process_gazettes(self) -> List[PolicyChange]
    def simulate_policy(self, change: PolicyChange, 
                       population: List[CitizenProfile]) -> ImpactReport
    def get_collector_dashboard(self, district: str) -> DashboardData
    def alert_policy_changes(self, scheme_id: str) -> List[Alert]
```


### 6. Trust Anchor (Blockchain)

**Purpose:** Provide tamper-proof execution proofs using Polygon blockchain for transparency and grievance support.

```python
class TrustAnchor:
    def __init__(self, polygon_rpc: str, contract_address: str):
        self.w3 = Web3(Web3.HTTPProvider(polygon_rpc))
        self.contract = self.w3.eth.contract(
            address=contract_address,
            abi=self.load_abi()
        )
        self.ipfs_client = ipfshttpclient.connect()
    
    def timestamp_action(self, citizen_id: str, action: Action) -> BlockchainReceipt:
        """Timestamp citizen action on Polygon blockchain"""
        
        # Create execution proof metadata
        proof_metadata = {
            'citizen_id_hash': self.hash_citizen_id(citizen_id),  # Privacy: hash, not raw ID
            'action_type': action.type,
            'scheme_id': action.scheme_id,
            'timestamp': action.timestamp.isoformat(),
            'status': action.status,
            'location': action.location
        }
        
        # Store metadata on IPFS
        ipfs_hash = self.ipfs_client.add_json(proof_metadata)
        
        # Timestamp IPFS hash on blockchain
        tx_hash = self.contract.functions.timestampProof(
            ipfs_hash,
            self.w3.keccak(text=citizen_id)  # Hashed citizen ID
        ).transact({'from': self.w3.eth.default_account})
        
        # Wait for confirmation
        receipt = self.w3.eth.wait_for_transaction_receipt(tx_hash)
        
        return BlockchainReceipt(
            transaction_hash=receipt['transactionHash'].hex(),
            block_number=receipt['blockNumber'],
            ipfs_hash=ipfs_hash,
            timestamp=datetime.now(),
            gas_used=receipt['gasUsed']
        )
    
    def verify_proof(self, transaction_hash: str) -> VerificationResult:
        """Verify execution proof from blockchain"""
        
        # Get transaction from blockchain
        tx = self.w3.eth.get_transaction(transaction_hash)
        receipt = self.w3.eth.get_transaction_receipt(transaction_hash)
        
        # Decode contract call
        decoded = self.contract.decode_function_input(tx['input'])
        ipfs_hash = decoded[1]['ipfsHash']
        
        # Retrieve metadata from IPFS
        metadata = self.ipfs_client.get_json(ipfs_hash)
        
        return VerificationResult(
            valid=True,
            metadata=metadata,
            block_number=receipt['blockNumber'],
            timestamp=self.get_block_timestamp(receipt['blockNumber']),
            confirmations=self.w3.eth.block_number - receipt['blockNumber']
        )
    
    def generate_grievance_package(self, citizen_id: str, 
                                   scheme_id: str) -> GrievancePackage:
        """Generate grievance-ready documentation with blockchain proofs"""
        
        # Get all actions for this citizen and scheme
        actions = self.get_citizen_actions(citizen_id, scheme_id)
        
        # Collect blockchain receipts
        receipts = [self.verify_proof(action.tx_hash) for action in actions]
        
        # Generate timeline
        timeline = self.generate_timeline(actions, receipts)
        
        # Create PDF package
        pdf = self.create_grievance_pdf(
            citizen_id=citizen_id,
            scheme_id=scheme_id,
            timeline=timeline,
            receipts=receipts
        )
        
        return GrievancePackage(
            pdf_content=pdf,
            blockchain_receipts=receipts,
            timeline=timeline,
            rti_ready=True
        )
    
    def integrate_with_rti(self, rti_request: RTIRequest) -> RTIResponse:
        """Integrate with RTI request systems"""
        
        # Fetch relevant blockchain proofs
        proofs = self.fetch_proofs_for_request(rti_request)
        
        # Generate response with verifiable data
        return RTIResponse(
            request_id=rti_request.id,
            proofs=proofs,
            verification_instructions=self.get_verification_guide(),
            blockchain_explorer_links=[
                f"https://polygonscan.com/tx/{proof.transaction_hash}"
                for proof in proofs
            ]
        )
```

**Smart Contract (Solidity):**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract HAQNEETITrustAnchor {
    struct ExecutionProof {
        bytes32 citizenIdHash;
        string ipfsHash;
        uint256 timestamp;
        address submitter;
    }
    
    mapping(bytes32 => ExecutionProof[]) public citizenProofs;
    
    event ProofTimestamped(
        bytes32 indexed citizenIdHash,
        string ipfsHash,
        uint256 timestamp
    );
    
    function timestampProof(string memory ipfsHash, bytes32 citizenIdHash) 
        public returns (uint256) {
        
        ExecutionProof memory proof = ExecutionProof({
            citizenIdHash: citizenIdHash,
            ipfsHash: ipfsHash,
            timestamp: block.timestamp,
            submitter: msg.sender
        });
        
        citizenProofs[citizenIdHash].push(proof);
        
        emit ProofTimestamped(citizenIdHash, ipfsHash, block.timestamp);
        
        return citizenProofs[citizenIdHash].length - 1;
    }
    
    function getProofCount(bytes32 citizenIdHash) 
        public view returns (uint256) {
        return citizenProofs[citizenIdHash].length;
    }
    
    function getProof(bytes32 citizenIdHash, uint256 index) 
        public view returns (ExecutionProof memory) {
        require(index < citizenProofs[citizenIdHash].length, "Invalid index");
        return citizenProofs[citizenIdHash][index];
    }
}
```

**Interface:**

```python
class TrustAnchorInterface:
    def timestamp_action(self, citizen_id: str, action: Action) -> BlockchainReceipt
    def verify_proof(self, transaction_hash: str) -> VerificationResult
    def generate_grievance_package(self, citizen_id: str, scheme_id: str) -> GrievancePackage
    def bulk_verify(self, transaction_hashes: List[str]) -> List[VerificationResult]
```


### 7. Voice Interface Layer

**Purpose:** Provide complete voice-first interaction supporting 12 Indian languages.

```python
class VoiceInterface:
    def __init__(self, bhashini_client: BhashiniClient, 
                 tts_engine: TTSEngine,
                 entity_extractor: VoiceEntityExtractor):
        self.bhashini = bhashini_client
        self.tts = tts_engine
        self.extractor = entity_extractor
        self.session_manager = SessionManager()
    
    def handle_voice_input(self, audio: bytes, session_id: str, 
                          language: str) -> VoiceResponse:
        """Process voice input and generate voice response"""
        
        session = self.session_manager.get_session(session_id)
        
        # Extract entities from voice
        entities = self.extractor.extract_from_voice(audio, language)
        
        # Update session context
        session.update_context(entities)
        
        # Determine intent
        intent = self.classify_intent(entities.raw_transcript, language)
        
        # Route to appropriate handler
        if intent == "QUERY_SCHEMES":
            response_text = self.handle_scheme_query(session, entities)
        elif intent == "CHECK_ELIGIBILITY":
            response_text = self.handle_eligibility_check(session, entities)
        elif intent == "GET_ROADMAP":
            response_text = self.handle_roadmap_request(session, entities)
        elif intent == "CHECK_RISK":
            response_text = self.handle_risk_check(session, entities)
        else:
            response_text = self.handle_clarification(session, entities)
        
        # Generate voice response
        audio_response = self.tts.synthesize(response_text, language)
        
        return VoiceResponse(
            audio=audio_response,
            text=response_text,
            intent=intent,
            confidence=entities.confidence,
            session_id=session_id
        )
    
    def handle_scheme_query(self, session: Session, 
                           entities: ExtractedEntities) -> str:
        """Handle scheme information queries"""
        
        # Build citizen profile from session
        profile = session.get_citizen_profile()
        
        # Query knowledge graph
        eligible_schemes = self.kg.find_eligible_schemes(profile)
        
        # Generate natural language response
        if not eligible_schemes:
            return self.localize(
                "Based on your information, I couldn't find eligible schemes. "
                "Let me ask a few more questions to help you better.",
                session.language
            )
        
        response = self.localize(
            f"I found {len(eligible_schemes)} schemes you may be eligible for. ",
            session.language
        )
        
        # List top 3 schemes
        for i, scheme in enumerate(eligible_schemes[:3]):
            response += self.localize(
                f"{i+1}. {scheme.name}: {scheme.description}. ",
                session.language
            )
        
        response += self.localize(
            "Would you like me to create an application roadmap for you?",
            session.language
        )
        
        return response
    
    def handle_roadmap_request(self, session: Session, 
                               entities: ExtractedEntities) -> str:
        """Generate and narrate optimal application roadmap"""
        
        profile = session.get_citizen_profile()
        
        # Compute optimal strategy
        strategy = self.strategy_optimizer.optimize_path(profile)
        
        # Generate voice roadmap
        roadmap = self.strategy_optimizer.generate_voice_roadmap(
            strategy, session.language
        )
        
        # Narrate roadmap
        response = self.localize(
            f"I've created an optimal roadmap with {len(strategy.sequence)} schemes. ",
            session.language
        )
        
        for step in roadmap.steps[:3]:  # First 3 steps
            response += self.localize(
                f"Step {step.number}: Apply for {step.scheme_name}. "
                f"You'll need {', '.join(step.documents_needed)}. "
                f"This takes about {step.estimated_time} days. ",
                session.language
            )
            
            if step.unlocks:
                response += self.localize(
                    f"Completing this will unlock {step.unlocks}. ",
                    session.language
                )
        
        response += self.localize(
            "I'm sending the complete roadmap to your WhatsApp. "
            "Shall I also check your rejection risk?",
            session.language
        )
        
        # Send WhatsApp message
        self.send_whatsapp_roadmap(session.phone_number, strategy, session.language)
        
        return response
    
    def handle_risk_check(self, session: Session, 
                         entities: ExtractedEntities) -> str:
        """Check and narrate rejection risk"""
        
        profile = session.get_citizen_profile()
        scheme_id = entities.get('scheme_id') or session.get_current_scheme()
        
        # Assess risk
        risk = self.risk_engine.assess_risk(profile, scheme_id)
        
        # Generate alert if needed
        alert = self.risk_engine.generate_alert(risk, session.language)
        
        if risk.overall_risk < 0.3:
            response = self.localize(
                f"Good news! Your rejection risk is low at {risk.overall_risk*100:.0f}%. "
                "You have a strong chance of approval.",
                session.language
            )
        else:
            response = self.localize(
                f"Your rejection risk is {risk.overall_risk*100:.0f}%. ",
                session.language
            )
            
            # Explain top risk factor
            if risk.risk_factors:
                top_factor = risk.risk_factors[0]
                response += self.localize(
                    f"The main issue is: {top_factor.description}. ",
                    session.language
                )
            
            # Provide mitigation
            if risk.mitigations:
                response += self.localize(
                    f"Here's what you can do: {risk.mitigations[0]}",
                    session.language
                )
        
        return response
    
    def adapt_to_comprehension(self, session: Session, 
                              response_text: str) -> str:
        """Adapt speech rate and complexity based on user comprehension"""
        
        # Track user's response times and clarification requests
        comprehension_score = session.get_comprehension_score()
        
        if comprehension_score < 0.5:
            # Simplify language and slow down
            response_text = self.simplify_text(response_text, session.language)
            self.tts.set_speech_rate(0.8)  # 80% of normal speed
        else:
            self.tts.set_speech_rate(1.0)
        
        return response_text
```

**WhatsApp Integration:**

```python
class WhatsAppInterface:
    def __init__(self, business_api_token: str):
        self.client = WhatsAppBusinessClient(business_api_token)
    
    def send_roadmap(self, phone: str, strategy: OptimalStrategy, 
                    language: str):
        """Send formatted roadmap via WhatsApp"""
        
        message = self.strategy_optimizer.generate_whatsapp_message(
            strategy, language
        )
        
        self.client.send_message(
            to=phone,
            message=message,
            language=language
        )
    
    def handle_voice_note(self, phone: str, audio_url: str, language: str):
        """Handle asynchronous voice notes"""
        
        # Download audio
        audio = requests.get(audio_url).content
        
        # Process voice input
        response = self.voice_interface.handle_voice_input(
            audio, 
            session_id=f"whatsapp_{phone}",
            language=language
        )
        
        # Send text response (WhatsApp doesn't support voice responses well)
        self.client.send_message(
            to=phone,
            message=response.text,
            language=language
        )
```

**Interface:**

```python
class VoiceInterfaceAPI:
    def process_voice(self, audio: bytes, session_id: str, 
                     language: str) -> VoiceResponse
    def send_whatsapp_roadmap(self, phone: str, strategy: OptimalStrategy, 
                             language: str)
    def handle_ivr_call(self, call_id: str, dtmf_input: str) -> IVRResponse
```


### 8. Demo System

**Purpose:** Showcase system capabilities using synthetic citizens with zero PII exposure.

```python
class DemoSystem:
    def __init__(self, sct_generator: SyntheticCitizenGenerator,
                 voice_interface: VoiceInterface,
                 knowledge_graph: KnowledgeGraph):
        self.sct = sct_generator
        self.voice = voice_interface
        self.kg = knowledge_graph
        self.demo_profiles = self.load_demo_profiles()
    
    def run_demo_flow(self, profile_name: str = "Priya") -> DemoExecution:
        """Execute 5-minute demo flow with synthetic citizen"""
        
        # Load pre-configured synthetic profile
        profile = self.demo_profiles[profile_name]
        
        demo = DemoExecution(profile=profile)
        
        # Step 1: Voice introduction (Marathi)
        demo.add_step(DemoStep(
            timestamp=0,
            title="Voice Introduction",
            description="Priya introduces herself in Marathi",
            audio=self.generate_demo_audio(
                "नमस्कार, माझे नाव प्रिया आहे. मी पुण्यातून आहे.",
                language="mr"
            ),
            visualization="profile_card"
        ))
        
        # Step 2: Entity extraction and graph building
        demo.add_step(DemoStep(
            timestamp=30,
            title="Live Knowledge Graph Building",
            description="Extracting entities and building graph in real-time",
            entities_extracted={
                'name': 'Priya',
                'location': 'Pune',
                'age': 28,
                'family_size': 4,
                'income': 'BPL'
            },
            visualization="graph_animation",
            graph_snapshot=self.kg.export_subgraph(profile)
        ))
        
        # Step 3: Scheme eligibility query
        eligible_schemes = self.kg.find_eligible_schemes(profile)
        demo.add_step(DemoStep(
            timestamp=60,
            title="Scheme Discovery",
            description=f"Found {len(eligible_schemes)} eligible schemes",
            data={'schemes': [s.name for s in eligible_schemes]},
            visualization="scheme_list"
        ))
        
        # Step 4: Strategy optimization
        strategy = self.strategy_optimizer.optimize_path(profile)
        demo.add_step(DemoStep(
            timestamp=90,
            title="Optimal Roadmap Generation",
            description="Computing cascade-aware application sequence",
            data={
                'sequence': [s.name for s in strategy.sequence],
                'cascades': strategy.cascade_unlocks,
                'total_benefit': strategy.total_benefit_value
            },
            visualization="roadmap_timeline"
        ))
        
        # Step 5: Rejection risk assessment
        risk = self.risk_engine.assess_risk(profile, strategy.sequence[0])
        demo.add_step(DemoStep(
            timestamp=120,
            title="Rejection Risk Alert",
            description=f"Risk: {risk.overall_risk*100:.0f}%",
            data={
                'risk_score': risk.overall_risk,
                'risk_factors': [f.factor for f in risk.risk_factors],
                'mitigations': risk.mitigations
            },
            visualization="risk_gauge",
            alert=self.risk_engine.generate_alert(risk, "mr")
        ))
        
        # Step 6: Blockchain timestamping
        receipt = self.trust_anchor.timestamp_action(
            profile.synthetic_id,
            Action(type="DEMO_QUERY", scheme_id=strategy.sequence[0].id)
        )
        demo.add_step(DemoStep(
            timestamp=150,
            title="Blockchain Receipt",
            description="Tamper-proof execution proof generated",
            data={
                'tx_hash': receipt.transaction_hash,
                'block_number': receipt.block_number,
                'explorer_link': f"https://polygonscan.com/tx/{receipt.transaction_hash}"
            },
            visualization="blockchain_receipt"
        ))
        
        # Step 7: Collector dashboard view
        impact = self.governance_engine.simulate_policy_impact(
            PolicyChange(type="ELIGIBILITY_EXPANSION", scheme_id=strategy.sequence[0].id),
            self.sct.generate_batch(1000, district="Pune")
        )
        demo.add_step(DemoStep(
            timestamp=180,
            title="Collector Dashboard - Policy Simulation",
            description="What-if analysis on synthetic population",
            data={
                'beneficiaries_before': impact.beneficiaries_before,
                'beneficiaries_after': impact.beneficiaries_after,
                'roi': impact.estimated_roi
            },
            visualization="collector_dashboard"
        ))
        
        # Step 8: Metrics summary
        demo.add_step(DemoStep(
            timestamp=240,
            title="Impact Metrics",
            description="Before/After comparison",
            data={
                'rejection_rate_before': '40%',
                'rejection_rate_after': '8%',
                'time_to_benefit_before': '6 months',
                'time_to_benefit_after': '11 days',
                'synthetic_accuracy': '94%',
                'voice_accuracy': '89%'
            },
            visualization="metrics_comparison"
        ))
        
        return demo
    
    def generate_demo_audio(self, text: str, language: str) -> bytes:
        """Generate demo audio with high-quality TTS"""
        return self.voice.tts.synthesize(text, language, quality="high")
    
    def export_demo_package(self, demo: DemoExecution) -> DemoPackage:
        """Export demo for presentation"""
        return DemoPackage(
            slides=self.generate_slides(demo),
            video=self.generate_video(demo),
            interactive_html=self.generate_interactive_demo(demo),
            data_files=self.export_demo_data(demo)
        )
```

**Visualization Components:**

```python
class DemoVisualizer:
    def visualize_graph_building(self, entities: Dict, 
                                 graph: KnowledgeGraph) -> Animation:
        """Animate knowledge graph construction"""
        # Use D3.js or Cytoscape.js for interactive graph visualization
        pass
    
    def visualize_cascade_unlocks(self, strategy: OptimalStrategy) -> Animation:
        """Show cascade unlocking as schemes complete"""
        # Animated timeline with branching paths
        pass
    
    def visualize_risk_gauge(self, risk: RiskAssessment) -> Widget:
        """Interactive risk gauge with factor breakdown"""
        # Gauge chart with drill-down to risk factors
        pass
    
    def visualize_collector_dashboard(self, impact: ImpactReport) -> Dashboard:
        """Interactive collector dashboard"""
        # Multi-panel dashboard with charts and simulation controls
        pass
```

**Interface:**

```python
class DemoSystemInterface:
    def run_demo(self, profile_name: str) -> DemoExecution
    def get_demo_profiles(self) -> List[str]
    def export_demo_package(self, demo_id: str) -> DemoPackage
    def verify_zero_pii(self, demo: DemoExecution) -> bool
```

## Data Models

### Core Data Structures

```python
from dataclasses import dataclass
from typing import List, Dict, Optional
from datetime import datetime
from enum import Enum

@dataclass
class CitizenProfile:
    """Represents a citizen (real or synthetic)"""
    id: str
    age: int
    gender: str
    caste_category: str
    income_level: str
    family_size: int
    district: str
    tehsil: str
    occupation: str
    education_level: str
    disability_status: bool
    documents_possessed: List[str]
    is_synthetic: bool = False
    synthetic_id: Optional[str] = None

@dataclass
class Scheme:
    """Welfare scheme entity"""
    id: str
    name: str
    description: str
    eligibility_rules: Dict
    required_documents: List[str]
    benefits: List['Benefit']
    issuing_authority: str
    processing_days: int
    active: bool
    state: Optional[str] = None
    district: Optional[str] = None

@dataclass
class Document:
    """Document entity"""
    id: str
    type: str
    issuing_authority: str
    validity_period: int  # days
    obtaining_process: str
    required_for_schemes: List[str]

@dataclass
class Benefit:
    """Scheme benefit"""
    type: str  # CASH, SUBSIDY, SERVICE
    amount: Optional[float]
    frequency: str  # ONE_TIME, MONTHLY, ANNUAL
    description: str

@dataclass
class OptimalStrategy:
    """Optimized scheme application strategy"""
    sequence: List[Scheme]
    total_benefit_value: float
    estimated_time: int  # days
    cascade_unlocks: Dict[str, List[str]]  # scheme_id -> unlocked_scheme_ids
    document_reuse_count: int

@dataclass
class RiskAssessment:
    """Rejection risk assessment"""
    overall_risk: float  # 0.0 to 1.0
    risk_level: str  # LOW, MEDIUM, HIGH
    risk_factors: List['RiskFactor']
    mitigations: List[str]
    geo_context: str

@dataclass
class RiskFactor:
    """Individual risk contributor"""
    factor: str
    impact: float
    description: str

@dataclass
class PolicyChange:
    """Policy change from gazette"""
    source: str
    date: datetime
    type: str  # NEW_SCHEME, ELIGIBILITY_CHANGE, SCHEME_DISCONTINUED
    scheme_id: str
    description: str
    effective_date: datetime
    raw_text: str

@dataclass
class BlockchainReceipt:
    """Blockchain timestamp receipt"""
    transaction_hash: str
    block_number: int
    ipfs_hash: str
    timestamp: datetime
    gas_used: int

@dataclass
class Action:
    """Citizen action to be timestamped"""
    type: str  # QUERY, APPLICATION, DOCUMENT_UPLOAD, STATUS_CHECK
    scheme_id: str
    timestamp: datetime
    status: str
    location: str

class Language(Enum):
    """Supported languages"""
    HINDI = "hi"
    MARATHI = "mr"
    TAMIL = "ta"
    TELUGU = "te"
    BENGALI = "bn"
    GUJARATI = "gu"
    KANNADA = "kn"
    MALAYALAM = "ml"
    ODIA = "or"
    PUNJABI = "pa"
    ASSAMESE = "as"
    URDU = "ur"
```


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Synthetic Citizen Twin Properties

Property 1: SECC Distribution Matching
*For any* batch of synthetic citizen profiles generated for a district, the demographic distributions (caste, income, occupation) should match the SECC distributions for that district within 5% variance when measured using Chi-square tests.
**Validates: Requirements 1.1**

Property 2: NFHS Attribute Completeness
*For any* generated synthetic citizen profile, all NFHS-based attributes (health indicators, family size, education level) should be present and non-null.
**Validates: Requirements 1.2, 1.7**

Property 3: Eligibility Rule Application
*For any* synthetic citizen profile and any scheme, the profile's applicable schemes should exactly match those for which the profile satisfies the eligibility rules.
**Validates: Requirements 1.3**

Property 4: PII Exclusion
*For any* generated synthetic citizen profile, no field should contain values that match known PII patterns (real names, addresses, phone numbers) or entries from real citizen databases.
**Validates: Requirements 1.6**

Property 5: Document-Demographic Correlation
*For any* batch of synthetic profiles, the correlation between income/education level and document possession should match expected patterns (e.g., higher income correlates with more documents).
**Validates: Requirements 1.8**

### Knowledge Graph Properties

Property 6: Node Attribute Completeness
*For any* node added to the Knowledge Graph (Scheme, Document, or TehsilRule), all required attributes for that node type should be present and non-null.
**Validates: Requirements 2.1, 2.2, 2.3**

Property 7: Relationship Creation Correctness
*For any* two nodes in the Knowledge Graph, if a relationship condition is met (document required, scheme enables another, schemes conflict), then the corresponding relationship (REQUIRES, ENABLES, CONFLICTS) should exist in the graph.
**Validates: Requirements 2.4, 2.5, 2.6**

Property 8: Eligibility Query Completeness
*For any* citizen profile, the schemes returned by eligibility queries should be exactly the set of schemes whose eligibility rules are satisfied by the profile (no false positives, no false negatives).
**Validates: Requirements 2.9**

Property 9: Document Query Completeness
*For any* set of schemes, the documents returned by document queries should be exactly the union of all documents required by those schemes.
**Validates: Requirements 2.10**

### Voice Entity Extraction Properties

Property 10: Entity Extraction Completeness
*For any* voice input containing entities (age, family size, income, location, occupation, documents), all present entities should be extracted and included in the output.
**Validates: Requirements 3.1**

Property 11: Clarification Language Consistency
*For any* ambiguous voice input in language L, if a clarifying question is generated, it should be in the same language L.
**Validates: Requirements 3.4**

Property 12: Colloquial Term Mapping
*For any* colloquial term in the term mapping dictionary, when that term appears in voice input, the extracted entity should use the standard terminology.
**Validates: Requirements 3.7, 15.1, 15.3**

Property 13: Document Extraction Completeness
*For any* voice input mentioning documents, both the document type and possession status (has/needs) should be extracted.
**Validates: Requirements 3.8**

### Strategy Optimization Properties

Property 14: Eligible Scheme Identification
*For any* citizen profile, the Strategy Optimizer should identify exactly the schemes returned by the Knowledge Graph eligibility query (no additions, no omissions).
**Validates: Requirements 4.1**

Property 15: Cascade Prioritization
*For any* optimal strategy sequence, schemes that enable other schemes should appear before the schemes they enable (respecting dependency order).
**Validates: Requirements 4.2, 4.3**

Property 16: Document Reuse Optimization
*For any* two optimal strategy sequences for the same profile, the chosen sequence should minimize the total number of unique documents required across all schemes.
**Validates: Requirements 4.4**

Property 17: Conflict Avoidance
*For any* optimal strategy sequence, no two schemes in the sequence should have a CONFLICTS relationship in the Knowledge Graph.
**Validates: Requirements 4.10**

Property 18: Cascade Recalculation
*For any* citizen profile and completed scheme S, if S enables scheme T, then the recalculated optimal path should include T in the eligible schemes.
**Validates: Requirements 4.9**

### Rejection Risk Properties

Property 19: Risk Score Bounds
*For any* risk assessment, the overall rejection probability should be a value between 0.0 and 1.0 (0% to 100%).
**Validates: Requirements 5.9**

Property 20: High Risk Alerting
*For any* risk assessment where overall risk exceeds 30%, an alert should be generated that includes specific risk factors.
**Validates: Requirements 5.3**

Property 21: Alert Language Matching
*For any* risk alert generated for a citizen with language preference L, the alert text should be in language L.
**Validates: Requirements 5.4, 8.2, 14.8**

Property 22: Risk Factor Identification
*For any* risk assessment with overall risk > 20%, at least one risk factor should be identified and included in the assessment.
**Validates: Requirements 5.6**

Property 23: Mitigation Suggestion Completeness
*For any* identified risk factor, at least one corresponding mitigation suggestion should be provided.
**Validates: Requirements 5.7**

### Predictive Governance Properties

Property 24: Policy Change Extraction
*For any* gazette document containing policy changes (new schemes, eligibility changes, discontinuations), all policy changes should be extracted with their type, scheme ID, and effective date.
**Validates: Requirements 6.2**

Property 25: Affected Citizen Alerting
*For any* policy change affecting scheme S, all citizens who have interacted with scheme S should receive alerts about the change.
**Validates: Requirements 6.3**

Property 26: Policy Simulation Impact Calculation
*For any* policy change and synthetic population, the simulation should calculate the number of beneficiaries before and after the change, and the change in rejection risk.
**Validates: Requirements 6.5, 6.6**

Property 27: Policy Conflict Detection
*For any* two policies from different government levels (central, state, district) that affect the same scheme, if they have contradictory requirements, a conflict should be detected.
**Validates: Requirements 6.7**

Property 28: Budget Impact Estimation
*For any* budget allocation change for a scheme, the estimated impact on scheme availability (number of beneficiaries supportable) should be calculated.
**Validates: Requirements 6.9**

### Trust Anchor Properties

Property 29: Interaction Timestamping
*For any* citizen interaction (query, application, status check), a blockchain timestamp should be created with a transaction hash.
**Validates: Requirements 7.1, 7.2**

Property 30: Execution Proof Completeness
*For any* execution proof stored by the Trust Anchor, it should contain timestamp, action type, scheme identifier, and status fields.
**Validates: Requirements 7.3**

Property 31: Proof Verifiability
*For any* blockchain receipt with transaction hash H, verifying H on the blockchain should return the corresponding execution proof metadata.
**Validates: Requirements 7.4**

Property 32: Blockchain Immutability
*For any* execution proof written to the blockchain at time T, reading it at any time T' > T should return the same data (no modifications).
**Validates: Requirements 7.6**

Property 33: Grievance History Completeness
*For any* grievance request for citizen C and scheme S, all interactions between C and S should be retrievable with blockchain verification.
**Validates: Requirements 7.7**

Property 34: On-Chain PII Exclusion
*For any* execution proof stored on the blockchain, the on-chain data should not contain PII fields (names, addresses, phone numbers, Aadhaar numbers).
**Validates: Requirements 7.8**

### Voice Interface Properties

Property 35: Voice Navigation Completeness
*For any* system function F, there should exist a voice command sequence that can invoke F without requiring keyboard or mouse input.
**Validates: Requirements 8.1**

Property 36: Voice Input Failure Fallback
*For any* failed voice input attempt, the system should provide fallback options (retry, alternative input method).
**Validates: Requirements 8.7**

Property 37: Comprehension-Based Adaptation
*For any* user session where comprehension score falls below 0.5, the speech rate should be reduced to 80% of normal speed and text should be simplified.
**Validates: Requirements 8.10**

### Demo System Properties

Property 38: Demo Data Synthetic Purity
*For any* demo execution, all citizen profiles used should have is_synthetic=True and no field should contain real PII.
**Validates: Requirements 9.7**

### Performance and Resilience Properties

Property 39: Non-Blocking Blockchain Operations
*For any* user interaction that triggers a blockchain transaction, the user should be able to continue interacting with the system while the transaction is pending (no blocking).
**Validates: Requirements 10.6**

Property 40: API Retry with Exponential Backoff
*For any* external API call that fails, the system should retry up to 3 times with exponentially increasing delays between attempts.
**Validates: Requirements 14.1**

Property 41: Bhashini Fallback to Whisper
*For any* voice input, if Bhashini API fails or is unavailable, the system should process the input using Whisper instead.
**Validates: Requirements 14.2**

Property 42: Blockchain Transaction Queueing
*For any* blockchain transaction that fails, it should be added to a retry queue and the user interaction should continue without blocking.
**Validates: Requirements 14.3**

Property 43: Cached Results on Timeout
*For any* Knowledge Graph query that times out, if cached results exist, they should be returned with a staleness indicator.
**Validates: Requirements 14.4**

Property 44: Low Confidence Confirmation
*For any* voice recognition result with confidence below 70%, the system should request user confirmation before proceeding.
**Validates: Requirements 14.5**

Property 45: Database Reconnection on Failure
*For any* database connection failure, the system should attempt reconnection and alert administrators.
**Validates: Requirements 14.7**

Property 46: Graceful Service Degradation
*For any* system overload condition, non-critical features should be disabled before core functionality (eligibility queries, voice interface) is affected.
**Validates: Requirements 14.10**

### Privacy and Security Properties

Property 47: Voice Recording Retention Limit
*For any* voice recording stored without explicit user consent, it should be automatically deleted after 30 days.
**Validates: Requirements 11.3**

Property 48: Role-Based Access Control
*For any* administrative function, access should be granted only to users with the appropriate role.
**Validates: Requirements 11.4**

Property 49: Data Access Logging
*For any* access to citizen data, an audit log entry should be created with timestamp, user, and data accessed.
**Validates: Requirements 11.6**

Property 50: Analytics Data Anonymization
*For any* data used for analytics or model training, all personally identifiable fields should be removed or hashed.
**Validates: Requirements 11.7**

Property 51: Encrypted Third-Party Audio Transmission
*For any* voice audio sent to third-party services, it should be encrypted before transmission.
**Validates: Requirements 11.9**

### Integration Properties

Property 52: Webhook Triggering on Status Updates
*For any* scheme application status change, if a webhook is configured, it should be triggered with the status update data.
**Validates: Requirements 12.6**

Property 53: API Rate Limiting
*For any* public API endpoint, if a client exceeds the rate limit, subsequent requests should be rejected with a 429 status code until the rate limit window resets.
**Validates: Requirements 12.10**

### Monitoring Properties

Property 54: Rejection Rate Alert Threshold
*For any* time period where the system-wide rejection rate exceeds the target threshold, an alert should be generated for administrators.
**Validates: Requirements 13.3**

### Localization Properties

Property 55: District-Specific Rule Support
*For any* scheme with district-specific variations, querying the scheme for a citizen in district D should return the rules specific to D.
**Validates: Requirements 15.4**

Property 56: Regional Date Format
*For any* date or time displayed to a citizen from region R, the format should match the standard format for region R.
**Validates: Requirements 15.5**

Property 57: Code-Switching Handling
*For any* voice input containing words from multiple languages, entities should be extracted from all languages present.
**Validates: Requirements 15.6**

Property 58: Local Landmark Resolution
*For any* local landmark or administrative division mentioned in voice input, it should be resolved to the corresponding standard location identifier.
**Validates: Requirements 15.7**

Property 59: State-Specific Scheme Names
*For any* scheme with different names across states, when presenting the scheme to a citizen from state S, the name used in state S should be displayed.
**Validates: Requirements 15.10**


## Error Handling

### Error Categories and Strategies

**1. External Service Failures**

- **Bhashini API Unavailable**: Fallback to Whisper-only processing (Property 41)
- **Blockchain Transaction Failures**: Queue for retry, continue user interaction (Property 42)
- **Knowledge Graph Query Timeouts**: Return cached results with staleness indicator (Property 43)
- **External API Failures**: Retry with exponential backoff up to 3 attempts (Property 40)

**2. Voice Processing Errors**

- **Low Confidence Recognition (<70%)**: Request user confirmation (Property 44)
- **Ambiguous Input**: Ask clarifying questions in same language (Property 11)
- **Voice Input Failure**: Provide retry and alternative input options (Property 36)
- **Background Noise**: Use noise reduction preprocessing, request clearer input if needed

**3. Data Errors**

- **Missing Profile Attributes**: Request missing information through voice dialogue
- **Invalid Eligibility Data**: Log error, alert administrators, use default rules
- **Corrupted Synthetic Profiles**: Regenerate profile, exclude from batch

**4. System Errors**

- **Database Connection Failures**: Attempt reconnection, alert administrators (Property 45)
- **Service Overload**: Graceful degradation - disable non-critical features first (Property 46)
- **Memory Exhaustion**: Implement circuit breakers, scale horizontally

**5. Security Errors**

- **Unauthorized Access Attempts**: Reject with 403, log attempt (Property 49)
- **Rate Limit Exceeded**: Reject with 429, implement backoff (Property 53)
- **PII Leak Detection**: Immediate alert, quarantine affected data, audit trail

### Error Response Format

All errors should follow a consistent format:

```python
@dataclass
class ErrorResponse:
    error_code: str  # e.g., "VOICE_LOW_CONFIDENCE"
    message: str  # User-friendly message in user's language
    details: Optional[Dict]  # Technical details for debugging
    suggested_action: str  # What user should do next
    retry_possible: bool
    timestamp: datetime
```

### Circuit Breaker Pattern

Implement circuit breakers for all external services:

```python
class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, timeout: int = 60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitBreakerOpenError("Service unavailable")
        
        try:
            result = func(*args, **kwargs)
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
            raise e
```

## Testing Strategy

### Dual Testing Approach

HAQNEETI requires both unit testing and property-based testing for comprehensive coverage:

- **Unit tests**: Verify specific examples, edge cases, and error conditions
- **Property tests**: Verify universal properties across all inputs using randomized testing
- Both approaches are complementary and necessary

### Property-Based Testing Configuration

**Framework Selection:**
- Python: Use `hypothesis` library for property-based testing
- Minimum 100 iterations per property test (due to randomization)
- Each property test must reference its design document property

**Test Tag Format:**
```python
# Feature: haqneeti, Property 1: SECC Distribution Matching
@given(district=st.sampled_from(DISTRICTS), count=st.integers(min_value=1000, max_value=5000))
def test_secc_distribution_matching(district, count):
    # Test implementation
    pass
```

### Testing Layers

**1. Synthetic Citizen Twin Testing**

*Unit Tests:*
- Test profile generation with specific SECC distributions
- Test edge cases: empty districts, missing NFHS data
- Test document possession patterns for specific demographics

*Property Tests:*
- Property 1: SECC distribution matching (Chi-square test on generated batches)
- Property 2: NFHS attribute completeness (all profiles have required fields)
- Property 3: Eligibility rule application (generated profiles match eligibility)
- Property 4: PII exclusion (no real PII in synthetic data)
- Property 5: Document-demographic correlation (income correlates with documents)

**2. Knowledge Graph Testing**

*Unit Tests:*
- Test specific scheme additions with known attributes
- Test relationship creation for specific scheme pairs
- Test query performance with known graph sizes

*Property Tests:*
- Property 6: Node attribute completeness (all nodes have required attributes)
- Property 7: Relationship creation correctness (conditions imply relationships)
- Property 8: Eligibility query completeness (no false positives/negatives)
- Property 9: Document query completeness (exact union of required documents)

**3. Voice Entity Extraction Testing**

*Unit Tests:*
- Test extraction with specific voice samples in each language
- Test handling of specific colloquial terms
- Test edge cases: very short input, very long input, silence

*Property Tests:*
- Property 10: Entity extraction completeness (all entities extracted)
- Property 11: Clarification language consistency (same language for clarifications)
- Property 12: Colloquial term mapping (mapped terms use standard terminology)
- Property 13: Document extraction completeness (type and status extracted)

**4. Strategy Optimization Testing**

*Unit Tests:*
- Test optimization with specific profile and scheme combinations
- Test cascade detection with known dependency chains
- Test conflict resolution with known conflicting schemes

*Property Tests:*
- Property 14: Eligible scheme identification (matches KG query results)
- Property 15: Cascade prioritization (dependencies respected)
- Property 16: Document reuse optimization (minimizes unique documents)
- Property 17: Conflict avoidance (no conflicting schemes in sequence)
- Property 18: Cascade recalculation (enabled schemes included after completion)

**5. Rejection Risk Testing**

*Unit Tests:*
- Test risk calculation with specific historical patterns
- Test alert generation at risk thresholds
- Test mitigation suggestions for specific risk factors

*Property Tests:*
- Property 19: Risk score bounds (0.0 to 1.0)
- Property 20: High risk alerting (alerts for risk > 30%)
- Property 21: Alert language matching (alerts in user's language)
- Property 22: Risk factor identification (factors identified for risk > 20%)
- Property 23: Mitigation suggestion completeness (suggestions for all factors)

**6. Predictive Governance Testing**

*Unit Tests:*
- Test gazette scraping with sample gazette documents
- Test policy extraction with known policy changes
- Test simulation with specific policy scenarios

*Property Tests:*
- Property 24: Policy change extraction (all changes extracted)
- Property 25: Affected citizen alerting (all affected citizens alerted)
- Property 26: Policy simulation impact calculation (before/after metrics calculated)
- Property 27: Policy conflict detection (conflicts detected)
- Property 28: Budget impact estimation (availability impact calculated)

**7. Trust Anchor Testing**

*Unit Tests:*
- Test blockchain timestamping with specific actions
- Test proof verification with known transaction hashes
- Test grievance package generation with specific interaction histories

*Property Tests:*
- Property 29: Interaction timestamping (all interactions timestamped)
- Property 30: Execution proof completeness (all required fields present)
- Property 31: Proof verifiability (proofs verifiable on blockchain)
- Property 32: Blockchain immutability (data unchanged over time)
- Property 33: Grievance history completeness (all interactions retrievable)
- Property 34: On-chain PII exclusion (no PII on blockchain)

**8. Voice Interface Testing**

*Unit Tests:*
- Test specific voice commands for each function
- Test voice response generation in each language
- Test IVR and WhatsApp integration with sample messages

*Property Tests:*
- Property 35: Voice navigation completeness (all functions accessible via voice)
- Property 36: Voice input failure fallback (fallbacks provided)
- Property 37: Comprehension-based adaptation (speech adapted to comprehension)

**9. Demo System Testing**

*Unit Tests:*
- Test demo flow execution with "Priya" profile
- Test visualization generation for specific demo steps
- Test demo package export

*Property Tests:*
- Property 38: Demo data synthetic purity (all demo data is synthetic)

**10. Resilience and Error Handling Testing**

*Unit Tests:*
- Test specific error scenarios (API timeout, DB failure, etc.)
- Test circuit breaker state transitions
- Test retry logic with mock failures

*Property Tests:*
- Property 39: Non-blocking blockchain operations (user can continue during tx)
- Property 40: API retry with exponential backoff (3 retries with backoff)
- Property 41: Bhashini fallback to Whisper (Whisper used when Bhashini fails)
- Property 42: Blockchain transaction queueing (failed txs queued)
- Property 43: Cached results on timeout (cache returned with staleness flag)
- Property 44: Low confidence confirmation (confirmation requested for <70%)
- Property 45: Database reconnection on failure (reconnection attempted)
- Property 46: Graceful service degradation (non-critical features disabled first)

**11. Privacy and Security Testing**

*Unit Tests:*
- Test encryption of specific data samples
- Test role-based access with specific user roles
- Test data deletion for specific citizen records

*Property Tests:*
- Property 47: Voice recording retention limit (deleted after 30 days)
- Property 48: Role-based access control (access granted only with role)
- Property 49: Data access logging (all access logged)
- Property 50: Analytics data anonymization (PII removed from analytics)
- Property 51: Encrypted third-party audio transmission (audio encrypted)

**12. Integration Testing**

*Unit Tests:*
- Test webhook delivery with specific status updates
- Test API rate limiting with specific request patterns
- Test data export in each format (JSON, CSV, PDF)

*Property Tests:*
- Property 52: Webhook triggering on status updates (webhooks triggered)
- Property 53: API rate limiting (requests rejected after limit)

**13. Monitoring Testing**

*Unit Tests:*
- Test alert generation with specific metric thresholds
- Test dashboard data aggregation
- Test report generation

*Property Tests:*
- Property 54: Rejection rate alert threshold (alerts when threshold exceeded)

**14. Localization Testing**

*Unit Tests:*
- Test specific regional terms in each language
- Test date formatting for specific regions
- Test scheme name localization for specific states

*Property Tests:*
- Property 55: District-specific rule support (rules match district)
- Property 56: Regional date format (format matches region)
- Property 57: Code-switching handling (entities from all languages extracted)
- Property 58: Local landmark resolution (landmarks resolved to IDs)
- Property 59: State-specific scheme names (names match state)

### Test Data Generation

**Synthetic Profile Generators:**
```python
from hypothesis import strategies as st

@st.composite
def citizen_profile_strategy(draw):
    return CitizenProfile(
        id=draw(st.uuids()),
        age=draw(st.integers(min_value=18, max_value=100)),
        gender=draw(st.sampled_from(['M', 'F', 'O'])),
        caste_category=draw(st.sampled_from(['GEN', 'OBC', 'SC', 'ST'])),
        income_level=draw(st.sampled_from(['BPL', 'APL', 'MIDDLE', 'HIGH'])),
        family_size=draw(st.integers(min_value=1, max_value=15)),
        district=draw(st.sampled_from(DISTRICTS)),
        tehsil=draw(st.text(min_size=3, max_size=20)),
        occupation=draw(st.sampled_from(OCCUPATIONS)),
        education_level=draw(st.sampled_from(['NONE', 'PRIMARY', 'SECONDARY', 'GRADUATE'])),
        disability_status=draw(st.booleans()),
        documents_possessed=draw(st.lists(st.sampled_from(DOCUMENT_TYPES), min_size=0, max_size=10)),
        is_synthetic=True
    )
```

**Scheme Generators:**
```python
@st.composite
def scheme_strategy(draw):
    return Scheme(
        id=draw(st.uuids()),
        name=draw(st.text(min_size=5, max_size=50)),
        description=draw(st.text(min_size=10, max_size=200)),
        eligibility_rules=draw(st.dictionaries(st.text(), st.text())),
        required_documents=draw(st.lists(st.sampled_from(DOCUMENT_TYPES), min_size=1, max_size=5)),
        benefits=draw(st.lists(benefit_strategy(), min_size=1, max_size=3)),
        issuing_authority=draw(st.text(min_size=5, max_size=30)),
        processing_days=draw(st.integers(min_value=1, max_value=180)),
        active=True
    )
```

### Integration Testing

**End-to-End Demo Flow Test:**
```python
def test_complete_demo_flow():
    """Test the complete 5-minute demo flow"""
    demo_system = DemoSystem()
    
    # Execute demo
    demo = demo_system.run_demo_flow("Priya")
    
    # Verify all steps completed
    assert len(demo.steps) == 8
    
    # Verify synthetic data only
    assert demo.profile.is_synthetic
    assert verify_no_pii(demo)
    
    # Verify key metrics shown
    final_step = demo.steps[-1]
    assert 'rejection_rate_before' in final_step.data
    assert 'rejection_rate_after' in final_step.data
```

### Performance Testing

While not property-based, performance tests should validate:
- 1,000 concurrent voice sessions without degradation
- Knowledge Graph queries < 500ms for 99% of requests
- Strategy optimization < 2 seconds for 50 eligible schemes
- Voice response < 3 seconds for 95% of queries

### Continuous Testing

- Run property tests on every commit (with reduced iteration count: 20)
- Run full property test suite (100 iterations) nightly
- Run integration tests before deployment
- Run performance tests weekly
- Monitor production metrics against test baselines

