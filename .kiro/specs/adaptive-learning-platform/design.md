# Design Document: Adaptive Learning Platform V1

## Overview

This design describes a hackathon-scope adaptive learning platform that enables students to learn algebra through practice-first interactive exercises. The system uses a single pre-trained LLM as its central reasoning engine, with deterministic tools handling all execution, rendering, and validation. This architecture ensures demo reliability while maintaining clear separation between AI planning and system control.

**Core Design Principles:**
- **Single LLM, Multiple Roles**: One LLM API handles misconception analysis, hint generation, explanation planning, and visual instruction generation
- **AI Plans, Tools Execute**: LLM outputs structured plans; deterministic tools execute them
- **Stateless and Browser-Local**: No backend persistence, no authentication, session data in browser storage only
- **Graceful Degradation**: Pre-written fallbacks for all AI-generated content
- **2D Visualizations Only**: SVG/Canvas graphics, no 3D rendering in V1
- **Rule-Based Adaptation**: Deterministic logic for exercise sequencing based on performance signals

**What This Is NOT:**
- Not a video-based learning platform
- Not a full Learning Management System (LMS)
- Not production-ready (hackathon proof-of-concept)
- Not multi-subject (algebra only in V1)
- Not mobile-optimized (desktop/tablet web only)

## High-Level Architecture

The platform consists of four primary layers:

```
┌─────────────────────────────────────────────────────────┐
│                    Frontend SPA Layer                    │
│  (React/Vue/Svelte - UI, State Management, Routing)     │
└─────────────────────────────────────────────────────────┘
                           ↓ ↑
┌─────────────────────────────────────────────────────────┐
│                   AI Service Layer                       │
│  (LLM Integration, Prompt Engineering, Output Parsing)  │
└─────────────────────────────────────────────────────────┘
                           ↓ ↑
┌─────────────────────────────────────────────────────────┐
│                     Tool Layer                           │
│  (Visual Renderer, Validator, Exercise Selector)        │
└─────────────────────────────────────────────────────────┘
                           ↓ ↑
┌─────────────────────────────────────────────────────────┐
│                    Content Layer                         │
│  (Static JSON Exercises, Fallback Content, Schemas)     │
└─────────────────────────────────────────────────────────┘
```

### Layer Responsibilities

**Frontend SPA Layer:**
- Renders UI components (exercise display, input forms, feedback panels)
- Manages browser-local session state (current concept, exercise history, performance signals)
- Handles user interactions (answer submission, hint requests, explanation style selection)
- Coordinates between AI Service and Tool layers
- Implements responsive layout for desktop/tablet

**AI Service Layer:**
- Integrates with single LLM API (OpenAI, Anthropic, or similar)
- Constructs prompts for four distinct tasks: misconception analysis, hint generation, explanation planning, visual instruction planning
- Enforces structured outputs using JSON schemas or function calling
- Implements timeout handling (5-second max), retry logic, and fallback mechanisms
- Validates and sanitizes all LLM outputs before passing downstream
- Never executes code or commands from LLM responses

**Tool Layer:**
- **Visual Renderer**: Executes visual plans to generate SVG/Canvas 2D graphics
- **Validator**: Checks answer correctness using deterministic logic
- **Exercise Selector**: Implements rule-based adaptive sequencing
- All tools operate deterministically given the same input
- Tools validate plan structure before execution

**Content Layer:**
- Static JSON files containing exercises (text, options, correct answers, concept tags)
- Pre-written fallback hints and explanations
- Pre-rendered fallback visuals
- JSON schemas for validation

## AI Design: One LLM, Four Roles

The platform uses a single LLM API endpoint for all reasoning tasks. The LLM never executes actions directly—it only generates structured plans that tools execute.

### Role 1: Misconception Analysis

**When Triggered:** Student submits an incorrect answer

**Input to LLM:**
- Exercise text and correct answer
- Student's incorrect answer
- Previous attempts (if any)
- Concept being practiced

**LLM Task:** Analyze the incorrect answer to detect the underlying misconception

**Output Format (Structured JSON):**
```json
{
  "misconception_detected": true,
  "misconception_type": "sign_error_when_moving_terms",
  "confidence": "high",
  "explanation": "Student likely forgot to change sign when moving term across equals sign",
  "suggested_hint_level": 2
}
```

**Guardrails:**
- JSON schema validation on output
- Timeout: 3 seconds max
- Fallback: If analysis fails, return generic misconception type
- Sanitization: Remove any code or executable content from explanation field

### Role 2: Hint Generation

**When Triggered:** Student clicks "Get Hint" button

**Input to LLM:**
- Exercise text
- Student's previous attempts and detected misconceptions
- Requested hint level (1=gentle nudge, 2=specific guidance, 3=detailed walkthrough)
- Concept being practiced

**LLM Task:** Generate a contextual hint that guides without revealing the answer

**Output Format (Structured JSON):**
```json
{
  "hint_text": "When you move a term to the other side of the equation, what happens to its sign?",
  "hint_level": 2,
  "reveals_answer": false
}
```

**Guardrails:**
- JSON schema validation
- Timeout: 3 seconds max
- Validation: Check that hint doesn't contain the exact answer
- Fallback: Pre-written hints indexed by concept and hint level
- Sanitization: Strip any HTML, scripts, or executable content

### Role 3: Explanation Planning

**When Triggered:** Student requests an explanation (after incorrect attempts or proactively)

**Input to LLM:**
- Exercise text and correct answer
- Detected misconception (if any)
- Requested explanation style (visual, logical, analogy)
- Concept being practiced

**LLM Task:** Generate a structured plan for explaining the concept

**Output Format (Structured JSON):**
```json
{
  "explanation_style": "logical",
  "steps": [
    {
      "step_number": 1,
      "text": "Start with the equation: 2x + 5 = 13",
      "emphasis": "equation"
    },
    {
      "step_number": 2,
      "text": "Subtract 5 from both sides to isolate the term with x",
      "emphasis": "both_sides"
    },
    {
      "step_number": 3,
      "text": "This gives us: 2x = 8",
      "emphasis": "result"
    },
    {
      "step_number": 4,
      "text": "Divide both sides by 2 to solve for x",
      "emphasis": "both_sides"
    },
    {
      "step_number": 5,
      "text": "Final answer: x = 4",
      "emphasis": "answer"
    }
  ],
  "key_insight": "Whatever you do to one side of an equation, you must do to the other side to maintain balance"
}
```

**Guardrails:**
- JSON schema validation with required fields
- Timeout: 3 seconds max
- Validation: Ensure steps are sequential and coherent
- Fallback: Pre-written step-by-step explanations indexed by concept
- Sanitization: Remove any executable content from text fields

### Role 4: Visual Instruction Planning

**When Triggered:** Student requests a visual explanation

**Input to LLM:**
- Exercise text and correct answer
- Concept being practiced
- Detected misconception (if any)

**LLM Task:** Generate structured instructions for rendering a 2D educational visualization

**Output Format (Structured JSON):**
```json
{
  "visualization_type": "number_line",
  "title": "Solving 2x + 5 = 13 on a Number Line",
  "elements": [
    {
      "type": "number_line",
      "range": [-2, 10],
      "marked_points": [4],
      "labels": {"4": "x = 4"}
    },
    {
      "type": "annotation",
      "position": [4, 1],
      "text": "Solution: x = 4",
      "style": "highlight"
    }
  ],
  "caption": "The solution x = 4 is the point where the equation balances"
}
```

**Alternative Visualization Types:**
- `graph`: For plotting equations (linear, quadratic)
- `area_model`: For factoring and multiplication
- `balance_scale`: For equation solving
- `step_diagram`: For multi-step procedures

**Guardrails:**
- JSON schema validation with strict type checking
- Timeout: 3 seconds max
- Validation: Ensure visualization type is supported (2D only), coordinates are within bounds
- Fallback: Pre-rendered static visuals indexed by concept
- Sanitization: Validate all numeric values, strip any script injection attempts

### Structured Output Enforcement

To ensure reliability during demos, the AI Service Layer enforces structured outputs using one of these methods:

1. **JSON Mode** (if LLM provider supports): Force LLM to output valid JSON
2. **Function Calling** (if LLM provider supports): Define output schema as function parameters
3. **Prompt Engineering + Parsing**: Include JSON schema in prompt, parse and validate response
4. **Retry Logic**: If output is malformed, retry once with clarified prompt

**Validation Pipeline:**
```
LLM Response → JSON Parse → Schema Validation → Sanitization → Output
                  ↓ fail          ↓ fail           ↓ fail
               Retry Once → Fallback Content → Log Error
```

## Tool-Based Execution Model

The Tool Layer contains deterministic components that execute plans generated by the LLM. This separation ensures testability, reliability, and clear boundaries between AI reasoning and system control.

### Tool 1: Visual Renderer

**Purpose:** Execute visual plans to generate 2D educational graphics

**Input:** Visual plan JSON (from LLM Role 4)

**Output:** SVG or Canvas element

**Execution Logic:**
1. Parse visual plan JSON
2. Validate visualization type is supported
3. Map plan elements to rendering primitives:
   - `number_line` → SVG line with tick marks and labels
   - `graph` → SVG coordinate system with plotted function
   - `area_model` → SVG rectangles with dimensions and labels
   - `balance_scale` → SVG balance beam with weights
   - `step_diagram` → SVG flowchart with boxes and arrows
4. Apply educational styling (clear fonts, high contrast, labeled axes)
5. Cache rendered SVG in session storage (keyed by plan hash)
6. Return SVG element for DOM insertion

**Determinism:** Same visual plan always produces identical SVG output

**Error Handling:** If plan is invalid, return fallback visual for the concept

**Example:**
```javascript
// Input: Visual plan for number line
const plan = {
  visualization_type: "number_line",
  elements: [
    { type: "number_line", range: [-2, 10], marked_points: [4] }
  ]
};

// Output: SVG element
const svg = visualRenderer.execute(plan);
// <svg>...</svg> with number line from -2 to 10, point at 4 marked
```

### Tool 2: Validator

**Purpose:** Check answer correctness using deterministic logic

**Input:** 
- Exercise data (correct answer, answer type)
- Student's submitted answer

**Output:** 
```json
{
  "is_correct": false,
  "feedback": "Not quite. Check your arithmetic.",
  "partial_credit": 0.5
}
```

**Execution Logic:**
1. Normalize student answer (trim whitespace, lowercase for text, parse numbers)
2. Compare to correct answer based on answer type:
   - **Multiple choice**: Exact match
   - **Numeric**: Within tolerance (e.g., ±0.01)
   - **Algebraic expression**: Symbolic equivalence check (using math library)
   - **Short text**: Normalized string match or keyword presence
3. Return correctness boolean and feedback message

**Determinism:** Same answer and exercise always produce same validation result

**Error Handling:** If validation logic fails, default to incorrect with generic feedback

### Tool 3: Exercise Selector (Adaptive Engine)

**Purpose:** Select next exercise based on performance signals using rule-based logic

**Input:**
- Current concept
- Performance signals:
  - Recent accuracy (% correct in last 5 exercises)
  - Average time per question
  - Hint usage count
  - Answer change count (indicates uncertainty)
- Available exercises (from Content Layer)

**Output:**
```json
{
  "next_exercise_id": "linear_eq_medium_03",
  "difficulty": "medium",
  "reason": "Student showing mastery, increasing difficulty"
}
```

**Execution Logic (Rule-Based):**

```
IF recent_accuracy >= 0.8 AND avg_time < 60s AND hints_used == 0:
  → Increase difficulty OR introduce new sub-concept
  
ELSE IF recent_accuracy >= 0.6 AND recent_accuracy < 0.8:
  → Maintain current difficulty
  
ELSE IF recent_accuracy < 0.6 OR hints_used >= 2:
  → Decrease difficulty OR provide reinforcement exercise
  
ELSE IF answer_changes >= 3:
  → Provide similar exercise with different numbers (build confidence)
```

**Difficulty Levels:** easy, medium, hard (tagged in exercise JSON)

**Determinism:** Same performance signals and exercise pool always produce same selection

**Error Handling:** If no suitable exercise found, select random exercise from current concept

**Example:**
```javascript
// Input: Performance signals
const signals = {
  recent_accuracy: 0.85,
  avg_time: 45,
  hints_used: 0,
  answer_changes: 1
};

// Output: Next exercise selection
const selection = exerciseSelector.execute(signals, currentConcept);
// { next_exercise_id: "linear_eq_hard_01", difficulty: "hard", ... }
```

### Tool Invocation Flow

```
User Action → Frontend → AI Service (if needed) → Tool Layer → Frontend
                              ↓
                         LLM API
                              ↓
                      Structured Plan
                              ↓
                         Tool Execution
                              ↓
                      Deterministic Output
```

**Example: Student Requests Visual Explanation**

1. Frontend: User clicks "Show Visual Explanation"
2. AI Service: Constructs prompt for LLM Role 4 (Visual Instruction Planning)
3. LLM API: Returns visual plan JSON
4. AI Service: Validates and sanitizes plan
5. Tool Layer: Visual Renderer executes plan → generates SVG
6. Frontend: Displays SVG in explanation panel

## Visualization System

The platform supports 2D educational visualizations only. All visuals are generated from LLM plans and rendered using SVG or Canvas.

### Supported Visualization Types

1. **Number Line**
   - Use case: Solving inequalities, showing solution sets
   - Elements: Line, tick marks, labeled points, shaded regions
   - Educational focus: Spatial representation of numeric relationships

2. **Coordinate Graph**
   - Use case: Plotting linear/quadratic equations, systems of equations
   - Elements: X/Y axes, grid lines, plotted curves, intersection points
   - Educational focus: Visual relationship between equations and graphs

3. **Area Model**
   - Use case: Factoring, polynomial multiplication
   - Elements: Rectangles with dimensions, area labels, color coding
   - Educational focus: Geometric interpretation of algebraic operations

4. **Balance Scale**
   - Use case: Equation solving, maintaining equality
   - Elements: Balance beam, weights on each side, labels
   - Educational focus: Equation as balanced relationship

5. **Step Diagram**
   - Use case: Multi-step problem solving
   - Elements: Boxes with steps, arrows showing flow, annotations
   - Educational focus: Procedural understanding

### Rendering Pipeline

```
LLM Visual Plan → Validation → Rendering → Caching → Display
                      ↓ fail       ↓ fail
                  Fallback Visual
```

**Validation Checks:**
- Visualization type is supported (2D only)
- Coordinates are within reasonable bounds
- Required elements are present
- No script injection attempts

**Rendering Implementation:**
- **SVG** (preferred): Scalable, accessible, easy to style
- **Canvas** (alternative): For complex animations or large datasets

**Caching Strategy:**
- Cache rendered visuals in session storage
- Key: Hash of visual plan JSON
- Benefit: Avoid re-rendering identical visuals, improve demo performance

**Educational Accuracy:**
- Prioritize clarity over visual realism
- Use high-contrast colors for accessibility
- Label all axes, points, and regions
- Include legends and captions
- Ensure mathematical correctness (e.g., accurate scale, proper curve shapes)

### Fallback Visuals

For each concept, pre-render 2-3 static visuals covering common scenarios:
- Linear equations: Graph showing y = mx + b with labeled slope and intercept
- Quadratic equations: Parabola with vertex and roots labeled
- Factoring: Area model showing (x + a)(x + b) expansion

Store as SVG files in Content Layer, indexed by concept.

## Adaptive Engine

The Adaptive Engine selects the next exercise based on performance signals using rule-based logic. This is implemented as the Exercise Selector tool.

### Inputs

**Performance Signals:**
1. **Recent Accuracy**: Percentage of correct answers in last 5 exercises
2. **Average Time**: Mean time spent per question (in seconds)
3. **Hint Usage**: Number of hints requested in current session
4. **Answer Changes**: Number of times student changed answer before submitting

**Context:**
- Current concept being practiced
- Available exercises (from Content Layer, filtered by concept)
- Exercise metadata (difficulty, sub-concept tags)

### Outputs

**Selection Decision:**
- Next exercise ID
- Difficulty level
- Reason for selection (for logging/debugging)

### Rule-Based Logic

The engine uses a decision tree based on performance thresholds:

```
┌─────────────────────────────────────────────────────────┐
│ Performance Assessment                                   │
└─────────────────────────────────────────────────────────┘
                           ↓
        ┌──────────────────┼──────────────────┐
        ↓                  ↓                   ↓
   High Mastery      Moderate Mastery    Low Mastery
   (acc >= 80%,      (acc 60-80%,        (acc < 60%,
    time < 60s,       time < 90s,         OR hints >= 2,
    hints == 0)       hints <= 1)         OR time > 90s)
        ↓                  ↓                   ↓
   Increase           Maintain            Decrease
   Difficulty         Difficulty          Difficulty
        ↓                  ↓                   ↓
   Select Hard        Select Medium       Select Easy
   OR New Concept     Same Concept        OR Reinforcement
```

**Special Cases:**
- **High Answer Changes (>= 3)**: Select similar exercise with different numbers (build confidence through repetition)
- **Consecutive Correct (>= 5)**: Introduce new sub-concept within current concept
- **Consecutive Incorrect (>= 3)**: Provide reinforcement exercise at easier difficulty

**Exercise Pool Management:**
- Filter exercises by current concept
- Further filter by selected difficulty
- Exclude recently seen exercises (last 3)
- Randomly select from remaining pool

### Example Scenarios

**Scenario 1: Student Excelling**
- Signals: 90% accuracy, 40s avg time, 0 hints
- Decision: Select hard exercise or introduce new sub-concept
- Reason: Student demonstrating mastery, ready for challenge

**Scenario 2: Student Struggling**
- Signals: 40% accuracy, 120s avg time, 3 hints
- Decision: Select easy reinforcement exercise
- Reason: Student needs more practice at foundational level

**Scenario 3: Student Uncertain**
- Signals: 70% accuracy, 60s avg time, 1 hint, 4 answer changes
- Decision: Select similar exercise with different numbers at same difficulty
- Reason: Student understands concept but lacks confidence

### Future ML Extension Point

The rule-based engine is designed to be replaceable with an ML model in V2:
- Same input/output interface
- ML model could learn optimal sequencing from student interaction data
- Rule-based logic serves as baseline for ML comparison

## Data Handling

The platform is stateless and browser-local. No backend persistence, no user authentication, no personal data collection.

### Session State (Browser Local Storage)

**Stored Data:**
```json
{
  "session_id": "uuid-v4",
  "current_concept": "linear_equations",
  "exercise_history": [
    {
      "exercise_id": "linear_eq_easy_01",
      "timestamp": "2024-01-15T10:30:00Z",
      "correct": true,
      "time_spent": 45,
      "hints_used": 0,
      "answer_changes": 1
    }
  ],
  "performance_signals": {
    "recent_accuracy": 0.8,
    "avg_time": 52,
    "total_hints": 2,
    "total_exercises": 10
  },
  "cached_visuals": {
    "plan_hash_abc123": "<svg>...</svg>"
  }
}
```

**Storage Mechanism:**
- Use browser `localStorage` API
- Key: `adaptive_learning_session`
- Max size: ~5MB (well within localStorage limits)

**Data Lifecycle:**
- Created: When student starts first exercise
- Updated: After each exercise submission, hint request, or explanation view
- Cleared: When student clicks "End Session" or closes browser (optional)

**Privacy:**
- No personal identifying information stored
- No data sent to backend servers
- Session data never leaves the browser
- Student can clear data at any time

### Static Content (JSON Files)

**Exercise Data:**
```json
{
  "exercises": [
    {
      "id": "linear_eq_easy_01",
      "concept": "linear_equations",
      "sub_concept": "one_step",
      "difficulty": "easy",
      "type": "short_answer",
      "text": "Solve for x: x + 5 = 12",
      "correct_answer": "7",
      "answer_type": "numeric",
      "fallback_hint": "Try subtracting 5 from both sides",
      "fallback_visual": "linear_eq_number_line_01.svg"
    }
  ]
}
```

**Loading:**
- Fetch JSON files on app initialization
- Validate structure using JSON schema
- Store in memory for session duration
- Log errors for malformed entries, skip invalid exercises

**Content Organization:**
```
/content
  /exercises
    linear_equations.json
    quadratic_equations.json
    factoring.json
    systems_of_equations.json
  /fallbacks
    /hints
      linear_equations.json
    /visuals
      linear_eq_number_line_01.svg
      quadratic_parabola_01.svg
```

### No Backend Persistence

**Implications:**
- No user accounts or authentication
- No cross-device progress sync
- No analytics or usage tracking
- No content management system

**Benefits for V1:**
- Simpler architecture
- Faster development
- No server costs
- No data privacy concerns
- Easier demo deployment (static hosting)

## Failure Handling

The platform implements graceful degradation to ensure demos remain functional even when the AI service has issues.

### LLM Service Failures

**Timeout (> 5 seconds):**
- Action: Cancel request, log timeout
- Fallback: Use pre-written content for the task
- User Experience: Brief "Generating..." message, then fallback content appears

**API Error (500, 503, rate limit):**
- Action: Log error with details
- Fallback: Use pre-written content
- User Experience: No error message shown, seamless fallback

**Malformed Output:**
- Action: Attempt to parse, validate against schema
- Retry: If parsing fails, retry once with clarified prompt
- Fallback: If retry fails, use pre-written content
- User Experience: Slight delay, then fallback content

**Invalid Plan (e.g., unsupported visualization type):**
- Action: Validate plan structure
- Fallback: Use pre-rendered visual for concept
- User Experience: Visual appears, may be less specific to current exercise

### Fallback Content Strategy

**Misconception Analysis Fallback:**
- Generic misconception type: "arithmetic_error"
- Generic explanation: "Check your calculations carefully"

**Hint Generation Fallback:**
- Pre-written hints indexed by concept and hint level
- Example: `hints/linear_equations.json`:
  ```json
  {
    "level_1": "Think about what operation would isolate the variable",
    "level_2": "Try moving the constant term to the other side",
    "level_3": "Subtract 5 from both sides, then divide by the coefficient"
  }
  ```

**Explanation Planning Fallback:**
- Pre-written step-by-step explanations indexed by concept
- Example: `explanations/linear_equations.json`:
  ```json
  {
    "steps": [
      "Identify the variable and constants",
      "Use inverse operations to isolate the variable",
      "Perform the same operation on both sides",
      "Simplify to find the solution"
    ]
  }
  ```

**Visual Instruction Planning Fallback:**
- Pre-rendered SVG visuals indexed by concept
- Example: `visuals/linear_eq_number_line_01.svg`

### Degradation Modes

**Mode 1: AI-Assisted (Normal)**
- LLM generates all content
- Tools execute plans
- Adaptive sequencing active

**Mode 2: Partial Fallback**
- Some LLM tasks succeed, others use fallbacks
- Tools still execute successfully
- Adaptive sequencing active

**Mode 3: Full Fallback (Graceful Degradation)**
- All LLM tasks use pre-written content
- Tools execute with fallback plans
- Adaptive sequencing may use simplified rules or random selection

**Mode 4: Static Mode (Extreme Failure)**
- No AI service available
- All content is pre-written
- Exercises presented in fixed order
- No adaptive behavior

**User Experience Across Modes:**
- Modes 1-3: Indistinguishable to user (seamless fallbacks)
- Mode 4: Noticeable (no personalization), but platform still functional

### Error Logging

**What to Log:**
- LLM API errors (status code, error message, timestamp)
- Timeout events
- Malformed output examples
- Fallback usage counts
- Tool execution failures

**Where to Log:**
- Browser console (for development)
- Optional: Send to logging service (if backend added in future)

**What NOT to Show Users:**
- Raw error messages
- Stack traces
- API keys or internal details
- "Something went wrong" messages (use fallbacks instead)

## Components and Interfaces

### Frontend Components

**ExerciseDisplay**
- Props: `exercise` (object), `onSubmit` (function)
- Renders: Exercise text, input field (MCQ/short answer), submit button
- State: Current answer, validation result

**FeedbackPanel**
- Props: `isCorrect` (boolean), `feedback` (string), `misconception` (object)
- Renders: Feedback message, misconception explanation (if detected)
- Actions: Request hint, request explanation

**HintDisplay**
- Props: `hint` (object), `hintLevel` (number)
- Renders: Hint text, hint level indicator
- Actions: Request next hint level

**ExplanationPanel**
- Props: `explanation` (object), `style` (string)
- Renders: Step-by-step explanation or visual explanation
- Actions: Switch explanation style

**VisualRenderer** (Component wrapper for Tool)
- Props: `visualPlan` (object)
- Renders: SVG or Canvas element
- Uses: Visual Renderer tool from Tool Layer

**ProgressIndicator**
- Props: `performanceSignals` (object), `currentConcept` (string)
- Renders: Progress bar, accuracy percentage, concept name

### AI Service Interface

```typescript
interface AIService {
  analyzeMisconception(
    exercise: Exercise,
    studentAnswer: string,
    previousAttempts: Attempt[]
  ): Promise<MisconceptionAnalysis>;

  generateHint(
    exercise: Exercise,
    misconception: MisconceptionAnalysis,
    hintLevel: number
  ): Promise<Hint>;

  planExplanation(
    exercise: Exercise,
    misconception: MisconceptionAnalysis,
    style: ExplanationStyle
  ): Promise<ExplanationPlan>;

  planVisual(
    exercise: Exercise,
    concept: string,
    misconception?: MisconceptionAnalysis
  ): Promise<VisualPlan>;
}
```

**Types:**
```typescript
type ExplanationStyle = "visual" | "logical" | "analogy";

interface MisconceptionAnalysis {
  misconception_detected: boolean;
  misconception_type: string;
  confidence: "low" | "medium" | "high";
  explanation: string;
  suggested_hint_level: number;
}

interface Hint {
  hint_text: string;
  hint_level: number;
  reveals_answer: boolean;
}

interface ExplanationPlan {
  explanation_style: ExplanationStyle;
  steps: ExplanationStep[];
  key_insight: string;
}

interface VisualPlan {
  visualization_type: "number_line" | "graph" | "area_model" | "balance_scale" | "step_diagram";
  title: string;
  elements: VisualElement[];
  caption: string;
}
```

### Tool Layer Interface

```typescript
interface ToolLayer {
  visualRenderer: VisualRenderer;
  validator: Validator;
  exerciseSelector: ExerciseSelector;
}

interface VisualRenderer {
  execute(plan: VisualPlan): SVGElement | HTMLCanvasElement;
  getCached(planHash: string): SVGElement | HTMLCanvasElement | null;
}

interface Validator {
  validate(exercise: Exercise, studentAnswer: string): ValidationResult;
}

interface ExerciseSelector {
  selectNext(
    signals: PerformanceSignals,
    concept: string,
    exercises: Exercise[]
  ): ExerciseSelection;
}
```

**Types:**
```typescript
interface ValidationResult {
  is_correct: boolean;
  feedback: string;
  partial_credit: number;
}

interface ExerciseSelection {
  next_exercise_id: string;
  difficulty: "easy" | "medium" | "hard";
  reason: string;
}

interface PerformanceSignals {
  recent_accuracy: number;
  avg_time: number;
  hints_used: number;
  answer_changes: number;
}
```

### Content Layer Interface

```typescript
interface ContentLayer {
  loadExercises(concept: string): Promise<Exercise[]>;
  getFallbackHint(concept: string, level: number): string;
  getFallbackExplanation(concept: string): ExplanationPlan;
  getFallbackVisual(concept: string): string; // SVG file path
}

interface Exercise {
  id: string;
  concept: string;
  sub_concept: string;
  difficulty: "easy" | "medium" | "hard";
  type: "multiple_choice" | "short_answer" | "puzzle";
  text: string;
  correct_answer: string;
  answer_type: "numeric" | "text" | "algebraic";
  options?: string[]; // For MCQ
  fallback_hint: string;
  fallback_visual: string;
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

Before defining properties, I need to analyze the acceptance criteria from the requirements document to determine which are testable as properties, examples, or edge cases.


### Property Reflection

After analyzing all acceptance criteria, I've identified several areas where properties can be consolidated:

**Consolidation Opportunities:**
1. **Exercise Type Support (1.3, 1.4, 1.5)**: Can be combined into one property about rendering any exercise type
2. **Performance Signal Consideration (5.2, 5.3, 5.4, 5.5)**: Can be combined into one property about signal influence on selection
3. **Explanation Style Support (6.1, 6.2, 6.3)**: Can be combined into one property about supporting all styles
4. **Tool Existence (10.3, 10.4, 10.5)**: These are examples, not properties - testing tool availability
5. **Fallback Mechanisms (2.5, 3.5, 4.8, 11.1, 11.2, 11.3, 11.4)**: Can be consolidated into fewer properties about fallback behavior
6. **JSON Structure Validation (12.2, 12.3)**: Can be combined into one property about JSON validation
7. **LLM Output Validation (9.2, 9.4)**: Can be combined into one property about output validation pipeline

**Properties to Keep Separate:**
- Deterministic tool execution (4.4, 10.7) - critical correctness property
- Timeout handling (2.2, 9.3) - performance requirements
- Security constraints (9.6, 9.7) - critical safety properties
- Adaptive behavior (5.7, 5.8) - core functionality
- Session state management (7.1, 7.2, 7.3, 7.4) - data handling properties

### Correctness Properties

Property 1: Exercise Type Rendering
*For any* exercise of type multiple_choice, short_answer, or puzzle, the Platform should successfully render the exercise with appropriate input controls
**Validates: Requirements 1.3, 1.4, 1.5**

Property 2: Answer Validation Performance
*For any* submitted answer, the Validator tool should return a validation result within 2 seconds
**Validates: Requirements 1.6**

Property 3: Correct Answer Advancement
*For any* exercise, when a correct answer is submitted, the Platform should display positive feedback and load the next exercise
**Validates: Requirements 1.7**

Property 4: Incorrect Answer Retention
*For any* exercise, when an incorrect answer is submitted, the Platform should retain the same exercise and display feedback options
**Validates: Requirements 1.8**

Property 5: Misconception Analysis Structure
*For any* incorrect answer, the LLM_Service should return a misconception analysis object containing misconception_detected, misconception_type, confidence, explanation, and suggested_hint_level fields
**Validates: Requirements 2.1, 2.4**

Property 6: Misconception Analysis Performance
*For any* incorrect answer, the LLM_Service should return misconception analysis within 3 seconds or trigger fallback
**Validates: Requirements 2.2**

Property 7: Multi-Attempt Pattern Detection
*For any* sequence of incorrect attempts on the same exercise, the LLM_Service should include information from previous attempts in its analysis
**Validates: Requirements 2.3**

Property 8: LLM Service Fallback on Failure
*For any* LLM_Service operation (misconception analysis, hint generation, explanation planning, visual planning), if the service fails or times out, the Platform should return pre-written fallback content without displaying errors
**Validates: Requirements 2.5, 3.5, 4.8, 11.1, 11.2, 11.3, 11.4**

Property 9: Contextual Hint Generation
*For any* hint request, the LLM_Service should generate a hint that references the current exercise and does not contain the exact correct answer
**Validates: Requirements 3.1, 3.3**

Property 10: Progressive Hint Specificity
*For any* sequence of hint requests on the same exercise, each subsequent hint should have a hint_level greater than or equal to the previous hint_level
**Validates: Requirements 3.2**

Property 11: Hint Usage Tracking
*For any* hint request, the Platform should increment the hints_used counter in the performance signals
**Validates: Requirements 3.4**

Property 12: Visual Plan Structure
*For any* visual explanation request, the LLM_Service should return a visual plan object containing visualization_type, title, elements, and caption fields, with visualization_type being one of: number_line, graph, area_model, balance_scale, or step_diagram
**Validates: Requirements 4.1, 4.2, 4.3**

Property 13: Deterministic Visual Rendering
*For any* visual plan, executing the plan through the Visual_Renderer multiple times should produce identical output (same SVG structure or Canvas rendering)
**Validates: Requirements 4.4, 10.7**

Property 14: Visual Output Type
*For any* visual plan execution, the Visual_Renderer should return either an SVGElement or HTMLCanvasElement
**Validates: Requirements 4.5**

Property 15: Visual Caching
*For any* visual plan, if the same plan (by hash) is rendered twice in a session, the second rendering should retrieve the cached result
**Validates: Requirements 4.6**

Property 16: Performance Signal Influence on Selection
*For any* two different performance signal sets (varying accuracy, time, hints, or answer changes), if the signals differ significantly, the Exercise_Selector should select exercises with different difficulty levels or concepts
**Validates: Requirements 5.1, 5.2, 5.3, 5.4, 5.5**

Property 17: Mastery-Based Difficulty Increase
*For any* performance signals indicating mastery (accuracy >= 80%, time < 60s, hints == 0), the Exercise_Selector should select an exercise with difficulty "hard" or a new sub-concept
**Validates: Requirements 5.7**

Property 18: Struggle-Based Difficulty Decrease
*For any* performance signals indicating struggle (accuracy < 60% OR hints >= 2 OR time > 90s), the Exercise_Selector should select an exercise with difficulty "easy" or "medium" (not harder than current)
**Validates: Requirements 5.8**

Property 19: Explanation Style Support
*For any* explanation request with style set to "visual", "logical", or "analogy", the LLM_Service should return an explanation plan with explanation_style matching the requested style
**Validates: Requirements 6.1, 6.2, 6.3**

Property 20: Multi-Style Explanation Availability
*For any* concept, the Platform should be able to generate explanations in all three styles (visual, logical, analogy) for the same concept
**Validates: Requirements 6.4**

Property 21: Misconception-Tailored Explanation
*For any* two explanation requests for the same exercise with different detected misconceptions, the explanation plans should differ in content or complexity
**Validates: Requirements 6.5**

Property 22: Browser-Local Storage Only
*For any* session state update, the Platform should write data to browser localStorage and should not make network requests to backend storage endpoints
**Validates: Requirements 7.1, 7.3**

Property 23: Session State Structure
*For any* session state stored in localStorage, the state object should contain current_concept, exercise_history, and performance_signals fields
**Validates: Requirements 7.2**

Property 24: No Personal Identifying Information
*For any* session state stored in localStorage, the state object should not contain fields for name, email, address, phone, or other PII
**Validates: Requirements 7.4**

Property 25: Graceful State Clearing
*For any* session, clearing localStorage should not cause the Platform to crash or display errors - it should initialize a new session
**Validates: Requirements 7.5**

Property 26: Algebra-Only Content Scope
*For all* exercises loaded by the Platform, the concept field should be one of: linear_equations, quadratic_equations, factoring, or systems_of_equations (all algebra concepts)
**Validates: Requirements 8.1, 8.6**

Property 27: Structured Output Validation
*For all* LLM_Service outputs (misconception analysis, hints, explanation plans, visual plans), the output should be valid JSON conforming to the defined schema for that output type
**Validates: Requirements 9.2, 9.4**

Property 28: LLM Service Timeout Enforcement
*For any* LLM_Service request, if the request takes longer than 5 seconds, the service should cancel the request and trigger fallback mechanisms
**Validates: Requirements 9.3**

Property 29: Invalid Output Recovery
*For any* LLM_Service request that returns invalid or malformed output, the service should either retry once or immediately fall back to default content
**Validates: Requirements 9.5**

Property 30: No Code Execution from LLM
*For all* LLM_Service outputs, the output should never be passed to eval(), exec(), Function(), or similar code execution functions
**Validates: Requirements 9.6**

Property 31: LLM Output Sanitization
*For all* LLM_Service outputs that will be displayed to students, the output should be sanitized (HTML escaped, script tags removed) before rendering
**Validates: Requirements 9.7**

Property 32: LLM Service Returns Plans Not Side Effects
*For all* LLM_Service methods (analyzeMisconception, generateHint, planExplanation, planVisual), the method should return a structured plan object and should not execute actions or modify state directly
**Validates: Requirements 10.1**

Property 33: Tool Layer Executes All Plans
*For any* plan generated by LLM_Service (visual plan, explanation plan), there should be a corresponding tool in the Tool_Layer that can execute that plan type
**Validates: Requirements 10.2**

Property 34: Plan Validation Before Execution
*For any* plan passed to the Tool_Layer, the tool should validate the plan structure before execution and reject invalid plans
**Validates: Requirements 10.6**

Property 35: No Raw Error Display
*For any* error that occurs in the Platform (LLM timeout, validation failure, tool error), the error message displayed to students should not contain stack traces, error codes, or technical implementation details
**Validates: Requirements 11.6**

Property 36: Exercise JSON Structure
*For all* exercises loaded from JSON files, each exercise object should contain id, concept, difficulty, type, text, correct_answer, and answer_type fields
**Validates: Requirements 12.1, 12.2**

Property 37: JSON Validation on Load
*For any* exercise JSON file loaded by the Platform, the Platform should validate the JSON structure against the exercise schema before making exercises available
**Validates: Requirements 12.3**

Property 38: Malformed JSON Handling
*For any* exercise JSON file that is malformed or fails validation, the Platform should log an error and skip the invalid exercises without crashing
**Validates: Requirements 12.4**

Property 39: Dynamic Exercise Addition
*For any* new exercise added to the JSON files, the exercise should become available in the Platform after reloading the content without requiring code changes or redeployment
**Validates: Requirements 12.5**

Property 40: Subject Extensibility
*For any* new subject (e.g., geometry, calculus), adding exercises with a new concept tag should not require modifications to the Frontend, AI Service, or Tool Layer core components
**Validates: Requirements 14.1**

Property 41: 3D Rendering Interface Compatibility
*For any* visual plan with a hypothetical 3D visualization_type (e.g., "3d_graph"), the Visual_Renderer interface should be able to accept the plan without interface changes (implementation may return "not supported" in V1)
**Validates: Requirements 14.2**

Property 42: ML Model Interface Compatibility
*For any* hypothetical ML-based Exercise_Selector implementation, the implementation should be able to replace the rule-based selector without changing the interface (same inputs: performance signals, concept, exercises; same output: exercise selection)
**Validates: Requirements 14.3**

Property 43: Separation of Concerns
*For all* Platform components, content data (exercises, fallbacks) should be in separate modules from business logic (AI Service, Tool Layer) and presentation (Frontend components)
**Validates: Requirements 14.4**

Property 44: LLM Provider Abstraction
*For any* LLM provider (OpenAI, Anthropic, Cohere, etc.), the AI Service should be able to integrate with the provider through a common interface without changing the Frontend or Tool Layer
**Validates: Requirements 14.5**

## Error Handling

The platform implements a multi-layered error handling strategy to ensure graceful degradation and demo reliability.

### Error Categories

**1. LLM Service Errors**
- Timeout (> 5 seconds)
- API errors (500, 503, rate limits)
- Malformed output (invalid JSON, missing fields)
- Invalid plans (unsupported types, out-of-bounds values)

**2. Tool Execution Errors**
- Visual rendering failures (invalid plan, rendering exception)
- Validation errors (unexpected answer format)
- Exercise selection failures (no suitable exercises)

**3. Content Loading Errors**
- Malformed JSON files
- Missing required fields
- Invalid exercise data

**4. Session State Errors**
- localStorage quota exceeded
- Corrupted session data
- Missing session state

### Error Handling Strategies

**Retry Logic:**
- LLM requests: Retry once with clarified prompt if output is malformed
- Content loading: Skip invalid entries, continue with valid ones
- Tool execution: No retry (deterministic, should succeed or fail consistently)

**Fallback Content:**
- Misconception analysis → Generic misconception type
- Hint generation → Pre-written hints indexed by concept and level
- Explanation planning → Pre-written step-by-step explanations
- Visual planning → Pre-rendered SVG visuals

**Graceful Degradation:**
- Mode 1 (Normal): All AI features working
- Mode 2 (Partial): Some AI features use fallbacks
- Mode 3 (Degraded): All AI features use fallbacks, adaptive logic simplified
- Mode 4 (Static): Fixed exercise order, no personalization

**User Experience:**
- Never show raw errors, stack traces, or technical details
- Use neutral messages: "Generating explanation..." → fallback appears
- Maintain functionality even in degraded modes
- Log all errors for debugging (console in dev, optional service in production)

### Error Recovery Examples

**Example 1: LLM Timeout**
```
User requests hint
→ AI Service calls LLM API
→ Request exceeds 5 seconds
→ AI Service cancels request
→ AI Service loads fallback hint from content layer
→ Frontend displays fallback hint
→ User sees: "Try isolating the variable on one side"
```

**Example 2: Invalid Visual Plan**
```
User requests visual explanation
→ AI Service receives visual plan from LLM
→ Plan validation fails (unsupported visualization type)
→ Tool Layer rejects plan
→ AI Service loads fallback visual from content layer
→ Frontend displays pre-rendered SVG
→ User sees: Generic number line visual for concept
```

**Example 3: Malformed Exercise JSON**
```
Platform loads exercises on initialization
→ Content Layer reads linear_equations.json
→ JSON parsing succeeds, schema validation fails on exercise #5
→ Content Layer logs error: "Invalid exercise: missing correct_answer field"
→ Content Layer skips exercise #5
→ Content Layer loads remaining valid exercises
→ Platform continues with 9 out of 10 exercises
```

## Testing Strategy

The platform requires comprehensive testing across multiple layers to ensure correctness, reliability, and demo readiness.

### Testing Approach

**Dual Testing Strategy:**
- **Unit Tests**: Verify specific examples, edge cases, and error conditions
- **Property-Based Tests**: Verify universal properties across randomized inputs

Both approaches are complementary and necessary for comprehensive coverage. Unit tests catch concrete bugs and validate specific scenarios, while property tests verify general correctness across a wide input space.

### Property-Based Testing Configuration

**Library Selection:**
- **JavaScript/TypeScript**: fast-check
- **Python**: Hypothesis
- **Other languages**: Use language-appropriate PBT library

**Test Configuration:**
- Minimum 100 iterations per property test (due to randomization)
- Each property test must reference its design document property
- Tag format: `// Feature: adaptive-learning-platform, Property N: [property text]`

**Example Property Test (TypeScript with fast-check):**
```typescript
// Feature: adaptive-learning-platform, Property 13: Deterministic Visual Rendering
test('Visual Renderer produces identical output for same plan', () => {
  fc.assert(
    fc.property(
      fc.record({
        visualization_type: fc.constantFrom('number_line', 'graph', 'area_model'),
        title: fc.string(),
        elements: fc.array(fc.record({
          type: fc.string(),
          // ... element properties
        })),
        caption: fc.string()
      }),
      (visualPlan) => {
        const output1 = visualRenderer.execute(visualPlan);
        const output2 = visualRenderer.execute(visualPlan);
        expect(output1.outerHTML).toEqual(output2.outerHTML);
      }
    ),
    { numRuns: 100 }
  );
});
```

### Unit Testing Focus

Unit tests should focus on:
- **Specific Examples**: Concrete scenarios that demonstrate correct behavior
- **Edge Cases**: Boundary conditions (empty inputs, maximum values, special characters)
- **Error Conditions**: Timeouts, invalid inputs, missing data
- **Integration Points**: Component interactions, data flow between layers

**Avoid writing too many unit tests** - property-based tests handle covering lots of inputs. Unit tests should be targeted and purposeful.

### Test Coverage by Layer

**Frontend Layer:**
- Unit tests: Component rendering, user interactions, state management
- Property tests: Exercise type rendering (Property 1), answer submission flows (Properties 3, 4)
- Integration tests: End-to-end user flows (start concept → answer exercise → receive feedback)

**AI Service Layer:**
- Unit tests: Prompt construction, output parsing, timeout handling, fallback logic
- Property tests: Structured output validation (Property 27), timeout enforcement (Property 28), sanitization (Property 31)
- Mock tests: LLM API responses (success, failure, timeout, malformed)

**Tool Layer:**
- Unit tests: Specific plan executions, validation logic, error handling
- Property tests: Deterministic rendering (Property 13), plan validation (Property 34), exercise selection logic (Properties 16, 17, 18)
- Determinism tests: Same input → same output verification

**Content Layer:**
- Unit tests: JSON loading, schema validation, fallback content retrieval
- Property tests: JSON structure validation (Property 36, 37), malformed JSON handling (Property 38)
- Data integrity tests: All exercises have required fields, fallbacks exist for all concepts

### Testing Priorities for Demo Reliability

**Critical Path Tests (Must Pass):**
1. Exercise loading and display
2. Answer validation and feedback
3. Fallback mechanisms (all AI features)
4. Session state persistence
5. Error handling (no crashes, no raw errors displayed)

**High Priority Tests:**
1. Adaptive exercise selection
2. Hint generation and progression
3. Visual rendering
4. Performance (validation < 2s, LLM < 5s)

**Medium Priority Tests:**
1. Explanation style variations
2. Misconception detection accuracy
3. Cache effectiveness
4. JSON validation

**Lower Priority Tests (Nice to Have):**
1. Visual realism and aesthetics
2. Explanation quality metrics
3. Advanced adaptive logic tuning

### Continuous Testing

**Pre-Demo Checklist:**
- [ ] All critical path tests passing
- [ ] Fallback mechanisms tested with simulated LLM failures
- [ ] Performance tests confirm < 2s validation, < 5s LLM responses
- [ ] No console errors in browser
- [ ] Session state persists across page refreshes
- [ ] All four algebra concepts have exercises loaded

**Automated Testing:**
- Run unit tests on every code change
- Run property tests nightly (longer execution time)
- Run integration tests before demos
- Monitor test coverage (aim for >80% on critical paths)

## Future-Ready Design

The platform is architected to support future enhancements without major rewrites.

### Extension Points

**1. Multi-Subject Support**
- Current: Algebra concepts only
- Extension: Add new concept tags (geometry, calculus, statistics)
- Impact: Content Layer only (add new JSON files)
- No changes needed: Frontend, AI Service, Tool Layer

**2. 3D Visualizations**
- Current: 2D visualizations (SVG/Canvas)
- Extension: Add 3D visualization types (WebGL, Three.js)
- Impact: Visual Renderer tool (add 3D rendering logic)
- No changes needed: AI Service interface (already outputs plans), Frontend (already displays rendered output)

**3. ML-Based Adaptive Engine**
- Current: Rule-based exercise selection
- Extension: Train ML model on student interaction data
- Impact: Exercise Selector tool (replace rule logic with model inference)
- No changes needed: Interface (same inputs/outputs), Frontend, AI Service

**4. Multi-LLM Support**
- Current: Single LLM API integration
- Extension: Support multiple providers (OpenAI, Anthropic, Cohere)
- Impact: AI Service (add provider abstraction layer)
- No changes needed: Frontend, Tool Layer (already consume structured plans)

**5. Backend Persistence**
- Current: Browser-local storage only
- Extension: Add backend API for cross-device sync
- Impact: Add backend service, modify session state management
- No changes needed: Core logic (already separates state from business logic)

**6. Institutional Features**
- Current: No authentication, no user management
- Extension: Add teacher dashboards, student progress tracking, class management
- Impact: Add authentication layer, backend services, new UI components
- No changes needed: Core learning logic (already separated from presentation)

### Design Principles for Extensibility

**Interface Stability:**
- AI Service outputs structured plans (not implementation-specific)
- Tool Layer accepts plans through defined interfaces
- Frontend consumes rendered outputs (not tied to rendering implementation)

**Separation of Concerns:**
- Content (JSON files) separate from logic (AI Service, Tools)
- Logic separate from presentation (Frontend components)
- Each layer can evolve independently

**Configuration Over Code:**
- Exercise data in JSON (add content without code changes)
- Fallback content in JSON (update fallbacks without code changes)
- Visualization types in schemas (add types without changing interfaces)

**Abstraction Layers:**
- LLM provider abstraction (swap providers without changing consumers)
- Tool interface abstraction (swap implementations without changing callers)
- Storage abstraction (swap localStorage for backend without changing logic)

### Migration Path to V2

**Phase 1: Enhanced Content**
- Add more algebra exercises
- Add more fallback content
- Improve visual quality
- No architecture changes

**Phase 2: Multi-Subject**
- Add geometry, calculus concepts
- Add subject-specific visualization types
- Extend content schemas
- Minimal code changes (Tool Layer only)

**Phase 3: Advanced AI**
- Integrate multiple LLM providers
- Add ML-based adaptive engine
- Improve misconception detection
- Moderate code changes (AI Service, Exercise Selector)

**Phase 4: Institutional Features**
- Add backend persistence
- Add authentication and user management
- Add teacher dashboards
- Significant code changes (new services, new UI)

**Phase 5: Production Readiness**
- Add monitoring and analytics
- Optimize performance
- Add accessibility features
- Add mobile support
- Comprehensive code changes (all layers)

Each phase builds on the previous without requiring rewrites, demonstrating the extensibility of the V1 architecture.

---

## Summary

This design describes a hackathon-scope adaptive learning platform that uses a single LLM for reasoning and planning, with deterministic tools handling execution. The architecture prioritizes demo reliability through graceful degradation, clear separation of concerns, and comprehensive fallback mechanisms. The platform is stateless, browser-local, and scoped to algebra in V1, with a future-ready design that supports multi-subject expansion, 3D visualizations, ML-based adaptation, and institutional features in later versions.

**Key Design Decisions:**
- One LLM, four roles (misconception analysis, hint generation, explanation planning, visual planning)
- AI plans, tools execute (clear separation between reasoning and execution)
- 2D visualizations only (SVG/Canvas, no 3D in V1)
- Rule-based adaptive logic (deterministic, testable, replaceable with ML later)
- Comprehensive fallbacks (pre-written content for all AI features)
- Browser-local storage (no backend, no auth, no PII)
- Property-based testing (100+ iterations per property, 44 properties defined)

This design balances hackathon constraints (limited time, demo focus) with engineering rigor (testability, extensibility, reliability).
