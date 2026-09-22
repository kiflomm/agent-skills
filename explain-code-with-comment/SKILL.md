---
name: explain-code-with-comment
description: Add `[explain]`-tagged comments that clarify complex code, function roles, or file purpose—never change code.

---
# Explain Code With Comments

Add `[explain]` block comments that make the code easier for a learner to understand.

## Rules

* **Never change the code.** Do not modify, remove, reorder, or refactor any code.
* **Do not modify existing comments.**
* Only add new comments using the `[explain]` tag.
* Prefer **block comments** over inline comments.
* Start with a block comment explaining the **file's purpose and overall flow**.
* Add comments before important classes, functions, sections, or complex logic.
* Explain **what the code does, why it exists, and how it connects to the surrounding code**.
* Explain the execution flow step-by-step when the code contains multiple operations.
* Briefly explain unfamiliar programming/framework concepts when they are important to understanding the code.
* Do not explain obvious lines or repeat what the code already clearly says.
* Do not invent behavior or assumptions that cannot be determined from the code.
* Keep explanations concise, clear, and learner-friendly.

## Output

Return the **complete original file with only `[explain]` comments added**.

