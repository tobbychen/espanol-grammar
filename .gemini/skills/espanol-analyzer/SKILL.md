---
name: espanol-analyzer
description: Analyzes Spanish words and phrases. For verbs, it identifies the infinitive, tense, and person. For nouns, it identifies the gender and number. For phrases, it analyzes the grammatical structure.
---

# Spanish Word and Phrase Analyzer

This skill analyzes Spanish text to determine its grammatical properties.

## Workflow

1.  **Identify Input Type:** Determine if the user has provided a single word or a phrase.
2.  **Analyze the Input:**
    *   **If it is a single word:**
        1.  Determine if it is a **noun** or a **verb**.
        2.  **For nouns:**
            *   Consult `references/grammar_rules.md`.
            *   Identify the gender (masculine/feminine) and number (singular/plural).
            *   Provide the English translation.
        3.  **For verbs:**
            *   Consult `references/verb_rules.md`.
            *   Identify the infinitive form (e.g., "-ar", "-er", "-ir").
            *   Identify the tense (e.g., Present, Preterite).
            *   Identify the person and number (e.g., 1st person singular).
            *   Provide the English translation.
    *   **If it is a phrase:**
        1.  Consult `references/grammar_rules.md`.
        2.  Identify the type of phrase (e.g., prepositional phrase, verb phrase).
        3.  Identify any subclauses.
        4.  Provide a grammatical breakdown of the phrase.
        5.  Provide the English translation of the full phrase.
3.  **Provide a clear, structured summary** of the analysis to the user.

## Output Format

### Word Analysis

<word>:
*   **Type**: <noun|verb>
*   **Gender**: <Masculine|Feminine>
*   **Number**: <Singular|Plural>
*   **Translation**: <English translation>

### Phrase Analysis

<phrase>:
*   **Type**: <phrase type>
*   **Grammatical Breakdown**: <breakdown>
*   **Translation**: <English translation>
