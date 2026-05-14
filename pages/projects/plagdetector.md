# Plagiarism Detector

A minimal, secure plagiarism detector using TF-IDF and cosine similarity against live web sources, keeping all processing fully local with no data persistence or third-party transmission.

**Status:** usable · **Stack:** Python, scikit-learn, NLTK, BeautifulSoup4, duckduckgo-search · **Source:** [GitHub](https://github.com/Shriya-Tyagi/PlagiarismChecker)

---

## The problem

Existing plagiarism detection tools typically transmit input text to third-party servers, require paid API access, or rely on exact string matching that fails against paraphrased content. The goal was a self-contained solution that operates entirely locally, incurs no recurring cost, and remains robust to surface-level rewriting.

## What it does

- Accepts input text and generates sliding-window search queries from it
- Queries DuckDuckGo for candidate sources and scrapes the top results
- Compares input text against each scraped page using character-level TF-IDF (5–8 grams) and cosine similarity
- Applies a coverage-based scoring model: measures what fraction of the input's chunks find a strong match within each source, rather than computing a single whole-document score
- Returns the top 5 matching URLs ranked by similarity

## What I learned

Character n-gram TF-IDF generalises better than word-level comparison for detecting paraphrasing — structural similarity at the character level persists even when vocabulary is substituted. The coverage-based score also proved more informative than a single similarity coefficient: a source with one heavily-matched passage and low overall overlap is surfaced correctly under the chunk approach, whereas a global score would suppress it.

## Demo

![Ketchup sample run — terminal output showing ranked source matches](./assets/ketchup-run.png)

![Code and output side by side, with top match highlighted](./assets/code-output.png)
---

[← back to projects](/projects)
