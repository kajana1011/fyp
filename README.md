# UNIVERSITY FINAL YEAR PROJECT MANAGEMENT SYSTEM

## Table of Contents

- [Project Overview](#project-overview)
- [Objectives](#objectives)
- [Key Features](#key-features)

- [Architecture & Tech Stack](#architecture--tech-stack)
- [Data Model](#data-model-recommended)
- [Duplicate Detection Design](#duplicate-detection-design)

- [Algorithm Choice & Rationale](#algorithm-choice--rationale)
- [Mentor Assignment Design](#mentor-assignment-design)
- [APIs and Endpoints (suggested)](#apis-and-endpoints-suggested)
- [Configuration](#configuration)

- [Developer Setup & Run](#developer-setup--run)
- [Testing](#testing)
- [Extensibility & Contribution](#extensibility--contribution)

- [Admin Notes](#admin-notes)
- [Notifications & Mentor Calendar](#notifications--mentor-calendar)
- [Contact / Maintainer](#contact--maintainer)

## Project Overview

The Final Year Project Management System automates evaluation-time checks to ensure final-year project titles, objectives, and implementations are unique, reducing manual review effort by faculty panels. The system also automates mentor assignment and provides admin views to manage settings and view student groups per mentor.

## Objectives

- Main Objective: Automatically detect duplicate project titles, objectives, and implementations to ensure unique student projects.
- Specific Objectives:
	- Design a database schema to store project details, mentor assignments, and student records.

## Key Features

- Configurable duplicate checking window (current year, last N years).
- Title/objective/implementation similarity detection using configurable algorithms.
- Automated mentor assignment with preference and load-balancing rules.

## Architecture & Tech Stack

- Frontend: Flutter (cross-platform mobile/desktop/web). See [lib/main.dart](lib/main.dart#L1).
- Backend: Flexible - can be a REST API (Node/Django/Flask/FastAPI) or integrated service. The repository currently contains a Flutter project; if a backend is added, keep it as a separate service.

- Database: Relational DB (Postgres/MySQL) recommended for structured queries; can also use Firestore or other managed DB.
- Duplicate detection: Text-similarity modules (TF-IDF + cosine, sentence embeddings, Levenshtein) depending on accuracy/performance tradeoffs.

## Data Model (recommended)

Entities and essential fields:

- `Student`:
	- `id`, `name`, `email`, `year`, `preferences` (optional)
- `Mentor`:

	- `id`, `name`, `email`, `expertise_tags`, `max_load`
- `Project`:
	- `id`, `student_id`, `title`, `objectives`, `implementation_summary`, `year`, `created_at`, `status`

- `MentorAssignment`:
	- `id`, `mentor_id`, `project_id`, `assigned_at`, `type` (auto/manual)

Indexes:
- Full-text indexes on `title`, `objectives`, and `implementation_summary` for fast similarity queries.

## Duplicate Detection Design

Goals: detect similar/duplicated project titles, objectives, or implementation descriptions with configurable sensitivity and look-back period.

Core pipeline (recommended):

1. Preprocessing: normalize case, remove punctuation, stopwords, and perform light stemming.
2. Vectorization options (configurable):

	- TF-IDF vectors + cosine similarity (fast, explainable).
	- Sentence embeddings (e.g., SBERT) + cosine similarity (better semantic matching).
	- Character-level Levenshtein for short titles (catch near-duplicates/typos).

3. Similarity scoring: combine measures (weighted) to produce a single similarity score.
4. Thresholding: expose a configurable threshold (e.g., 0.75) to classify duplicates; admin can tune per corpus and year window.
5. History scope: query projects from current year or last N years depending on admin setting.

### Similarity Score Classification

After computing the final similarity score (combined / weighted from embedding, lexical, and edit-distance signals), the system classifies results as follows:

| Similarity Score | System Action |
|---:|---|
| < 0.6 | Auto-approve (no duplicate action required) |
| 0.6 – 0.8 | Flag for admin review (present matches and explainability info) |
| ≥ 0.8 | Auto-flag as duplicate (prevent submission or require student action) |

These ranges are configurable by the admin; the defaults above provide a balance between minimizing false positives (admins review medium scores) and catching high-confidence duplicates automatically.

### Recommended Pipeline (TF-IDF → SBERT → Decision)

For submissions the system will apply a two-stage pipeline by default:

1. TF-IDF similarity (fast lexical filter): quickly retrieve top candidate projects that share lexical overlap or important tokens with the new submission. This stage is very fast and provides explainable token matches for admin review.
2. SBERT similarity (semantic confirmation): compute or reuse precomputed SBERT embeddings for the submission and re-rank the TF-IDF candidates by cosine similarity to confirm semantic similarity and catch paraphrases.
3. Decision + admin review: combine TF-IDF and SBERT signals (with optional Levenshtein/title guard) to compute the final similarity score and apply the classification rules (auto-approve / flag for review / auto-flag duplicate).

Flow (visual):

New project submitted
	↓
TF-IDF similarity (fast lexical filter)
	↓
SBERT similarity (semantic confirmation)
	↓
Decision + admin review

Notes:
- TF-IDF stage reduces the number of expensive embedding comparisons and provides lexical evidence.
- SBERT stage improves recall for rephrased content and reduces false negatives.
- Weighting between TF-IDF and SBERT is configurable; tune on a labeled validation set.

Performance & scaling:
- Precompute and persist vector embeddings for all stored projects; update embeddings on create/update.
- Use approximate nearest neighbor (ANN) index (e.g., Faiss, Annoy) for large corpora.

Explainability:
- Provide the administrator with the matched fields, similarity score, and top contributing tokens/phrases for each match.

## Algorithm Choice & Rationale

1) What algorithm will you use and why?

- Primary: Sentence embeddings (SBERT / sentence-transformers) for semantic similarity — chosen because embeddings capture meaning and paraphrase relationships, not just surface word overlap. SBERT gives high-quality sentence/passage vectors with efficient cosine similarity.
- Secondary (hybrid): TF-IDF + cosine for fast lexical checks and explainability; Levenshtein for short-title typo/near-match detection. A hybrid approach balances speed, explainability, and semantic accuracy.

2) How will you use the algorithm?

- Preprocess text (normalize, strip punctuation, basic token filtering).
- Compute sentence embeddings for `title`, `objectives`, and `implementation_summary` at create/update and store them alongside the DB record.
- Use an ANN index (Faiss/Annoy) over embeddings to quickly retrieve nearest neighbors for a submitted project, then compute final similarity scores by combining embedding cosine similarity, TF-IDF lexical score, and Levenshtein where applicable.
- Apply configurable thresholds and return the top-K matches with per-field scores and explanations to the admin/UI.

3) What will be your minimum storage?

- Storage per project (approximate):
	- Embedding vector (example SBERT 768-dim float32): 768 * 4 bytes ≈ 3,072 bytes (~3 KB).
	- Project metadata (title/objectives/summary, indices, timestamps): ~1 KB (varies).
	- Total ≈ 4 KB per project.
- Examples:
	- 10,000 projects → ~40 MB for embeddings + metadata.
	- 100,000 projects → ~400 MB. ANN index and any replication/backups add extra overhead (order of tens to hundreds of MB depending on index type and compression). Using quantization (Faiss PQ/OPQ) can reduce vector storage significantly (e.g., to ~0.5–1 KB per vector).

4) What is the significance of your project?

- Saves faculty time: automates repetitive duplicate checks during final-year evaluations.
- Improves fairness and originality: ensures students work on distinct, meaningful projects and reduces accidental reuse.
- Administrative efficiency: configurable historical scope, audit trails, and automated mentor assignment reduces manual workload.
- Research/analytics: collected data enables trends analysis (popular topics, mentor loads, common rephrasings) and supports continuous improvement.

5) How will the chosen algorithm detect similarity beyond word overlap (rephrasing)?

- Sentence embeddings map semantically similar sentences/paragraphs to nearby vectors even when wording changes; SBERT is trained to bring paraphrases closer in vector space, so rephrased objectives or summaries produce high cosine similarity despite low token overlap.
- The hybrid pipeline reinforces this: TF-IDF catches exact or partial lexical matches, Levenshtein detects near-typos, and the embedding similarity captures semantic equivalence. Combining these signals with configurable weights reduces false negatives for paraphrases and false positives for coincidental word overlap.

### Hybrid as the Default and When a Single Algorithm Is Used

- Default choice: **hybrid pipeline** (ANN over SBERT embeddings + TF-IDF/BM25 + title-level Levenshtein) because it balances semantic recall, lexical explainability, typo-robustness, and scalability. The hybrid design provides high recall for paraphrases (embeddings), transparent lexical evidence for admin review (TF-IDF/BM25), and short-text guards (Levenshtein) for titles.
- When only embeddings (SBERT) are used: choose this when semantic matching is the priority and you have resources to host embeddings and an ANN index, and when privacy requirements allow storing embeddings. Use embeddings-only retrieval for broad semantic search where explainability is less critical.
- When only TF-IDF/BM25 is used: choose this lightweight option when constrained by compute, memory, or strict privacy (no embeddings/external APIs), or when the corpus is small and phrasing differences are minimal. TF-IDF/BM25 is fast, explainable, and easy to host.
- When only Levenshtein/n-gram is used: apply this narrowly for very short fields (project `title`) to catch typos and small edits; it is not sufficient alone for semantic paraphrase detection.

In practice we start with the hybrid default and provide admin-configurable fallbacks so operators can run TF-IDF-only or embeddings-only modes when operational constraints demand it.


## Mentor Assignment Design

Auto-assignment strategy (recommended):

1. Compute topic similarity between project and mentor expertise tags or mentor-profile embeddings.
2. Filter mentors by availability (current load < `max_load`).
3. Respect student preference list when present — try to match preferences first.

4. Apply load-balancing by selecting the mentor with highest similarity and lowest load.
5. Allow manual override in admin UI; record overrides in `MentorAssignment.type = 'manual'`.

## APIs and Endpoints (suggested)

These are suggested REST endpoints for a backend service. Adjust to your chosen framework.

- POST `/api/projects` — create a project (store project + compute & store embeddings).
- GET `/api/projects/{id}/duplicates?years_back=3&threshold=0.75` — returns potential duplicates with scores.
- POST `/api/projects/check-duplicates` — realtime check for an input title/objectives.

- POST `/api/assignments/auto-assign` — auto-assign mentors for unassigned projects.
- GET `/api/mentors/{id}/students` — returns students assigned to that mentor.
- GET `/api/admin/settings` and PUT `/api/admin/settings` — admin settings (years_back, thresholds).

Authentication & Authorization:
- Protect admin APIs with role-based auth (JWT/OAuth). Student endpoints limited to their own data.

## Configuration

Key configurable parameters:

- `DUPLICATE_SEARCH_YEARS_BACK` (int)
- `DUPLICATE_SIMILARITY_THRESHOLD` (float 0..1)
- `DUPLICATE_ALGORITHM` (enum: TFIDF|EMBEDDING|HYBRID)

- `MENTOR_MAX_LOAD_DEFAULT` (int)
- DB connection string and auth secrets

Store these in environment variables or a secure settings file; never commit secrets to source control.

## Developer Setup & Run

Prerequisites:

- Install Flutter SDK (stable channel). Ensure `flutter` is on PATH.
- For mobile targets: Android Studio and Android SDK or Xcode for iOS (macOS only).

Common commands:

```bash
# from project root
flutter pub get

flutter analyze
flutter test
flutter run    # choose device when prompted

```

If you add a backend service, include a README in that service with its own setup steps (install deps, migrations, run server).

Local dev tips:
- Use a small seed dataset to test duplicate detection logic.
- Add a `--mock-backend` or environment flag to run the Flutter UI against a mock server during UI work.

## Testing

- Unit tests: test preprocessing, vectorization, similarity scoring, and mentor-assignment logic.
- Integration tests: end-to-end flow from project creation → duplicate detection → mentor assignment.

- Performance tests: measure duplicate check latency with realistic dataset sizes; consider ANN indexing for large datasets.

## Extensibility & Contribution

- Adding a new duplicate detection algorithm:
	- Implement a module with `compute_embedding(text)` and `similarity(a,b)` interfaces.

	- Update the config switch and migration to compute embeddings for existing records.
- Adding a backend: keep API contract stable; document new endpoints and update the Flutter API client.

- Coding style: follow Dart/Flutter style and lint rules. Run `flutter format` before commits.

## Admin Notes

- Admins can tune how many years back to search for duplicates; increasing the window reduces false negatives but may increase false positives.
- Provide a review workflow where flagged duplicates are presented to faculty with an option to mark false positives.

## Notifications & Mentor Calendar

- Mentor Availability & Notifications: Mentors publish available meeting time slots. Supervisees are notified of published slots and are expected to attend at the scheduled time. Students may request a reschedule which requires mentor approval; student-initiated cancellations must be approved by the mentor or follow a configured cancellation policy.
- Mentor Availability Calendar: Mentors maintain a calendar of available time slots. The calendar is visible to their supervisees to view scheduled meetings and availability.
- Appointment Deadlines: Admins can configure deadlines for mentor-student appointments (e.g., meetings must be scheduled X days before a review). The system surfaces upcoming deadlines in mentor and student dashboards.
- Appointment History: Maintain a history of appointment records and statuses (proposed, confirmed, completed, cancelled, reschedule_requested) per mentor and per student for auditing and tracking.
- Notification channels: in-app push notifications are primary; optionally add email notifications for critical events.


Implementation notes:

- Model additions: `Appointment` entity with fields: `id`, `mentor_id`, `student_id`, `start_time`, `end_time`, `status` (enum: `proposed|confirmed|completed|cancelled|reschedule_requested`), `created_at`, `updated_at`, `notes`, `reschedule_requested_by` (nullable), `cancelled_by` (nullable), `cancel_reason` (nullable).
- Calendar backend: use iCalendar or a simple time-slot model; expose endpoints for CRUD, availability queries, and listing confirmed appointments.
- Workflow & validation: mentors publish available slots; when a mentor publishes a slot the system notifies supervisees. Students can confirm attendance or request rescheduling; reschedule requests require mentor approval. Student cancellations are subject to mentor/admin approval per policy.
- Conflicts & validation: prevent overlapping confirmed appointments for the same mentor; enforce `max_load` constraints where applicable.
- Background jobs: scheduled reminder notifications, automatic enforcement of appointment deadlines, and periodic cleanup/archival jobs.
- UI: calendar view (week/month), notification bell, confirm/reschedule UX for students, mentor approval screens, and an appointment history list.

## Contact / Maintainer

Mentor: Miss Joyce Ringo

If you join development, open a PR with a description of the change, tests, and any migration scripts.

---
This README provides a technical overview to help future developers understand, run, extend, and maintain the Final Year Project Management System. For implementation-specific questions, consult code comments and open issues in the repository.

# fyp

A new Flutter project.

## Getting Started

This project is a starting point for a Flutter application.

A few resources to get you started if this is your first Flutter project:

- [Lab: Write your first Flutter app](https://docs.flutter.dev/get-started/codelab)
- [Cookbook: Useful Flutter samples](https://docs.flutter.dev/cookbook)

For help getting started with Flutter development, view the
[online documentation](https://docs.flutter.dev/), which offers tutorials,
samples, guidance on mobile development, and a full API reference.
