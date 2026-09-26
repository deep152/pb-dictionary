# Punjabi Dictionary Tools

My contributions to [The Punjabi App](https://punjabiapp.com/) — a free Punjabi dictionary and language-learning tool. We planned it together before I started coding it. I mainly did the dictionary-processing backend, while they handled the front end, hosting and deployment. Code written summer 2024–present, but it was never cleanly or consistently pushed here.

These are the standalone versions of the scripts I wrote and tested locally. My partner handles integration and deployment, and the versions running on the live site have since been updated. This repo reflects the cleaned-up state of my contributions; additional scripts and earlier iterations exist locally and will be added over time.

## What's in this repo

The existing dictionary tool needed more work to make it comprehensive and optimized for learning. These are the backend scripts I built toward that (all in `final/`):

| Script | What it does |
|---|---|
| `sentence_examples.py` | Picks the best real example sentences for a word from a sentence corpus. Each candidate is scored on how common its other words are (60%, against the 500-common-word list) and its length (40%, 7–12 words scores best). Sentences with ellipses are skipped, repeating the target word is penalized, and the top 5 are returned. |
| `synonyms.py` | Suggests related words for a Gurmukhi dictionary entry by comparing English definitions with TF-IDF cosine similarity, filtered to matching part of speech |
| `word_difficulty.py` | Scores how "easy" a word is using word frequency (against the 500-common-word list) and word length |
| `fuzzy.py` | Typo-tolerant lookup. Tries an exact match first, otherwise returns entries within 2 edits, closest first. The edit distance is a Levenshtein implementation I wrote with dynamic programming instead of importing a library, with a cheap length pre-filter to skip obvious non-matches. |
| `word_of_day.py` | Picks a random "easy" word not previously shown, tracked via a local history file |

## Data

- The full cleaned dictionary used by the live site has ~300K entries and isn't included here.
- `files/punjabi_university_sample.csv` is a ~1,000-entry sample of it (word, Shahmukhi transliteration, part of speech, English definition, source) so the scripts can be run.
- `files/500_common_punjabi_words.csv` is the ranked list of the 500 most common Punjabi words used by the scoring scripts. The frequency list comes from [learnpunjabi.org](https://www.learnpunjabi.org/statistics.html).
- `files/sam_arth_kosh_cleaned.json` is a cleaned Sam Arth Kosh synonym dictionary (2,533 headwords).
- The sentence corpus used by `sentence_examples.py` isn't included.

## Running

The scripts read their CSVs from the working directory:

```bash
cd files
python ../final/synonyms.py
```

`sentence_examples.py` also needs a sentence corpus CSV, which isn't in this repo.

## Sample output

[Sentence ranker runs](archive/sample_runs.md) — from an earlier iteration of the sentence ranker. It printed a single score per sentence; the current `sentence_examples.py` prints the final, commonness and length scores separately.

## Original planning notes

Some of the original notes from when I started one main function for the dictionary, kept because they're honestly a more accurate record of the actual thought process:

> **Task:** Synonyms and antonyms — generate a list of synonyms and antonyms for each word entry, taking the CSV as input and producing a list per word (maybe along with a score).
>
> **Initial plan:**
> - Parse the CSV (pandas)
> - Process definitions: extract and analyze the English definitions for each Punjabi word
> - Use NLTK's WordNet to find synonyms/antonyms for the English words in the definitions
> - Map back to the original Punjabi words
> - Approach could change — this was just laying a starting point
>
> **Things to take care of:**
> - Comma and semicolon splits in the definition column
> - Synonyms/antonyms need to match part of speech
> - Finish `clean_definition`, `split_definition`
>
> **Things to add:**
> - Basic error handling — `isGurmukhi()`, `getWordnetPOS()`, etc.
> - A scoring function to test accuracy
>
> **Open questions:**
> - Polysemy (a word meaning several different things) — how does this factor in?
> - Looked into [l3cube-pune/punjabi-sentence-bert-nli](https://huggingface.co/l3cube-pune/punjabi-sentence-bert-nli) as a possible NLP-based similarity approach instead of pure WordNet/TF-IDF

`archive/` holds earlier iterations and dead ends from this process, kept as an honest record rather than cleaned out.

## How the synonym finder evolved

My first version looked up synonyms one English word at a time in WordNet. It ran without errors, but some results were wrong. For the word meaning "constituency, electoral college," it returned the Punjabi words for "college" and listed the input word as its own synonym (see `archive/jda/trace`).

The cause was splitting definitions into single words. I tried matching short phrases (`archive/main w pos strat2.py`), which helped but wasn't enough. The final version (`final/synonyms.py`) compares whole definitions with TF-IDF cosine similarity, filters by part of speech, excludes the word itself, and keeps only matches above a 0.35 score. I also wrote `archive/jda/random_analyzer.py` to test random words and check score quality across the dictionary.

## Known limitations

- **POS handling is incomplete.** `map_pos()` correctly handles noun/verb/adjective/adverb/conjunct verb, but phrases, interjections, and pronouns all fall through to a shared `None` category, so they can get cross-matched with each other in the synonym finder. Low impact for common vocabulary, more noticeable at full dictionary scale (~300K entries).
- **TF-IDF is refit per query.** `synonyms.py` rebuilds the vectorizer against the whole corpus on every lookup instead of precomputing it once — fine for a demo, not for production at scale.
- **Fuzzy search is a linear scan.** `fuzzy.py` computes a pure-Python edit distance against every row (the length pre-filter helps), which is fine for the sample but would need a BK-tree or a library like `rapidfuzz` at ~300K entries. Edit distance also counts Unicode code points, not visible Gurmukhi letters, so a vowel sign counts as its own edit.
- **Sentence matching is exact-token.** `sentence_examples.py` splits on whitespace and matches whole tokens, so inflected forms, words with attached punctuation, and multi-word entries aren't matched. It also rescans the whole corpus on every lookup.
- **Word-of-the-day isn't date-locked** — it picks on every run rather than deterministically per calendar day, and the "easy word" heuristic is just length + definition length + common-word lookup rather than real difficulty data.
- None of the scripts handle the Shahmukhi transliteration column yet — only the Gurmukhi `word` column is searched.

## Requirements

```
pandas
scikit-learn
nltk
```

NLTK's `wordnet` and `stopwords` corpora need to be downloaded once:
```python
import nltk
nltk.download('wordnet')
nltk.download('stopwords')
```
