# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Spanish language learning project (`espanol`) that provides:
- Structured Spanish lessons (01_introduction.md through 06_analyzing_complex_sentences.md)
- Quiz files for practice (*_quiz.md, *_extended_quiz.md)
- English analysis of Spanish articles (*_analysis.md)
- A Spanish text analyzer skill for vocabulary and grammar study

## Key Commands

No build/test commands — this is a markdown-based learning project with no code compilation.

## Core Skill: espanol-analyzer

The main workflow lives in `espanol-analyzer/SKILL.md`. It analyzes Spanish text in three modes:

### Single Word
- Identifies part of speech (noun, verb, adjective, etc.)
- For nouns: gender, number, English meaning
- For verbs: infinitive, tense, mood, person, number

### Phrase
- Identifies phrase type and grammatical structure
- Provides English translation

### Full Article
- Creates `*-analysis.md` file with:
  - `# Analysis of <topic>`
  - `## 1. Original Spanish Text`
  - `## 2. English Translation`
  - `## 3. Glossary` (with word-level analysis: gender/number for nouns, infinitive/tense for verbs)
  - `## 4. Grammar Analysis`
  - `## 5. Idioms and Expressions`
- Filename uses topic-based hyphenated abstract (e.g., `greek-orthodox-lent-analysis.md`)

## Reference Files

- `references/grammar_rules.md` — noun gender/number, prepositional phrases, subclauses
- `references/verb_rules.md` — conjugation basics, present/preterite tenses
- `espanol-analyzer/references/preterite_stem_changes.md` — stem-changing verbs, spelling changes, irregular preterite stems

## Article Analysis Format

Analysis files follow this structure (see `greek-orthodox-lent-analysis.md`):
- Summary section
- Grammar Notes (key patterns with examples from text)
- Idioms and Expressions
- Glossary (word-level analysis)
- Raw Spanish Text
- English Translation
