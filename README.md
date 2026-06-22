# FitFindr — Starter Kit

This starter kit contains everything you need to begin Project 2 of the AI201 CodePath class. This project was done by:

## George Alvarado-Salinas

## URL for project

https://www.youtube.com/watch?v=qlFM4QWi0wQ

## What's Included

```
ai201-project2-fitfindr-starter/
├── data/
│   ├── listings.json          # 40 mock secondhand listings
│   └── wardrobe_schema.json   # Wardrobe format + example wardrobe
├── utils/
│   └── data_loader.py         # Helper functions for loading the data
├── planning.md                # Your planning template — fill this out first
└── requirements.txt           # Python dependencies
```

## Setup

**macOS / Linux:**

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

**Windows:**

```bash
python -m venv .venv
source .venv/Scripts/activate
pip install -r requirements.txt
```

Set your Groq API key in a `.env` file (get a free key at [console.groq.com](https://console.groq.com)):

```
GROQ_API_KEY=your_key_here
```

## The Mock Listings Dataset

`data/listings.json` contains 40 mock secondhand listings across categories (tops, bottoms, outerwear, shoes, accessories) and styles (vintage, y2k, grunge, cottagecore, streetwear, and more).

Each listing has: `id`, `title`, `description`, `category`, `style_tags`, `size`, `condition`, `price`, `colors`, `brand`, and `platform`.

Load it with:

```python
from utils.data_loader import load_listings
listings = load_listings()
```

## The Wardrobe Schema

`data/wardrobe_schema.json` defines the format your agent uses to represent a user's existing wardrobe. It includes:

- `schema`: field definitions for a wardrobe item
- `example_wardrobe`: a sample wardrobe with 10 items you can use for testing
- `empty_wardrobe`: a starting template for a new user

Load an example wardrobe with:

```python
from utils.data_loader import get_example_wardrobe
wardrobe = get_example_wardrobe()
```

## Tool Inventory

Your README submission must document each tool's name, inputs, and return value. **These must exactly match your actual function signatures in `tools.py`.** Your documented interfaces will be checked against your actual function signatures in `tools.py` — if the parameter count or types contradict what's in the code, you may not receive full credit for that tool.

---

### Function 1: `search_listings()`

#### Input / Output Contract

**Inputs:**

| Parameter     | Type            | Description                                                                                                         |
| ------------- | --------------- | ------------------------------------------------------------------------------------------------------------------- |
| `description` | `str`           | Keywords describing the item the user wants (e.g. `"vintage graphic tee"`). Stop words are filtered before scoring. |
| `size`        | `str \| None`   | Size string to filter by, or `None` to skip. Matching is case-insensitive (`"M"` matches `"S/M"`).                  |
| `max_price`   | `float \| None` | Maximum price (inclusive), or `None` to skip price filtering.                                                       |

**Output:** `list[dict]`

When **matches are found**, return a list of listing dicts sorted by keyword overlap score (highest first). Listings with a score of 0 are excluded. Each dict contains:

```python
{
  "id": str, "title": str, "description": str, "category": str,
  "style_tags": list[str], "size": str, "condition": str,
  "price": float, "colors": list[str], "brand": str | None, "platform": str
}
```

When **nothing matches**, return:

```python
[]   # empty list — does NOT raise an exception
```

**Effect on the agent loop:** If the returned list is empty, the agent sets `session["error"]` to a helpful message (explaining what didn't match and suggesting alternatives), then **returns immediately**. `suggest_outfit` and `create_fit_card` are never called.

---

### Function 2: `suggest_outfit()`

#### Input / Output Contract

**Inputs:**

| Parameter  | Type   | Description                                                                                          |
| ---------- | ------ | ---------------------------------------------------------------------------------------------------- |
| `new_item` | `dict` | A listing dict for the item the user is considering buying (same shape as `search_listings` output). |
| `wardrobe` | `dict` | A wardrobe dict with an `"items"` key containing a list of wardrobe item dicts. May be empty.        |

**Output:** `str`

When the **wardrobe has items**, return a non-empty string with 1–2 specific outfit combinations using named pieces from the wardrobe:

```python
"Layer this black windbreaker over your white cropped tank and black leggings..."
```

When the **wardrobe is empty**, return general styling advice for the item instead of raising or returning an empty string:

```python
"This jacket pairs well with neutral basics and sneakers for a casual streetwear look..."
```

When an **unexpected error occurs**, log a warning and return fallback styling tips based on the item's `colors`, `style_tags`, and `category` — never raises an exception.

**Effect on the agent loop:** `suggest_outfit` is designed to always return a non-empty string, so the loop continues to `create_fit_card` in all cases. If somehow an empty string is returned (unexpected), the agent sets `session["error"]` and **returns immediately**. Note: `create_fit_card` itself _can_ handle an empty outfit string gracefully — but the agent loop stops before reaching it as an extra defensive measure.

---

### Function 3: `create_fit_card()`

#### Input / Output Contract

**Inputs:**

| Parameter  | Type   | Description                                                                      |
| ---------- | ------ | -------------------------------------------------------------------------------- |
| `outfit`   | `str`  | The outfit suggestion string returned by `suggest_outfit()`.                     |
| `new_item` | `dict` | The listing dict for the thrifted item (same shape as `search_listings` output). |

**Output:** `str`

When **both inputs are valid**, return a 2–4 sentence LLM-generated caption suitable for Instagram or TikTok:

```python
"Found a sleek black windbreaker on Poshmark for $68 — perfect with my white crop and black leggings..."
```

When **`outfit` is empty, `None`, or whitespace-only**, log a warning and return a caption generated from `new_item` data alone (title, price, platform, colors, style_tags) — does NOT call the LLM with an empty outfit:

```python
"Just thrifted this Y2K Baby Tee on Depop for $18 and I can't stop thinking about it..."
```

When **`new_item` is also missing or malformed**, return:

```python
"Couldn't generate a fit card — item data was incomplete."
```

Never raises an exception.

**Effect on the agent loop:** `create_fit_card` is the final step — the agent loop always attempts it as long as `suggest_outfit` succeeded. Whatever string it returns (full caption, item-only fallback, or error message) is stored in `session["fit_card"]` and displayed to the user. The loop does not stop early based on `create_fit_card`'s output.

---

## Interaction Walkthrough

<!-- Walk through a complete interaction step by step: natural language query → each tool call (and why) → final fit card.
     Walk through this carefully — it's how graders follow your agent's reasoning without a live demo.
     Use a specific example — do not leave this as a template. -->

**User query:**
I'd like some athletic wear

**Step 1 — Tool called:**

- Tool: search_listings()
- Input: description was the query, size is None, and max_price is None.
- Why this tool: It's guaranteed to be the first tool called. With the description, it can extrapolate the size and price (which both don't get set in this example).
- Output: returns a list of dicts, of which, the first listing is sent to the next tool.

**Step 2 — Tool called:**

- Tool: suggest_outfit()
- Input: new_item is the 90s Track Jacket — Navy/White Stripe which can be found in the listings.json file. The wardrobe is the example one, which can be found in the wardrob_schema.json file.
- Why this tool: Once a thrift item is found, the LLM should return things it can be paired with if a wardrobe exists, or give back general pairings if no wardrobe is attached to this tool call.
- Output: Based on your existing wardrobe, here are two outfit combinations you can create with the thrifted 90s track jacket:

1. Pair the track jacket with your dark blue baggy straight-leg jeans and white ribbed tank top for a casual, nostalgic look. Add your chunky white sneakers for a sporty touch. This outfit captures the essence of 90s streetwear style.
2. Combine the track jacket with your khaki wide-leg trousers and black cropped zip hoodie for a chic, athleisure-inspired outfit. You can complete the look with your brown leather belt and black combat boots for a stylish contrast.

**Step 3 — Tool called:**

- Tool: create_fit_card()
- Input: outfit is thte suggestion string from the previous tool, suggest_outfit().
- Why this tool: This tool is called because if search_listings gets any item back, this will be called.
- Output: An instagram-like quote that you can share based on the outfit suggested.

**Final output to user:**

Just vibing with my new favorite throwback find - this retro navy/white stripe 90s track jacket ($45 on Poshmark). Paired it with my trusty dark blue jeans and white tee for a nostalgic streetwear look that's giving me 90s kid feels. Adding my chunky whites brings the whole aesthetic together - casual, sporty, and oh-so-authentic. #90sinspired #thriftlife #streetwear

---

## Planning Loop Explanation

A really short explanation of the Planning Loop goes like this:

1. User queries in FitFindr
2. search_listings extracts the pertinent information using an LLM, and either returns the item with the highest similarity score, or, doesn't return anything given the parameters (description, size, price).
3. If there is no item, the loop stops and lets the user know gracefully. If an item is found, we go into the next tool.
4. suggest_outfit is called, taking in a user's wardrobe (or not), as well as the item from search_listings.
5. If there is no wardrobe, You ask the LLM for general styling ideas with the item from before.
6. If there is a wardrobe, the LLM will return stylized ideas geared towards the item from the search_listing call, alongside your own wardrobe.
7. Create_fit_card is called, which will always have a new_item. if outfit is missing, the fit card returned (think instagram caption) will only talk about the new_item.
8. If the outfit is present, you'll get a small blurb of the new_item alongside some of your wardrobe articles.

---

## State Management Approach

The agent maintains a single session dict that accumulates state as tools run. Each tool receives only the fields it needs — no tool has access to the full session.

| Variable            | Type          | Set when                                                              | Passed to                                                              |
| ------------------- | ------------- | --------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `parsed`            | `dict`        | Before any tool call — extracted from the user's query by an LLM call | `search_listings` (as `description`, `size`, `max_price`)              |
| `search_results`    | `list[dict]`  | After `search_listings` returns                                       | Used to select `selected_item`                                         |
| `selected_item`     | `dict`        | After `search_listings` — top result picked from `search_results`     | `suggest_outfit` (as `new_item`) and `create_fit_card` (as `new_item`) |
| `wardrobe`          | `dict`        | Loaded once at session start — never modified                         | `suggest_outfit` (as `wardrobe`)                                       |
| `outfit_suggestion` | `str`         | After `suggest_outfit` returns                                        | `create_fit_card` (as `outfit`)                                        |
| `fit_card`          | `str`         | After `create_fit_card` returns                                       | Returned to the user                                                   |
| `error`             | `str \| None` | Set if any tool causes early termination                              | Surfaced to the user in place of normal output                         |

Data flows forward linearly — no tool reaches back into a previous result directly:

```
user query → _parse_query → parsed
parsed → search_listings → search_results → selected_item
selected_item + wardrobe → suggest_outfit → outfit_suggestion
outfit_suggestion + selected_item → create_fit_card → fit_card
```

Nothing is persisted between sessions. If the user starts a new conversation, the wardrobe is reloaded and all intermediate state is cleared.

---

## Error Handling and Fail Points

<!-- For each tool, describe the specific failure mode and what your agent does in response.
     This maps to the error handling section of the rubric (F5-C1). -->

| Tool              | Failure mode                                          | Agent response                                                                                                                                                                                                          |
| ----------------- | ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `search_listings` | Returns an empty list — no listings matched the query | Sets `session["error"]` with a message explaining what didn't match and suggesting alternatives (broader keywords, higher price, no size filter). Loop stops — `suggest_outfit` and `create_fit_card` are never called. |
| `suggest_outfit`  | Wardrobe is empty                                     | Returns general styling advice for the item instead of specific outfit combinations. Loop continues normally to `create_fit_card`.                                                                                      |
| `create_fit_card` | `outfit` is empty, `None`, or whitespace-only         | Logs a warning and generates a caption from `new_item` data alone (title, price, platform, colors, style_tags) without calling the LLM. Loop does not stop — the fallback caption is returned to the user.              |

---

## One concrete example from testing

I will leave the two tests I added below. The first one is what happens when you give suggest_outfit() an empty wardrobe. It asserts that what is returned is a String, and that the string isn't empty (showing that the tool doesn't crash, and does return something).

The same is true for the create_fit_card(), the call creates a non-empty string.

Both tests, along with the others, passed.

```python
def test_suggest_outfit_empty_wardrobe():
    item = {
        "title": "Black Windbreaker Jacket",
        "price": 68.0,
        "platform": "Poshmark",
        "condition": "Lightly used",
        "size": "S",
        "colors": ["black"],
        "style_tags": ["streetwear", "casual"],
    }
    result = suggest_outfit(item, get_empty_wardrobe())
    assert isinstance(result, str)
    assert len(result) > 0

def test_create_fit_card_empty_outfit():
    item = {
        "title": "Black Windbreaker Jacket",
        "price": 68.0,
        "platform": "Poshmark",
        "condition": "Lightly used",
        "size": "S",
        "colors": ["black"],
        "style_tags": ["streetwear", "casual"],
    }
    result = create_fit_card("", item)
    assert isinstance(result, str)
    assert len(result) > 0
```

---

## Spec Reflection

<!-- Answer both questions with at least 2–3 sentences each. -->

**One way planning.md helped during implementation:**
The planning.md file really helped when implementing the tools. It was really easy to get Claude to know what you wanted when it's stated in detail first. It also allows you to plan for worst case scenarios without having to go back and edit anything (thinking about code).

**One divergence from your spec, and why:**
I believe I added it in the planning loop, but I originally was using regex to get keywords from the user query, to then use against the listings. This was causing problems as a test query that included what you wanted AND what you wanted to style it with could give back listings that were more similar to the items you already owned. I changed my implementation when I realized this and tried updating it to the best of my ability in the planning.md file.

---

## AI Usage Log

| #   | Tool   | What I asked for                                                          | What it gave me                                                                                                         | What I changed                                                                                                                                                   |
| --- | ------ | ------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Claude | I asked for implementations of my tools which were defined in planning.md | Was given implementations for all three tools                                                                           | Did not have to change anything really. A little bit was changed in how parts of the description were used for the the score, but I think that's about it.       |
| 2   | Claude | Help with README inputting and formatting                                 | Table Templates, Tables filled in, or text to use that was defined elsewhere.                                           | I read text that is given back to make sure it states what was accurate in my case. I also filled in tables if they were not given back in a way that was clean. |
| 3   | Claude | Writing tests based on failure cases.                                     | Tests that followed the given structure from the previously written tests that hit the failure cases for tools 2 and 3. | I didn't change anything about them.                                                                                                                             |

---

## Where to Start

1. **Read `planning.md` and fill it out before writing any code.**
2. Verify the data loads correctly by running `python utils/data_loader.py`.
3. Build and test each tool individually before connecting them through your planning loop.

Your implementation files go in this same directory. There's no required file structure for your agent code — organize it however makes sense for your design.
