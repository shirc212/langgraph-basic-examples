# Dana's Bakery — LangGraph Order System

## 1. Project Objective

Build a complete, executable Google Colab notebook named `danas_bakery.ipynb` that implements a deterministic bakery order-processing system using Python and LangGraph.

The system must process orders, validate inputs, check inventory, apply product substitutions, calculate discounts, handle delivery, generate friendly customer messages, and record the processing steps in chronological order.

**Technical constraints:**
- Use LangGraph's `StateGraph` with explicit nodes and conditional edges.
- Use deterministic Python only.
- Do not use LLMs, AI models, external AI APIs, or API keys.
- The notebook must run from a fresh Google Colab session.
- Install required dependencies inside the notebook.
- Display the actual compiled graph using `draw_mermaid_png()`.
- Include at least six test orders covering every possible decision path.

## 2. Input and Output

The graph must support this exact invocation:

```python
result = graph.invoke({
    "item": "babka",
    "quantity": 2,
    "day": "Thursday",
    "city": "Jerusalem"
})
```

Every result must contain these six keys:

- `status`: `"accepted"` or `"rejected"`.
- `item`: the product actually sold, or `None` if rejected.
- `delivery`: `"pickup"` or `"delivery"`.
- `total`: final amount in NIS, including delivery, or `0` if rejected.
- `message`: a polite customer-facing response.
- `steps`: a chronological list of the processing nodes visited.

Additional internal state fields are allowed.

## 3. Product Catalog and Inventory

Use the following fixed product catalog:

| Product | Price (NIS) | Stock |
|---|---:|---:|
| challah | 18 | 10 |
| babka | 45 | 0 |
| rugelach | 30 | 3 |
| sourdough | 25 | 4 |

Croissants are discontinued. They must not be accepted as valid products.

Treat these stock quantities as a fixed snapshot for each independent graph invocation. Do not persist inventory changes between test orders.

## 4. Order Validation

Create a `validate_order` node.

An order is valid only when:
- `item` is a string matching an active product.
- `quantity` is a positive integer, excluding booleans.
- `day` is a valid English weekday.
- `city` is a string or `None`.

Normalize whitespace and case for item, day, and city.

An empty or whitespace-only city means pickup.

For invalid orders:
- Set `status` to `"rejected"`.
- Set `item` to `None`.
- Set `delivery` to `"pickup"`.
- Set `total` to `0`.
- Generate a polite error message.
- Do not continue to inventory, pricing, or delivery processing.

Use a conditional edge to route valid and invalid orders.

## 5. Inventory and Substitution Rules

Create a `check_stock` node and a `try_substitute` node.

If enough stock exists for the requested item and full quantity, accept the original product.

Otherwise, attempt substitutions using this fixed chain:

- `babka` → `rugelach`
- `rugelach` → `sourdough`
- `sourdough` → `challah`

Challah has no further substitute.

Rules:
1. A substitute is acceptable only if its stock covers the entire requested quantity.
2. Do not mix different products within one order.
3. Preserve the original quantity.
4. Use the substitute's price.
5. Allow a maximum of two substitution attempts.
6. If both attempts fail, or no further substitute exists, reject the order.
7. Record each substitution attempt in `steps`.

Use conditional edges to route between stock checking, substitution, rejection, and pricing.

A substitution is applied automatically for this assignment. The customer message must clearly disclose the replacement. No interactive customer approval step is required.

## 6. Pricing and Discount Rules

Create a `calculate_price` node.

Calculate the baked-goods subtotal using the final selected product:

`subtotal = quantity * unit_price`

Apply a 15% discount only when:
- The baked-goods subtotal is strictly greater than 150 NIS.
- The order day is not Friday.

The earlier rule of 10% above 200 NIS is obsolete and must not be implemented.

The discount applies only to baked goods, never to delivery fees.

Use `Decimal` or an equivalent precise monetary calculation, and round the final amount to two decimal places.

## 7. Delivery Rules

Create a `handle_delivery` node.

- If `city` is empty or `None`, set delivery to `"pickup"` with no fee.
- If `city` is `"Jerusalem"`, set delivery to `"delivery"` and add 20 NIS.
- For any other city, keep the order accepted but change delivery to `"pickup"` with no fee.

The customer message must explain when delivery was changed to pickup because the city is outside Jerusalem.

The final total must include the delivery fee after applying the baked-goods discount.

## 8. Customer Messages

Create a `generate_message` node.

Generate warm, friendly messages using deterministic Python templates.

Do not use an LLM.

Messages must distinguish between:
- Successfully accepted original orders.
- Accepted orders with substitutions.
- Accepted delivery orders.
- Pickup orders.
- Orders changed to pickup because the city is outside Jerusalem.
- Invalid orders.
- Orders cancelled because of insufficient stock or failed substitutions.

Accepted-order messages must include the final product, quantity, total amount, and delivery method.

Rejected-order messages must explain the reason politely.

## 9. LangGraph State

Define a typed state using `TypedDict`.

Include at least these fields:

**Input fields:**
- `item`
- `quantity`
- `day`
- `city`

**Internal processing fields:**
- `original_item`
- `substitution_attempts`
- `subtotal`
- `discount`
- `delivery_fee`
- `rejection_reason`

**Required output fields:**
- `status`
- `item`
- `delivery`
- `total`
- `message`
- `steps`

Initialize optional internal fields safely for every invocation.

Ensure `steps` is a list that records the actual processing nodes and substitution attempts in chronological order.

Each invocation must start with a fresh steps list.

## 10. Graph Architecture

Use the following logical flow:

START → validate_order

If invalid:
validate_order → reject_order → generate_message → END

If valid:
validate_order → check_stock

If enough stock:
check_stock → calculate_price

If insufficient stock:
check_stock → try_substitute

From try_substitute:
- If a suitable substitute is found → calculate_price
- If the substitute is unavailable and another attempt is allowed → try_substitute
- If two attempts fail or no further substitute exists → reject_order

For accepted orders:
calculate_price → handle_delivery → generate_message → END

Use actual LangGraph nodes and conditional edges. Do not implement the entire workflow as one function.

Make sure the graph terminates on all possible paths.

## 11. Graph Visualization

In the notebook, display the actual compiled LangGraph structure using:

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

The visualization must match the implemented workflow, including conditional branches and substitution loops.

## 12. Tests

Include at least the following test cases, with clear expected results and assertions:

1. **Valid original order:** 2 challahs, Thursday, pickup. Expected total: 36 NIS.
2. **First substitution succeeds:** 2 babkas, Thursday, Jerusalem. Replace with 2 rugelach boxes. Expected total: 80 NIS.
3. **Second substitution succeeds:** 4 babkas, Thursday, pickup. Replace with 4 sourdough loaves. Expected total: 100 NIS.
4. **Substitution failure:** 5 babkas, Thursday, pickup. Reject after two failed substitutions.
5. **Invalid product:** pizza. Reject politely.
6. **Invalid quantity:** 0 or a negative number. Reject politely.
7. **Discount applied:** 10 challahs, Thursday, pickup. Subtotal 180 NIS, 15% discount, final total 153 NIS.
8. **Friday discount exclusion:** 10 challahs, Friday, pickup. Final total 180 NIS.
9. **Outside Jerusalem:** 2 challahs, Thursday, Tel Aviv. Accept as pickup, total 36 NIS, and explain the delivery restriction.
10. **Discontinued product:** croissant. Reject politely.
11. **Delivery fee excluded from discount:** 10 challahs, Thursday, Jerusalem. Final total 173 NIS.
12. **No available substitute:** 11 challahs, Thursday, pickup. Reject because challah has no substitute.

Add tests for the exact discount threshold of 150 NIS if feasible, malformed input types, and repeated invocations.

For every test, assert that all six required output keys exist.

Verify that the `steps` list matches the actual path and is ordered correctly.

## 13. Notebook Organization

Organize the notebook into clearly labeled sections:

1. Project overview
2. Dependency installation
3. Imports
4. Product catalog and inventory
5. State definition
6. Node implementations
7. Conditional routing functions
8. Graph construction and compilation
9. Mermaid graph visualization
10. Test orders and assertions
11. Summary of results

Use readable Python code with brief explanatory comments.

## 14. Deliverables

Create the following files in the current GitHub repository:

- `prompt.md` — this technical specification.
- `danas_bakery.ipynb` — the complete executable Colab notebook.

Do not create Instagram integration.

Do not request an API key.

Do not stop after generating code. Review the implementation for logical errors, run the tests where the environment permits, and correct any failures.

If execution is not possible in the current environment, state that clearly instead of claiming the tests passed.

## 15. Acceptance Criteria

The implementation is complete only if:

- The notebook runs in a fresh Google Colab session.
- The graph uses LangGraph.
- The graph drawing is displayed.
- No LLM or API key is required.
- All business rules are implemented.
- Every invocation returns the six required keys.
- Substitution attempts never exceed two.
- Discounts and delivery fees are calculated correctly.
- All tests pass.
- The processing history accurately reflects the graph path.