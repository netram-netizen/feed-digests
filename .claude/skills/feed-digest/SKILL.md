---
name: feed-digest
description: Fetch latest blog posts from RSS feeds and write an overview with reading recommendations.
argument-hint: "[time period, e.g. 'last two weeks', 'since 8am CET', '4 hours']"
disable-model-invocation: true
allowed-tools: Bash(uv run *), Read
---

# Feed Digest

Fetch the latest blog posts and present an overview with reading recommendations.

## Steps

1. Convert the time period argument into an `--hours` value for `fetch.py`. If no argument is given, use the default (36 hours).
   - The argument is: $ARGUMENTS
   - Examples: "last two weeks" → `--hours 336`, "since 8am CET" → calculate hours from that time to now, "4 hours" → `--hours 4`
2. Run `uv run fetch.py --hours <N>` from the project root.
3. Read the output. When fewer than 20 posts, `fetch.py` automatically fetches full articles via Playwright and saves them as markdown files in `.articles/`. Each post that was successfully fetched shows a `Full article: .articles/NNN.md` path.
4. For each post, write a summary:
   - **With full article** (path shown): Read the `.articles/NNN.md` file and write a 2–4 sentence briefing covering key arguments and takeaways.
   - **Without full article (≥ 20 posts or fetch failed)**: Write a one-line summary based on the title and summary text.
5. Recommend which posts are most worth reading, considering:
   - Novelty and insight (original ideas over announcements)
   - Practical relevance for a software engineer
   - Depth of content (prefer substantial articles over short updates)
6. While reading full articles, identify links within them that look interesting (e.g. referenced papers, tools, projects, deep-dives) but that are NOT just navigation, social media, or self-referential links. For the most promising ones (up to 5 total across all articles), scrape them with `uv run scrape.py <url> .articles/link-NNN.md` and include a brief note about what they contain.
7. Present the output as:
   - **New Posts** — grouped by source, each with a summary and link
   - **Recommended Reading** — your top picks (up to 5) with linked titles and a sentence on why each is worth reading
   - **Interesting Links** — notable links found within articles, with a one-line description of each
8. Ask if the user wants to see a different time window.
