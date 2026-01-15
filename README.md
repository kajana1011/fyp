# UNIVERSITY FINAL YEAR PROJECT MANAGEMENT SYSTEM

## Table of Contents

- Project Overview
- Objectives
- Key Features

- Architecture & Tech Stack
- Data Model
- Duplicate Detection Design

- Mentor Assignment Design
- APIs and Endpoints (suggested)
- Configuration

- Developer Setup & Run
- Testing
- Extensibility & Contribution

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

Performance & scaling:
- Precompute and persist vector embeddings for all stored projects; update embeddings on create/update.
- Use approximate nearest neighbor (ANN) index (e.g., Faiss, Annoy) for large corpora.

Explainability:
- Provide the administrator with the matched fields, similarity score, and top contributing tokens/phrases for each match.

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
