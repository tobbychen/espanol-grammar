---
name: espanol-analyzer
description: Analyzes Spanish words, phrases, and full articles. For articles, it produces an English grammar and idiom analysis, a glossary with word-level analysis, and writes the result to a new markdown file.
---

# Spanish Text Analyzer

This skill analyzes Spanish text and explains it in English. It supports single words, phrases, and full Spanish articles.

## Workflow

1. **Identify input type:** Determine whether the user provided a single word, a short phrase, or a full Spanish article.
2. **Analyze the input:**
   - **If it is a single word:**
     1. Determine whether it is a noun, verb, adjective, adverb, or another part of speech.
     2. For nouns, consult `references/grammar_rules.md` and identify gender, number, and English meaning.
     3. For verbs, consult `references/verb_rules.md` and identify infinitive, tense, mood when relevant, person, number, and English meaning.
     4. For other word classes, identify the part of speech and give a concise English gloss.
   - **If it is a phrase:**
     1. Consult `references/grammar_rules.md`.
     2. Identify the phrase type and any important subclauses or grammatical structures.
     3. Provide a grammatical breakdown in English.
     4. Provide the English translation of the full phrase.
   - **If it is a full article:**
     1. Read the full text and infer a short descriptive abstract from the content.
     2. Create a new markdown file in the current working directory named `*-analysis.md`, where `*` is a concise lowercase hyphenated abstract that makes the topic obvious from the filename.
     3. Write the analysis to that file in English with this structure:
        - `# Analysis of <short topic>`
        - `## 1. Original Spanish Text`
        - `## 2. English Translation`
        - `## 3. Glossary`
        - `## 4. Grammar Analysis`
        - `## 5. Idioms and Expressions`
     4. In the glossary, include useful vocabulary, collocations, and short phrases from the article. For each entry, include:
        - the original Spanish term
        - the English translation
        - word-level analysis consistent with this skill's normal behavior
        - for verbs: infinitive, tense or form, person/number when applicable
        - for nouns: gender and number
        - for phrases: short structural note where useful
     5. In the grammar section, explain the key grammar patterns used in the article in English, using examples from the text.
     6. In the idioms and expressions section, explain idiomatic or high-value expressions in English and what they mean in context.
     7. Preserve the raw Spanish text and provide a full English translation in the file.
3. **Output requirement:** For article input, the markdown file is required, not optional. Also summarize the generated filename to the user.

## Output Notes

- Keep the explanation in English unless the user asks otherwise.
- Prefer concise but complete glossary entries.
- When choosing the filename abstract, use the article topic rather than generic names like `spanish-article-analysis.md`.
- If a file with the intended name already exists, choose a nearby descriptive variant instead of overwriting it silently.
