# 🗺️ AgentsVille Trip Planner

An AI-powered travel itinerary planner that demonstrates core agentic AI patterns:
**Chain-of-Thought prompting**, **LLM-as-Judge evaluation**, and **ReAct loops with tool use**.

Built as a hands-on exercise in the *Agentic AI — Da zero alla produzione* course.

---

## What it does

Given a group of travelers with different interests, a destination, date range, and budget, the system:

1. Fetches weather forecasts and available activities for the trip dates
2. Generates a day-by-day itinerary using a Chain-of-Thought prompt
3. Evaluates the itinerary against 6 criteria (dates, budget, hallucinations, interests, weather compatibility)
4. Runs a ReAct agent that iteratively revises the plan using tools until all evaluations pass
5. Narrates the final trip as a fun audio summary

---

## Project structure

```
.
├── project_starter.ipynb     # notebook 
├── project_lib.py            # Helper classes and mock API data
└── README.md
```

---

## Key concepts implemented

### 1. Pydantic data validation
Vacation details, activities, and the final travel plan are all typed with Pydantic models. This enforces structure at every boundary between Python code and LLM output.

```python
class VacationInfo(BaseModel):
    travelers: List[Traveler]
    destination: str
    date_of_arrival: datetime.date
    date_of_departure: datetime.date
    budget: int
```

### 2. Chain-of-Thought (CoT) prompting — `ItineraryAgent`
The system prompt forces the model to reason step-by-step in an `ANALYSIS:` section before producing the final JSON output. This reduces hallucinations and budget errors significantly.

```
ANALYSIS:
* Day 1 (2025-06-10, clear): available activities are [...], Yuri matches tennis...
* Budget check: 20 + 15 = 35, within budget of 130.

FINAL OUTPUT:
```json
{ "city": "AgentsVille", ... }
```

### 3. Context injection
Weather and activity data is embedded directly into the system prompt as JSON. Without this, the model invents activities that don't exist.

```python
ITINERARY_AGENT_SYSTEM_PROMPT = f"""
...
## Context
### Weather Forecast
{json.dumps(weather_for_dates, indent=None)}

### Available Activities
{json.dumps(activities_for_dates, indent=None)}
"""
```

### 4. LLM-as-Judge — `eval_activities_and_weather_are_compatible`
A fast, cheap model (`gpt-4.1-nano`) evaluates whether each activity is suitable for the day's weather. The prompt forces a binary output (`IS_COMPATIBLE` / `IS_INCOMPATIBLE`) with reasoning first.

```
REASONING:
The activity is held entirely outdoors with no indoor backup. Thunderstorm conditions make it incompatible.

FINAL ANSWER:
IS_INCOMPATIBLE
```

### 5. ReAct loop — `ItineraryRevisionAgent`
The revision agent follows a strict `THOUGHT → ACTION → OBSERVATION` cycle. Python parses the `ACTION:` block, executes the tool, and returns an `OBSERVATION:` message. The loop ends when the agent calls `final_answer_tool`.

```
THOUGHT:
I need to check the current itinerary for issues. I'll run the evaluations first.

ACTION:
{"tool_name": "run_evals_tool", "arguments": {"travel_plan": {...}}}
```

---

## Evaluation criteria

| Function | What it checks |
|---|---|
| `eval_start_end_dates_match` | Dates in the plan match the requested dates |
| `eval_total_cost_is_accurate` | `total_cost` equals the sum of all activity prices |
| `eval_total_cost_is_within_budget` | Total cost does not exceed the budget |
| `eval_itinerary_events_match_actual_events` | No hallucinated activities — all IDs exist in the API |
| `eval_itinerary_satisfies_interests` | Each traveler has at least one matching activity |
| `eval_activities_and_weather_are_compatible` | No outdoor-only events on rainy/thunderstorm days |
| `eval_traveler_feedback_is_incorporated` | Traveler feedback (≥2 activities/day) is respected |

---

## Tools available to the ReAct agent

| Tool | Purpose |
|---|---|
| `calculator_tool` | Evaluates math expressions — avoids LLM arithmetic errors |
| `get_activities_by_date_tool` | Fetches available activities for a given date |
| `run_evals_tool` | Runs all evaluation functions on a travel plan |
| `final_answer_tool` | Signals the end of the ReAct loop and returns the final plan |

---

## Setup

### Requirements

```bash
pip install json-repair numexpr openai pandas pydantic python-dotenv
```
 
### Models

The notebook uses `gpt-4.1-mini` by default. If unavailable on your account, switch to:

```python
MODEL = OpenAIModel.GPT_4O_MINI  # universally available fallback
```

---
 
