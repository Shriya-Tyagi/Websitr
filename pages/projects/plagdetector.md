# Plagiarism Detector

A minimal, secure plagiarism detector using TF-IDF and cosine similarity against live web sources, keeping all processing fully local with no data persistence or third-party transmission.

**Status:** usable · **Stack:** Python, scikit-learn, NLTK, BeautifulSoup4, duckduckgo-search · **Source:** [GitHub](https://github.com/Shriya-Tyagi/PlagiarismChecker)

---

## The problem

Most plagiarism checkers either phone home with your text, require a paid API, or only catch direct copy-paste. I wanted something that runs entirely on my machine, costs nothing, and is actually hard to fool with light paraphrasing.

## What it does

- Takes input text and generates sliding-window search queries from it
- Searches DuckDuckGo for matching sources and scrapes the top results
- Compares your text against each scraped page using character-level TF-IDF (5–8 grams) and cosine similarity
- Uses a coverage-based scoring model: measures what fraction of your text's chunks find a strong match in each source, rather than a single whole-document score
- Returns the top 5 matching URLs ranked by similarity

## What I learned

Character n-gram TF-IDF catches paraphrasing far better than word-level comparison — if someone rewrites sentences but keeps the structure, the character patterns still match. The coverage score also turned out to be more meaningful than a single similarity number: a document can have one heavily-matched paragraph and a low overall score, which the chunk approach surfaces correctly.

---

[← back to projects](/projects)
