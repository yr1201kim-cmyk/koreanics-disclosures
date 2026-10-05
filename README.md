# Koreanics technical disclosures

This repository describes, in technical detail, the methods [Koreanics](https://koreanics.com) uses to track Korean legislation, build its English database of Korean laws and run its publishing system. We publish these descriptions so that the methods are on the public record with a date, and remain free for anyone to use.

Version 1, published October 5, 2026. Each version is a commit in this repository, and the Koreanics daily [record seal](https://koreanics.com/seal/) timestamps the production code that implements these methods. "In operation since" dates are the earliest dates in our code history (which begins on September 25, 2026); some methods are older.


## 1. Tracking bill stages from official records
*In operation since September 25, 2026 (hourly since September 27, 2026).*
1. Every hour, the system pulls four official National Assembly Open API services: the list of member-sponsored bills, the bill receipt list (which covers government and committee bills), the committee review list and the subcommittee review list.
2. Rows from the four services are merged per bill before anything is saved. Saving row by row would record false intermediate stage moves the first time a bill is loaded.
3. Where a bill has several subcommittee rows, exactly one is used: the one with the latest processing date, with ties going to the row that has a recorded result. Mixing fields from different rows makes the stage flicker from day to day.
4. A deterministic, versioned rule set assigns each bill one stage. The checks run in this order: a re-vote after a presidential veto (passed means law, failed means ended); the plenary result (passed as is or amended; merged into a committee alternative; discarded, withdrawn, rejected or not referred); the committee result (absorbed or ended overrides the date checks); then dates (judiciary review cleared, committee decision, subcommittee decision; a subcommittee result of "absorbed" or "discarded" keeps the bill at the committee stage, because it becomes final only at the committee vote); then committee referral; otherwise filed.
5. For vetoed bills, the original record and the re-vote record from the full-history service are merged into one lifecycle. Re-vote records are recognised by proposer and identifier patterns, which differ between Assembly terms.
6. A stage event (from stage, to stage, time detected) is appended only when a bill's stage actually changes. A bill seen for the first time does not count as a move.
7. When the rule set's version changes, stored stages are recomputed without writing stage events, because the bill did not move; the rule did.
8. Collection health is checked on every run: if the API returns far fewer bills than are already stored, the run waits and retries once and then stops without saving; missing names or dates on a large share of rows suggest renamed upstream fields; and zero stage changes over several consecutive completed business days (public holidays excluded) are flagged.
9. Once a day, each bill is enriched from the full-history service (judiciary review, plenary, transfer to the government, promulgation, veto), bill documents and their supplementary provisions, links to committee alternatives, plenary vote totals and official effective dates from the Ministry of Government Legislation. Recently passed bills are rechecked hourly for promulgation and vetoes.
10. A daily dashboard lists, without any language-model text, the stage changes detected in the previous 24 hours grouped by stage and dated by the day the Assembly acted (not the day the change was detected, because the API publishes days late), together with new filings, promulgations, vetoes, laws taking effect, meetings in the next seven days, comment deadlines and seat changes. An empty day says so and names the next meeting. The newsletter and social posts reuse these computed numbers as their single source.
Variations: the same method applies to any legislature that publishes stage data in separate lists, at any polling interval, and to any set of stages.

## 2. A status box and timeline that update without editing the article
*In operation since September 25, 2026.*
1. Each article about a bill carries a status box and a stage timeline inside a shortcode. The shortcode's body holds a snapshot of the status at the time of writing, used only as a fallback.
2. When the page is displayed, the site replaces the snapshot with the latest values, which the hourly collection writes to a single site setting. Published articles therefore show the current stage without being edited or re-saved.
3. The status rows are: stage and date, next step, whether the bill is on an agenda, the plenary vote total, the effective date, and the date last checked. Promulgated laws show "Promulgated", "Partly in force" or "In force" using per-provision effective dates; vetoed bills show the override status.
4. The timeline shows dates for completed stages and labels the next stage "Next". A bill that was absorbed or ended stops at the stage where it ended. For committee-drafted alternatives, the first steps collapse into one step, "Drafted by the committee". The last step is "Takes effect".
5. If the bill has moved past the stage at which the analysis was written, or has since been vetoed or promulgated, the box adds a notice that the analysis below has not yet been revised.
6. The box can show the historical share of bills at the same stage that became law, labelled as a past record and not a forecast.

## 3. Linking pending bills to the articles of current law
*In operation since September 26, 2026 (article level since October 3, 2026).*
1. Each bill's name is normalised (the National Assembly and the Ministry of Government Legislation use different separators and spacing) and resolved to the identifier of the law it amends. Failed lookups are retried after 30 days. Enactments are linked only after promulgation.
2. For each law, the pending bills that would amend it are collected. The article numbers each bill would amend are read from the references to proposed articles in the bill's own official summary of reasons and contents.
3. A law's English page states how many pending bills would amend it and lists each in one line: an English one-sentence summary, its current stage and a link to its bill page. Each article's page lists the pending bills that would amend that article.
4. In the translated text, references such as "Article 15", "Articles 3 and 5", "Article 434 of the Commercial Act" and, in decrees, "Article 7 of the Act" become links to the referenced article's English page. "Presidential Decree" in an Act links to that Act's decree. Ambiguous references are left unlinked: "of the same Act", a bare "the Act" inside an Act, and laws not in the database.
5. Each article lists the articles of other laws that refer to it, with decree articles first.

## 4. Parsing Korean amendment sentences into operations
*In operation since October 4, 2026, as an internal tool.*
Korean amending bills are written as sentences such as "&lt;Law&gt; is partially amended as follows", followed by statements like "Article 5 (2) is changed from A to B". The parser turns this text into a list of operations using a grammar, without language models.
1. Split the text into per-law sections and stop at the supplementary provisions and the old/new comparison table.
2. Replace every quoted string with a placeholder, so that quoted wording cannot confuse the grammar.
3. Split the text into statements at clause connectives and at the sentence-final verb.
4. Parse each location: part, chapter, section, sub-section and division; article, including inserted articles numbered "Article N-M"; paragraph, item and sub-item; parts of a provision (main text, proviso, first or second sentence, the part other than the items, title); "the same article, paragraph or item" carried over from context; lists and ranges.
5. Classify each statement as one operation: replace text A with B; delete text A; insert text B after A; delete a unit; rewrite a unit with a new body; insert a new unit with its body; renumber a unit; change an article title. Amendments to an already promulgated amending act are tagged as references.
6. Anything that does not parse is kept as unparsed text. The parser never guesses an operation.
7. Each operation is checked against two independent answer keys: the law text (if the bill has been promulgated, the result of the operation must appear in the current official text; if not, the original wording must appear in the text before the amendment) and the old/new comparison table in the same bill document. The two results are combined into a confidence level: confirmed by both, by the law text only, by the table only, suspect, or unchecked.
8. Output: the list of operations with locations, old and new wording, new bodies and verification results. The output is used internally to measure parsing accuracy and to pull the cited articles and table rows for fact-checking; it is not displayed on law or bill pages.

## 5. An English database of Korean laws, translated by version
*In operation since October 3, 2026.*
1. The full list of current Korean laws and regulations (about 5,600) is fetched daily from the Ministry of Government Legislation. They are built in tiers: a curated first list, then Acts that pending bills would amend, other Acts, their decrees, other decrees and ministerial rules. A law's subject area comes from the curated list or from the responsible ministry.
2. English names come from the Ministry's official English name where one exists; otherwise from a rule for decrees and rules ("Enforcement Decree of the …"); otherwise from a separate translation of the title.
3. Each law's current text is stored as a version, identified by the law and its promulgation number. A version holds its articles and structural headings in order; each article is stored as structured lead text, paragraphs, items and sub-items.
4. The unit of translation is one article of one version: (law, promulgation number, article). When a new version arrives, every article of that version is translated from scratch. Translations of earlier versions are not carried over, revised or looked up, and there is no translation memory at sentence or phrase level. Headings are also translated per version.
5. Articles are translated in batches per law. Each batch carries the official English names of the laws it refers to and a fixed glossary.
6. Code blocks a translation if the numbers of paragraphs, items or sub-items differ from the original; if numbers, amounts, ratios or cited article numbers differ; if Korean text remains; or if an explanation breaks length or wording rules. A blocked article is retried once with the problems listed and otherwise left for a later run. Only translations for the law's current version are accepted.
7. A different model compares a random sample of translated articles with the Korean original. A mismatch deletes the translation, so it is redone.
8. Short "in plain English" notes are written separately from the translation. Code supplies the conditional clauses found in the article; a different model checks each note against the original; only notes that pass are published, and a failing note is deleted while the translation is kept.
9. Display rule: a law with one version shows that version, even while translation is in progress. When a newer version exists, the page shows one whole version, the most recent one whose translation is at least 90% complete, and never mixes articles from different versions. If the version shown is not the newest, the page says so.
10. Part, chapter, section, sub-section and division headings form a table-of-contents tree. Each node records its first and last article, empty nodes are dropped, labels such as "Chapter 2-2" are generated by code, and titles appear once translated.
11. A full-text index with weighted headings and snippets serves site search. Only laws and articles that changed are submitted to search engines.

## 6. Judging relevance and topics bill by bill
*In operation since October 4, 2026.*
1. Bills are not selected by matching words in their titles. A language-model reviewer reads each bill that is newly filed or that has moved to the subcommittee stage or later and has no article, using its title, committee, stage and its official summary of reasons and contents.
2. The reviewer applies one test: if the bill passes, would it change the obligations, costs, permits, sanctions or rights of foreign companies or foreign nationals, or the conditions for doing business, living or trading in Korea? Bills limited to the internal workings of public bodies, with no effect on foreign businesses, fail the test. No list of sectors is given in advance.
3. Each judgment is appended to a log with: the bills judged, whether to write an article or not, whether the bill gets a public fact page, one to three topics from a fixed list of 25, a one-sentence reason, and an angle when an article is to be written.
4. Code builds the latest judgment per bill from the log. That judgment decides whether a bill has a fact page and drives the watch lists, topic pages, the taking-effect calendar, the early-stage list and the list of article candidates. Pages created before October 4, 2026 come from an earlier selection process and are replaced as new judgments are made.

## 7. Writing and verification
*In operation since September 26, 2026 (multi-agent line since September 27, 2026).*
1. Writing starts from the source documents (the committee-approved text, the expert review report, the plenary amendment or the official law text). First, a list of action facts is extracted: who must do what, from and by when, how much, and under what conditions ("may" or "shall", "without justifiable reason", thresholds).
2. The body is written from that list. The summary layer (headline deck, key points, search description, frequently asked questions) is then written only by copying from the body, never adding facts; a fact whose condition cannot be kept is left out of the summary. Statements about current law come only from the current-law column of the comparison table or from the official current text.
3. Before publication, at least three action facts are recorded with their source location and the condition words that must be kept. Automated checks use them to flag summary sentences that dropped a condition, and flag summary sentences that lost a limiting word such as "only", "unless" or "within" found in the closest body sentence.
4. A writing agent drafts the article. A source-checking agent in a separate session, which sees only the sources and the draft, verifies it sentence by sentence and can send it back twice. An auditing agent running a different model re-verifies it from the sources just before publication.
5. A code gate re-runs on a clean copy of the committed code, re-reads both final verdicts and runs about 30 blocking text checks, including: Korean text in the body; bill numbers and lawmakers' names; predictive wording; definitive wording before a bill is final; present tense before promulgation; inconsistent fine amounts; topics outside the fixed list; side-by-side or highlighted article comparisons; and statements that a party supported or opposed a bill. Any blocking item stops publication.
6. Code then generates images, creates the post, records the change, waits for the automated test suite and publishes.
7. Each run also re-verifies one already-published article from its sources, oldest first.

## 8. Public corrections
*In operation since September 26, 2026.*
1. Every error found after publication is appended to a ledger with the type of error, how it was found, the stage at which it entered and the rule added to prevent it.
2. The article receives a dated "Correction:" line in its public update log, and the correction is listed on the [corrections page](https://koreanics.com/corrections/).
3. Publication is blocked if a recorded correction has no dated "Correction:" line in the article, or if the corrected wrong wording, stored as a pattern, still appears anywhere in the article.

## 9. Forecasts recorded before outcomes are known
*In operation since September 27, 2026. Forecasts are not published on the site.*
1. Models estimate whether the subject of a bill becomes law within a fixed horizon (one year in the first models; several horizons from about one month to about two years in a later model). "Becomes law" means passed by the plenary, or absorbed into a committee alternative that passed; a failed veto override counts as no. Only stages before the plenary vote are estimated.
2. The models are simple enough for anyone to recompute from public data: a base rate per stage, and tables of historical rates over a few public bill attributes, falling back to coarser groups where data is thin.
3. Models are trained on completed Assembly terms. Tuning uses one older term for training and the next for testing. Before a model goes live, it is tested once on the most recent completed term and that result is sealed; any change made after seeing it becomes a new model version or name. The delay between an Assembly action and its appearance in the API is measured per field and applied in back-tests.
4. Every day each model writes one row per bill on a roster (bills that reached a new stage, plus a monthly census of all pending bills) to a table that the database prevents from being updated or deleted. Fingerprints of the rows, the model tables, the specification and the scoring rules enter that day's record seal, before outcomes are known.
5. Scoring rules are fixed in advance and versioned: a Brier score per model on identical rows, and skill relative to the base-rate model. Rows whose horizon has not closed are not scored.
6. New models are proposed by an agent and attacked by a separate agent looking for leakage, label errors, contaminated samples and inflated scores. The data they work on has the evaluation term's outcomes removed, and model code written by agents runs only in a restricted environment.

## 10. The daily record seal
*In operation since September 27, 2026. See also [how the seal works](https://koreanics.com/how-we-track/#seal).*
1. Each day, a manifest lists the SHA-256 fingerprint of each sealed file (the day's dashboard data, forecast rows, model tables, test results and news-count rows), the versions of the stage and scoring rules, and the commit identifier of the production code.
2. Files are fingerprinted in a canonical JSON form: sorted keys, no whitespace, Unicode unescaped.
3. The day's seal is SHA-256 of the previous day's seal, a newline and the canonical manifest. The first seal follows 64 zeros. There is one seal per day, chained in date order, in a table that cannot be updated or deleted.
4. Each seal is timestamped by three independent RFC 3161 services and submitted to OpenTimestamps, whose proofs are completed the next day with a Bitcoin block attestation. The [seal page](https://koreanics.com/seal/) lists each day's seal and its witnesses.
5. Internet Archive copies are requested daily for the dashboard and the seal page. Pages on the site that explain our methods are copied whenever their text changes, and a copy counts as made only when it appears in the Archive's own listing.
6. The chain is recomputed from the stored manifests daily, and again after weekly backup-restore tests. If the daily run is cut short, a later hourly run creates the missing seal.

## 11. Autonomous editorial and maintenance operations
*In operation since September 27, 2026, extended through October 5, 2026.*
1. Division of work: code does everything that must be deterministic (collection, status, dashboard, seal, forecasts, backups, integrity checks, publishing and deployment) without language models. Language-model agents do judgment and prose only, and write files at most. A writer and its checker always run in separate sessions.
2. Roles include: a relevance reviewer; writer, source checker and auditor; summary writer and checker for bill pages; law translator and law checker; law-name checkers; a social-post writer; a site monitor that reads the live site daily; an alert responder; a daily line reviewer; a failure analyst; a code fixer; weekly code reviewers with a rebuttal step; a reviewer of passed bills that received no article; model-lab agents; and idea scouts.
3. Code builds the work queue: updates for bills with articles that have moved; new candidates; newly filed bills; deferred and blocked items within a retry window; the weekly issue; trims of long older articles; and the sampled re-check. There is no cap on new articles; a time budget per run defers the rest to the next run.
4. Least privilege: each agent has a fixed list of permitted tools, reads the database only through a read-only script, and may write only in its own folder. After every call, code compares the repository state and file fingerprints; any write outside the permitted folder discards the whole run. The code fixer may not touch governance rules, editorial decisions, legal guardrail documents and their tests, itself and its tests, deployment scripts, secrets, the correction and incident ledgers, or data and logs.
5. Before any change is deployed: text checks, the publication gate re-run on the committed code, static analysis, the full unit-test suite, template syntax checks, output comparisons for refactors and guardrail tests. The fixer reverts all its changes if any check fails; the server resets to the previous code if its own tests fail after an update; a failed deployment removes the draft post and retries.
6. Alerts from every job go into an append-only ledger, de-duplicated per day, instead of email. Code closes informational alerts; a read-only responder agent diagnoses the rest; only predefined reruns are executed, and code executes them; open items go to the code fixer.
7. Self-supervision: daily reviewer agents may issue short working guidance to other agents, but code removes any guidance that touches pass or block criteria, guidance expires after three days, and code measures quality and cost before and after each piece of guidance and revokes guidance that made things worse. A failure type that recurs after being marked fixed is flagged as recurring by code. The weekly rebuttal agent is given planted false findings that it is expected to reject. Checkers are scored on past real errors and on a control group of articles with no known errors. Repeated failures stop a line, and a single pause switch stops everything.
8. Idea scouting: every hour, new posts from a public technology-news feed are read for insights applicable to our product, design, business model, code or operations, and an internal report is produced only when there is one. Separately, every hour code draws a random encyclopedia article (re-drawing biographies and disambiguation pages) and a random thinking technique, and an agent brainstorms business ideas from that pair, with a log to avoid repeats.

## 12. Follow emails and topic alerts
*Follow emails in operation since September 26, 2026; topic alerts are a planned design.*
1. A reader enters an email address on a bill page or a law page. Confirmation is double opt-in: the link in the confirmation email opens a page, and a button on that page activates the follow, so email scanners that open links automatically cannot confirm.
2. An hourly job sends an email when a followed bill's stage changes. For a followed law, it sends at most one email a day when a pending bill that would amend the law moves, is filed or drops off, or when the law itself is amended.
3. Every email carries one-click unsubscribe and links to stop a single item or all items. Unsubscribing deletes the record. Only the address and the followed items are stored.
4. Planned topic alerts: a reader picks one or more topics from the same fixed list of 25. Alerts cover stage changes, new filings and effective dates of bills assigned to that topic under section 6. Everyone who picks the same topic receives the same content, and nothing is computed per reader.

## 13. Open data and access for AI assistants
*In operation since September 30, 2026 (AI assistant access since October 4, 2026).*
1. Open data files are rebuilt hourly without language models: a bill list (English title, stage, dates, topics, one-line summary, link), observed stage changes only, and effective dates, with a README stating the terms of use. Details are on the [open data page](https://koreanics.com/open-data/).
2. A [Model Context Protocol server](https://koreanics.com/mcp-server/) lets AI assistants query the same data read-only over stateless HTTP: full-text search of law articles, a single article, the list of laws, bill search, a single bill and the latest articles. Every answer carries the terms of use.
3. A file for AI crawlers states that search and citation are welcome and that model training is not permitted. A fixed, visible marker string appears on law pages, articles, the open data README and the [copyright page](https://koreanics.com/copyright/), and suspect text can be compared with our published text by word-sequence fingerprints.

## 14. Daily social cards
*In operation since October 5, 2026.*
1. A pool of candidates is drawn from the published English law articles: laws are shuffled with a random generator seeded by the date, one article per law is picked within a length range (excluding deleted and already-used articles, and preferring articles with checked notes), and each candidate is split into paragraph and item parts with links to those exact parts.
2. The card type rotates by date among three series: a business rule, a surprising but real rule, and "Can I … in Korea?".
3. The visual style rotates by date over a fixed cycle of colour theme, left or right layout and mascot expression, so consecutive days differ while the character stays the same.
4. A writing agent produces three options from the article's facts only; code checks length, banned words and bill numbers; a checking agent compares the options with the Korean original; the first option that passes is rendered from an HTML template to an image in a headless browser. The reply that links to the source is built by code. Used articles are logged so they do not repeat.

## 15. Other methods
1. Effective dates: supplementary provisions are parsed for effective-date clauses, including periods counted from promulgation and exceptions for particular provisions; periods are resolved to dates on promulgation and cross-checked with the official per-provision dates. A 90-day calendar of laws taking effect lists one line per date and offers subscribable calendar feeds for everything, for comment deadlines and for each topic. (Since September 25 and 29, 2026.)
2. Early-stage list: bills on agendas in the next seven days, open comment periods, bills that passed subcommittee, government bills in committee and open government pre-submission notices, each section headed by the historical share of bills at that stage that became law, with no probabilities for individual bills. (Since September 28, 2026.)
3. Government bills before submission: milestones are tracked from the government's legislative-status service and linked exactly to Assembly bill numbers, with facts known before submission stored separately from later outcomes. (Since September 28, 2026.)
4. Reconciliation: once a month, the database's bill and passage counts for each Assembly term are compared with the Assembly's official statistics; completed terms must match exactly, and explained differences are listed. (Since September 26, 2026.)
5. News-attention record: daily public news-search counts per pending bill are frozen, grouped by search query to save calls, in a table that cannot be changed, and the day's rows enter the record seal, so the data cannot be reconstructed later with hindsight. (Since September 28, 2026.)
6. Candidate score: a transparent, versioned rule-based score with two parts, outside attention and importance signals within the bill's own documents, is stored for every candidate, including those not chosen, and is used only as reference input for the relevance reviewer. (Since September 26, 2026.)
7. "By the numbers" pages: code fills numbers from the database into sentence templates, so the writing agent never types a number; only completed terms are used. (Since September 28, 2026.)
8. Integrity check: a daily job counts, without fixing, stored values that differ from current rules, seal-chain mismatches, latest stage events that disagree with the current stage, collection stalls, articles pointing to missing bills and promulgated laws still without an effective date after 30 days. Fixes are a separate step: backup, dry run, then apply. (Since September 27, 2026.)
9. Bill fact pages: code builds a fact page per bill as a data file served by the site rather than as a post. The English "What the bill proposes" summary is written by one model and checked by a different model, and a fingerprint of the source invalidates the summary when the source changes. (Since September 27, 2026.)
10. Article images: code redraws an article's lead image when the stage label drawn into the image becomes out of date.
11. New regulations: new enforcement decrees and rules for laws covered by articles are detected, with the first lookup stored as a baseline rather than reported as new. (Since September 26, 2026.)
12. Meetings: the list of committee meetings at which each bill was discussed is collected as a list. (Since September 27, 2026.)
13. Party lineages: renamed political parties are grouped into lineages using dates, which separates parties that reused a name. Used for internal analysis only. (Since September 26, 2026.)

## How to verify the date
Each version of this file is timestamped by independent services, and the proofs are kept in the [`proofs/`](proofs/) folder of this repository:

- RFC 3161 timestamps of the file's SHA-256 from FreeTSA, DigiCert and Sectigo (`*.tsr`). Check with `openssl ts -reply -in <file>.tsr -text`, and verify against the authority's certificate.
- An OpenTimestamps proof (`README.md.ots`), which is anchored in a Bitcoin block once confirmed. Check with `ots verify proofs/v1/README.md.ots` against the README of that version.
- Copies of this repository's pages in the Internet Archive and in Software Heritage, which record the date on which each captured the public repository.

The Koreanics daily [record seal](https://koreanics.com/seal/) also includes the SHA-256 of this text and of the production code that implements these methods.

## License
The text of this repository is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). The methods described here are free for anyone to use.

## Version history
Version 1, October 5, 2026: first publication.
