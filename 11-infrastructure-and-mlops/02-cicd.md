# CI/CD for LLM Applications

Deploying LLM applications requires adapting traditional CI/CD practices for AI-specific concerns like model evaluation, prompt testing, and quality gates.

## Table of Contents

- [LLM CI/CD Challenges](#llm-cicd-challenges)
- [Pipeline Architecture](#pipeline-architecture)
- [Testing Stages](#testing-stages)
- [Quality Gates](#quality-gates)
- [Deployment Strategies](#deployment-strategies)
- [Rollback Procedures](#rollback-procedures)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## LLM CI/CD Challenges

### What Makes LLM Deployments Different

| Traditional CI/CD | LLM CI/CD |
|-------------------|-----------|
| Binary tests (pass/fail) | Probabilistic evaluation |
| Fast tests | Slow, expensive evaluations |
| Deterministic outputs | Non-deterministic outputs |
| Code changes only | Prompt + model + data changes |
| Version control obvious | Prompt versioning complex |

### Change Types

| Change Type | Risk | Testing Required |
|-------------|------|------------------|
| Prompt text | Medium | Regression + quality eval |
| System prompt | High | Full evaluation suite |
| Model version | High | Comprehensive benchmark |
| RAG index | Medium | Retrieval + quality eval |
| Parameters (temp, etc) | Low-Medium | Quality sampling |

---

## Pipeline Architecture

### Full Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                       LLM CI/CD PIPELINE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐                                               │
│  │   Commit     │                                               │
│  │   Trigger    │                                               │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │   Validate   │ ─── Prompt syntax, config validation         │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │ Unit Tests   │ ─── Fast, deterministic tests                │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │  Golden Set  │ ─── Known input/output pairs                 │
│  │    Tests     │                                               │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │   LLM Eval   │ ─── Quality scoring, regression detection    │
│  │   (Sampled)  │                                               │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐                                               │
│  │ Quality Gate │ ─── Pass/fail based on thresholds            │
│  └──────┬───────┘                                               │
│         │                                                        │
│    ┌────┴────┐                                                  │
│    ▼         ▼                                                  │
│ ┌──────┐ ┌───────┐                                             │
│ │Canary│ │Blocked│                                             │
│ │Deploy│ │       │                                             │
│ └──┬───┘ └───────┘                                             │
│    │                                                            │
│    ▼                                                            │
│ ┌──────────────┐                                               │
│ │  Production  │                                               │
│ │  Monitoring  │                                               │
│ └──────────────┘                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Testing Stages

### Stage 1: Static Validation

```python
class PromptValidator:
    def validate(self, prompt_config: dict) -> ValidationResult:
        errors = []
        
        # Required fields
        if not prompt_config.get("system_prompt"):
            errors.append("Missing system_prompt")
        
        # Template syntax
        try:
            Template(prompt_config["user_template"]).substitute({})
        except KeyError:
            pass  # Expected for templates with variables
        except ValueError as e:
            errors.append(f"Invalid template syntax: {e}")
        
        # Token limits
        system_tokens = count_tokens(prompt_config.get("system_prompt", ""))
        if system_tokens > 4000:
            errors.append(f"System prompt too long: {system_tokens} tokens")
        
        return ValidationResult(
            valid=len(errors) == 0,
            errors=errors
        )
```

### Stage 2: Unit Tests

```python
class PromptUnitTests:
    def test_template_rendering(self):
        prompt = PromptTemplate(SYSTEM_PROMPT, USER_TEMPLATE)
        
        rendered = prompt.render(
            query="test query",
            context="test context"
        )
        
        assert "test query" in rendered
        assert "test context" in rendered
        assert len(rendered) < 10000  # Token limit
    
    def test_output_parsing(self):
        parser = OutputParser()
        
        valid_output = '{"answer": "test", "confidence": 0.9}'
        result = parser.parse(valid_output)
        assert result["answer"] == "test"
        
        invalid_output = "not json"
        with pytest.raises(ParseError):
            parser.parse(invalid_output)
```

### Stage 3: Golden Set Tests

```python
class GoldenSetRunner:
    def __init__(self, golden_set: list[dict]):
        self.golden_set = golden_set
    
    async def run(self, llm_client) -> TestResults:
        results = []
        
        for example in self.golden_set:
            response = await llm_client.generate(example["input"])
            
            # Exact match for deterministic outputs
            if example.get("exact_match"):
                passed = response == example["expected"]
            # Contains check for flexible outputs
            elif example.get("must_contain"):
                passed = all(
                    phrase in response 
                    for phrase in example["must_contain"]
                )
            # LLM judge for quality
            else:
                passed = await self.judge_quality(
                    response, example["expected"]
                )
            
            results.append(TestResult(
                input=example["input"],
                expected=example["expected"],
                actual=response,
                passed=passed
            ))
        
        return TestResults(
            total=len(results),
            passed=sum(1 for r in results if r.passed),
            failed=[r for r in results if not r.passed]
        )
```

### Stage 4: LLM Evaluation

```python
class LLMEvaluationStage:
    def __init__(self, eval_set: list[dict], sample_rate: float = 0.1):
        self.eval_set = eval_set
        self.sample_rate = sample_rate
        self.evaluator = LLMEvaluator()
    
    async def run(self, llm_client) -> EvalResults:
        # Sample for cost efficiency
        sample = random.sample(
            self.eval_set,
            int(len(self.eval_set) * self.sample_rate)
        )
        
        scores = []
        for example in sample:
            response = await llm_client.generate(example["input"])
            
            score = await self.evaluator.evaluate(
                query=example["input"],
                response=response,
                reference=example.get("reference"),
                criteria=["relevance", "accuracy", "helpfulness"]
            )
            scores.append(score)
        
        return EvalResults(
            sample_size=len(sample),
            avg_relevance=np.mean([s["relevance"] for s in scores]),
            avg_accuracy=np.mean([s["accuracy"] for s in scores]),
            avg_helpfulness=np.mean([s["helpfulness"] for s in scores])
        )
```

---

## Quality Gates

### Gate Configuration

```python
class QualityGate:
    def __init__(self, thresholds: dict):
        self.thresholds = thresholds
    
    def evaluate(self, results: dict) -> GateResult:
        failures = []
        
        # Golden set pass rate
        if results["golden_pass_rate"] < self.thresholds["golden_pass_rate"]:
            failures.append({
                "metric": "golden_pass_rate",
                "actual": results["golden_pass_rate"],
                "threshold": self.thresholds["golden_pass_rate"]
            })
        
        # Quality scores
        for metric in ["relevance", "accuracy", "helpfulness"]:
            if results.get(f"avg_{metric}", 0) < self.thresholds.get(metric, 0):
                failures.append({
                    "metric": metric,
                    "actual": results.get(f"avg_{metric}"),
                    "threshold": self.thresholds[metric]
                })
        
        # Regression detection
        if results.get("regression_detected"):
            failures.append({
                "metric": "regression",
                "details": results["regression_details"]
            })
        
        return GateResult(
            passed=len(failures) == 0,
            failures=failures
        )

# Example thresholds
QUALITY_THRESHOLDS = {
    "golden_pass_rate": 0.95,  # 95% of golden tests must pass
    "relevance": 4.0,          # Average score >= 4.0/5.0
    "accuracy": 4.0,
    "helpfulness": 3.5
}
```

---

## Deployment Strategies

### Canary Deployment

```python
class CanaryDeployer:
    def __init__(
        self,
        initial_percentage: int = 5,
        increment: int = 10,
        bake_time_minutes: int = 30
    ):
        self.initial_percentage = initial_percentage
        self.increment = increment
        self.bake_time = bake_time_minutes
    
    async def deploy(self, new_version: str):
        # Start canary
        await self.router.set_canary(new_version, self.initial_percentage)
        
        percentage = self.initial_percentage
        while percentage < 100:
            # Wait for bake time
            await asyncio.sleep(self.bake_time * 60)
            
            # Check canary health
            metrics = await self.get_canary_metrics(new_version)
            
            if not self.is_healthy(metrics):
                await self.rollback(new_version)
                raise CanaryFailedError(metrics)
            
            # Increment traffic
            percentage = min(100, percentage + self.increment)
            await self.router.set_canary(new_version, percentage)
        
        # Full rollout
        await self.router.promote_canary(new_version)
```

### Shadow Deployment

```python
class ShadowDeployer:
    async def shadow_test(
        self,
        new_version: str,
        duration_hours: int = 24
    ):
        # Run new version in shadow mode
        await self.enable_shadow(new_version)
        
        # Collect comparison data
        start = datetime.now()
        while datetime.now() - start < timedelta(hours=duration_hours):
            await asyncio.sleep(60)
            
            comparison = await self.compare_outputs()
            if comparison["divergence_rate"] > 0.1:
                await self.alert("High divergence in shadow test", comparison)
        
        # Analyze results
        return await self.generate_comparison_report(new_version)
```

---

## Rollback Procedures

### Automated Rollback

```python
class AutoRollback:
    def __init__(self, rollback_thresholds: dict):
        self.thresholds = rollback_thresholds
    
    async def monitor_and_rollback(self, version: str):
        while True:
            metrics = await self.get_live_metrics(version)
            
            # Check error rate
            if metrics["error_rate"] > self.thresholds["error_rate"]:
                await self.trigger_rollback(version, "error_rate_exceeded")
                return
            
            # Check latency
            if metrics["p99_latency"] > self.thresholds["p99_latency"]:
                await self.trigger_rollback(version, "latency_exceeded")
                return
            
            # Check quality (sampled)
            if metrics.get("quality_score", 5) < self.thresholds["quality_score"]:
                await self.trigger_rollback(version, "quality_degradation")
                return
            
            await asyncio.sleep(60)
    
    async def trigger_rollback(self, version: str, reason: str):
        previous = await self.get_previous_version()
        await self.router.rollback_to(previous)
        await self.alert(f"Auto-rollback from {version}: {reason}")
```

---

## Interview Questions

### Q: How do you test prompt changes before production?

**Strong answer:**

"I use a multi-stage testing pipeline:

**Stage 1: Static validation.** Syntax check, token limits, template errors. Fast and cheap.

**Stage 2: Unit tests.** Template rendering, output parsing, deterministic behavior. Still fast.

**Stage 3: Golden set tests.** Known input/output pairs that must pass. Catches obvious regressions.

**Stage 4: LLM evaluation.** Sampled evaluation using LLM-as-judge. Measures quality dimensions (relevance, accuracy). More expensive but catches subtle issues.

**Quality gates:** All stages must pass thresholds. Golden set > 95% pass rate, quality scores > 4.0/5.0.

**Deployment:** Canary at 5% traffic, bake for 30 minutes, monitor metrics, gradually increase.

The key insight is that LLM outputs are non-deterministic, so testing must be statistical. I cannot guarantee 100% correctness, but I can ensure quality stays within acceptable bounds."

### Q: What triggers should cause automatic rollback?

**Strong answer:**

"I configure multiple rollback triggers:

**Error rate:** If errors exceed 5% for 5 consecutive minutes, rollback. This catches outright failures.

**Latency:** If P99 latency exceeds SLA (e.g., 10s) for 10 minutes, rollback. This catches performance regressions.

**Quality score:** If sampled quality score drops below 3.5/5.0, rollback. This catches subtle quality degradation.

**User signals:** If negative feedback rate spikes 2x baseline, investigate and potentially rollback.

**Implementation:**
- Prometheus alerts trigger rollback script
- Automatic notification to team
- Rollback to last known good version
- Block further deploys until investigated

The key is fast detection and action. A bad prompt in production for 10 minutes is acceptable. For 10 hours is not."

---

## References

- ML Ops: https://ml-ops.org/
- LangSmith: https://docs.smith.langchain.com/

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **CI/CD** | Continuous Integration / Continuous Delivery — the practice of automatically building, testing, and deploying code on every change | Ensures software changes are validated and released safely and frequently |
| **Prompt Versioning** | Tracking prompt templates under version control, similar to how code is versioned | Allows rollback to a known-good prompt and traceability of quality changes |
| **Golden Set** | A curated collection of input/output pairs whose correct answers are known and fixed | Provides a fast, deterministic regression check for every pipeline run |
| **Quality Gate** | A mandatory checkpoint in the pipeline that blocks deployment if metrics fall below defined thresholds | Prevents regressions in LLM quality from reaching production users |
| **Canary Deployment** | A release strategy that exposes a new version to a small percentage of traffic first, then gradually increases it | Limits blast radius if a bad prompt or model version is deployed |
| **Shadow Deployment** | Running a new version in parallel with production to compare outputs without serving users from it | Safely evaluates new versions on real traffic with zero user impact |
| **Rollback** | Reverting a deployment to the previously known-good version when the new version causes problems | Quickly restores quality or stability after a bad release |
| **Regression Detection** | Automatically identifying when a new change causes quality or behavior to worsen compared to the baseline | Catches subtle prompt regressions before they affect all users |
| **LLM-as-Judge** | Using a separate LLM to automatically score another model's responses for quality dimensions like relevance and accuracy | Enables scalable automated quality evaluation without human review for every sample |
| **Static Validation** | Checking prompts and configs for syntax errors and missing fields without calling the LLM | Catches trivial errors instantly and cheaply before running expensive evaluations |
| **Probabilistic Evaluation** | Assessing LLM outputs by sampling and scoring distributions rather than checking exact matches | Handles the non-deterministic nature of LLM responses in testing |
| **Non-Determinism** | The property of LLMs where the same input can produce different outputs on different runs | Requires statistical testing approaches instead of exact binary pass/fail checks |
| **RAG Index** | A pre-built vector database index of documents that a retrieval-augmented system queries at runtime | The retrieval backbone for grounding LLM answers; changes to it need their own testing |
| **System Prompt** | The fixed instructions prepended to every conversation that define the LLM's behavior and persona | The highest-risk change type because it affects all downstream responses |
| **Template Rendering** | The process of substituting variables into a prompt template string to produce the final prompt | Ensures the prompt structure is correct before it is sent to the model |
| **Output Parsing** | Extracting structured data (e.g., JSON fields) from an LLM's free-text response | Makes LLM outputs programmatically usable and validates expected format |
| **Bake Time** | A waiting period after a canary deployment during which metrics are monitored before widening traffic | Gives time for latency, error rate, and quality issues to surface at small scale |
| **p99 Latency** | The 99th-percentile response time — 99% of requests complete faster than this value | A key SLA metric; a rollback trigger if it spikes after a new deployment |
| **Prometheus** | An open-source monitoring and alerting toolkit that scrapes and stores time-series metrics | Commonly used to watch error rates, latency, and quality scores and fire rollback alerts |
| **Divergence Rate** | The fraction of requests where a shadow version produces a meaningfully different output from the production version | Measures how much a new version's behavior differs before committing to it |
| **Evaluation Set (Eval Set)** | A larger collection of examples used to statistically measure model quality via sampling | Provides a broader quality signal than the golden set for LLM evaluation stages |
| **Sample Rate** | The fraction of the evaluation set used in a given pipeline run to control cost | Balances evaluation quality against the time and money spent on LLM calls |

---

*Previous: [LLM Infrastructure](01-llm-infrastructure.md) · Next: [AI Gateways and Model Routing](03-ai-gateways-and-model-routing.md)*
