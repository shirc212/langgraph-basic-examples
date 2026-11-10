# Dana’s Bakery — LangGraph

A deterministic bakery-order workflow implemented as a Google Colab notebook. No LLM calls or API keys are required.

## Run in Google Colab

1. Download `dana_bakery-class.ipynb` from this repository.
2. Open [Google Colab](https://colab.research.google.com/) and choose **File → Upload notebook**.
3. Select the notebook, then choose **Runtime → Run all** in a fresh Python runtime.

The first code cell installs LangGraph 1.2.20. Internet access is needed for installation and the `draw_mermaid_png()` graph renderer. The notebook includes an embedded graph drawing and saved local outputs.

## Files

- [Notebook](dana_bakery-class.ipynb): implementation, graph drawing, example invocation, tests and submission checklist.

New filenames use `-class` according to repository instructions. The assignment calls the prompt `prompt.md`; this project uses `prompt-class.md`.

## Behavior

The graph validates orders, checks stock, tries up to two alternatives, calculates prices, chooses delivery or pickup, and produces a warm template reply with an ordered processing diary.

- Menu: challah 18, babka 45, rugelach box 30, sourdough 25 shekels.
- Each invocation independently starts with 10 challahs, 0 babkas, 3 rugelach boxes and 4 sourdough loaves.
- Substitution order: babka → rugelach → sourdough → challah. The entire quantity must be available.
- Goods above 150 shekels receive 15% off, except Friday. Delivery is never discounted.
- Jerusalem delivery costs 20 shekels. Blank city means free pickup; other cities switch to free pickup with an explanation.
- Invalid orders and discontinued croissants are rejected.

```python
result = graph.invoke({
    "item": "babka",
    "quantity": 2,
    "day": "Thursday",
    "city": "Jerusalem",
})
```

Expected: `status="accepted"`, `item="rugelach"`, `delivery="delivery"`, `total=80.0`, plus `message` and ordered `steps`.

## Validation

Runner: **Google Colab → Runtime → Run all**.

Local verification passed all 33 order cases, exact pricing boundaries, repeated-call checks, trace-order assertions and every graph edge using Python 3.12 and LangGraph 1.2.20. The graph PNG was rendered and visually checked. Local checks executed the notebook Python cells directly after installing dependencies; they were not a Colab run.

Fresh-Colab validation remains pending. Before submission, run all cells in Colab, save outputs, share the notebook with view access for anyone with the link, and check both the repository and Colab links while signed out.
