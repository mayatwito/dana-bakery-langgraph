# Dana's Bakery — LangGraph Order Processing System

## 1. Project Objective

Develop a complete, beginner-friendly order-processing system for Dana's Bakery using Python and LangGraph. Translate informal business requirements into a structured, deterministic workflow that validates customer orders, checks inventory, handles substitutions, calculates discounts and delivery fees, and generates customer-friendly responses. The implementation must run in Google Colab.

**Technical constraints:** Use Python and LangGraph; deterministic business rules only; no LLMs, OpenAI API, or external AI services; no API keys or paid services; keep code readable, modular, and suitable for students learning LangGraph; include English Markdown explanations throughout the notebook.

Create `prompt.md` (this complete technical specification) and `dana_bakery.ipynb` (the working Google Colab notebook implementing the system). Do not create Git commits or push files to GitHub until the implementation has been reviewed and tested.

## 2. Customer Order Input

Each order contains `item` (e.g. `"babka"`), `quantity` (e.g. `2`), `day` (e.g. `"Thursday"`), and `city` (e.g. `"Jerusalem"`). The compiled graph must support:

```python
result = graph.invoke({"item": "babka", "quantity": 2, "day": "Thursday", "city": "Jerusalem"})
```

The graph processes the order and returns a structured result. All required output fields must be present, including for rejected orders.

## 3. Menu, Prices, and Inventory

| Product | Unit Price (ILS) | Available Quantity |
|---|---:|---:|
| `challah` | 18 | 10 |
| `babka` | 45 | 0 |
| `rugelach` | 30 | 3 |
| `sourdough` | 25 | 4 |

Croissants are discontinued and must not be accepted as a valid product. Products are sold at current menu prices. Orders must be fulfilled in their entire requested quantity; partial fulfillment is not allowed. Substitutions use the substituted product's price. Define menu and inventory as clearly named Python constants or dictionaries.

## 4. Order Validation

Validate before checking inventory or calculating prices. Reject when the product is not on the active menu; quantity is zero, negative, not a positive integer; or required information is missing or invalid. Empty city is valid and indicates pickup. Normalize product, city, and day by removing unnecessary whitespace and handling capitalization differences (`Babka`, `babka`, and `BABKA` are equivalent).

On validation failure: set `status="rejected"`, `total=0`, generate a polite explanation, record validation and rejection steps, and do not continue to inventory or pricing.

## 5. Inventory Checking and Product Substitution

First check requested product availability for the full requested quantity. If insufficient, attempt substitutions in this fixed sequence: **babka → rugelach → sourdough → challah**.

Rules: attempt at most two alternatives; follow the sequence; check each alternative against the full unchanged quantity; price the selected alternative; inform the customer about substitutions; if two alternatives fail, reject and cancel; if no next alternative exists, cancel; never attempt a third alternative.

Example A: babka, quantity 2, Thursday, Jerusalem. Babka stock 0; rugelach stock 3 is sufficient. Substitute two rugelach. Total `2 × 30 + 20 = 80 ILS`. Accept.

Example B: babka, quantity 4, Wednesday, empty city. Rugelach stock 3 is insufficient; sourdough stock 4 is sufficient. Substitute four sourdough. Total `4 × 25 = 100 ILS`. Accept for pickup.

Example C: babka, quantity 5, Monday, Jerusalem. Rugelach stock 3 and sourdough stock 4 are insufficient. Reject after two alternatives; do not try challah.

## 6. Discount Calculation

Apply the final policy established by Avi, responsible for financial decisions. The superseded policy of 10% discount above 200 ILS must not be implemented.

Apply a **15% discount** only when baked-goods subtotal is strictly greater than 150 ILS and the order is not on Friday. Exactly 150 does not qualify. Friday orders receive no discount. Discount baked goods only; never discount delivery fees. Determine eligibility from the selected product's price, including substitutions. Example: 10 challah on Thursday: subtotal 180, discount 27, after discount 153, delivery 20, total 173 ILS. On Friday: subtotal 180, discount 0, delivery 20, total 200 ILS. Use clear calculations and avoid unnecessary floating-point rounding errors.

## 7. Delivery and Pickup Rules

| Condition | Fulfillment Method | Fee |
|---|---|---:|
| City is Jerusalem | Delivery | 20 ILS |
| City is empty | Pickup | 0 ILS |
| City is outside Jerusalem | Pickup | 0 ILS |

Do not reject otherwise-valid orders outside Jerusalem. Automatically switch those orders to pickup and clearly explain this in the customer message. Add delivery fees only after calculating baked-goods discount.

## 8. LangGraph State Design

Use `StateGraph` and a clearly defined `TypedDict` state. Required inputs: `item`, `quantity`, `day`, `city`. Every completed invocation returns these six fields:

| Field | Required Value |
|---|---|
| `status` | `"accepted"` or `"rejected"` |
| `item` | Product selected by workflow |
| `delivery` | `"pickup"` or `"delivery"` |
| `total` | Final payable amount including delivery |
| `message` | Customer-facing response |
| `steps` | Ordered history of executed workflow steps |

Useful internal fields: original item, alternative attempts, subtotal, discount, delivery fee, rejection reason. Initialize safely so invocation needs only the four input fields.

Rejected conventions: status rejected; total zero; delivery pickup; message explains rejection; steps contain actual executed steps. Preserve the last product evaluated in `item`. If validation fails before a product is evaluated, preserve submitted product value.

## 9. LangGraph Nodes

Implement separate functions for `validate_order`, `check_stock`, `choose_alternative`, `reject_order`, `calculate_discount`, `check_delivery`, `calculate_total`, and `generate_message`. Use conditional routing.

Required routing:
- Validation: valid → `check_stock`; invalid → `reject_order`.
- Inventory: sufficient → `calculate_discount`; insufficient with permitted alternative → `choose_alternative`; otherwise → `reject_order`.
- Alternative selection selects next alternative, increments attempt counter, returns to `check_stock`.
- Successful path: `calculate_discount → check_delivery → calculate_total → generate_message → END`.
- Rejected path: `reject_order → END`.
- All paths terminate; substitution must not loop infinitely.

## 10. Step History and Reducers

Implement chronological accumulation with LangGraph state reducer:

```python
from typing import Annotated
import operator
steps: Annotated[list[str], operator.add]
```

Each node appends its own name. Never overwrite history. Repeated nodes appear for every execution. Example one-substitution path:

```python
["validate_order", "check_stock", "choose_alternative", "check_stock", "calculate_discount", "check_delivery", "calculate_total", "generate_message"]
```

History reflects actual executed path, including rejected orders, and must not include nodes never executed.

## 11. Customer Messages

Use deterministic Python string templates, not an LLM. Accepted messages include confirmation, product and quantity, substitution details if applicable, discount information if applicable, delivery or pickup details, and final amount. Rejected messages politely explain why; distinguish invalid orders from inventory cancellations; avoid misleading payment information. Keep messages warm, clear, and professional.

## 12. Google Colab Notebook Structure

Create `dana_bakery.ipynb` with English Markdown headings and short explanations, organized into:
1. Project Introduction — purpose and workflow.
2. Environment Setup — install LangGraph.
3. Business Rules — menu, inventory, discounts, delivery policies.
4. State Definition — TypedDict state.
5. Graph Nodes — all processing functions.
6. Conditional Routing — routing decisions.
7. Graph Construction — connect nodes and compile graph.
8. Graph Visualization — display complete workflow.
9. Test Orders — execute multiple realistic scenarios.
10. Results and Validation — outcomes and correctness checks.

Use required visualization method `graph.get_graph().draw_mermaid_png()` and display its image inside Google Colab. The diagram must be the actual LangGraph drawing and show conditional branches, substitution loops, rejection and successful completion paths. Do not substitute a manually created diagram.

## 13. Test Cases and Expected Results

Include at least six independent test orders, and more as needed to cover all conditional paths. Minimum scenarios:

| Test | Item | Qty. | Day | City | Expected Result |
|---|---|---:|---|---|---|
| 1 | challah | 2 | Monday | Jerusalem | Accepted, delivery, 56 ILS |
| 2 | babka | 2 | Thursday | Jerusalem | First substitution to rugelach, 80 ILS |
| 3 | babka | 4 | Wednesday | Empty | Second substitution to sourdough, 100 ILS |
| 4 | babka | 5 | Monday | Jerusalem | Rejected after two failed alternatives |
| 5 | challah | 10 | Thursday | Jerusalem | Accepted, 15% discount, 173 ILS |
| 6 | challah | 10 | Friday | Jerusalem | Accepted, no discount, 200 ILS |
| 7 | challah | 0 | Monday | Jerusalem | Rejected, invalid quantity |
| 8 | pizza | 2 | Tuesday | Jerusalem | Rejected, invalid product |
| 9 | croissant | 2 | Tuesday | Jerusalem | Rejected, discontinued product |
| 10 | challah | 2 | Tuesday | Tel Aviv | Accepted, switched to pickup, 36 ILS |
| 11 | challah | 2 | Tuesday | Empty | Accepted, pickup, 36 ILS |
| 12 | challah | 11 | Wednesday | Jerusalem | Rejected, insufficient stock and no alternative |

Each test is independent and uses initial inventory. Rejected orders total 0 ILS.

Automated Python assertions verify: required six output fields; correct status; selected/last-evaluated product; delivery method; total; no more than two substitution attempts; rejected total zero; actual step path; all graph paths terminate. Print readable summary of each test with input, result, pass/fail. A test passes only if assertions validate expected behavior, not merely if invocation returns.

## 14. Implementation Assumptions and Scope

Document consistently in notebook:
- Each `graph.invoke()` uses original inventory; no persistent inventory reduction.
- Only positive integers accepted; reject negatives, zero, fractions, booleans, nonnumeric strings.
- Empty city means pickup. Missing/invalid item, quantity, or day rejects.
- Maximum two alternatives per order; if chain ends earlier, cancel.
- Dana requested AI-generated personal responses, but technical spec prohibits LLM; use deterministic templates.
- Excluded: Instagram posting, customer account management, online payments, permanent inventory databases, external delivery integrations, LLM-generated messages. These are outside assignment scope.

## 15. Final Verification and Deliverables

Before completion: verify valid `.ipynb` structure and Colab compatibility; LangGraph implements actual workflow; business rules are implemented; no LLM/API key required; visualization calls `draw_mermaid_png()`; tests cover graph paths; every invocation returns six fields; no third substitution occurs; notebook headings and explanations are English.

Preserve this complete specification in `prompt.md`. If testing reveals problems and prompt is revised, maintain a short **Prompt Revision History** section in `prompt.md` documenting changes and reasons. Do not invent revisions that did not occur. Do not commit or push to GitHub yet.

Once ready, summarize files created, graph structure, business rules, tests, validations, and issues still requiring Google Colab testing. Distinguish local checks from unexecuted tests. Wait for instructions before committing or pushing.
