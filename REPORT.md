# X "For You" Feed Algorithm - Repository Report

## 1. Overview
This repository contains the core recommendation system powering the "For You" feed on X. The primary function of this system is to retrieve, rank, and filter posts to present the most engaging content to a user. It combines in-network content (posts from accounts the user follows) with out-of-network content (discovered through machine learning retrieval) and ranks them using a Grok-based transformer model.

The system is characterized by the elimination of hand-engineered features; instead, it relies heavily on the Grok-based transformer model to deeply understand user engagement history and predict relevance across multiple action types (like, reply, repost, etc.).

## 2. System Architecture & Main Workflow
The system orchestrates recommendations through a pipeline consisting of several key stages:
1.  **Query Hydration:** Gathers user context, including engagement history and preferences.
2.  **Candidate Sourcing:** Retrieves candidates from both in-network (Thunder) and out-of-network (Phoenix Retrieval) sources.
3.  **Candidate Hydration:** Enriches candidates with essential data like core post metadata, author information, and media details.
4.  **Pre-Scoring Filters:** Removes duplicate, old, self-posted, or muted/blocked content.
5.  **Scoring:** Predicts engagement probabilities using the Phoenix Scorer, then combines them via a Weighted Scorer, and finally adjusts them (e.g., for Author Diversity).
6.  **Selection:** Sorts candidates by their final scores and selects the top 'K'.
7.  **Post-Selection Filtering:** Performs final checks such as removing deleted, spam, or explicit content.

## 3. Core Modules

### 3.1. Home Mixer (`home-mixer/`)
The `Home Mixer` acts as the orchestration layer for the entire "For You" feed. It utilizes a `CandidatePipeline` framework to assemble the feed by running through the stages of query hydration, candidate sourcing, candidate hydration, filtering, scoring, selection, and side-effect processing (such as caching). It exposes a gRPC endpoint (`ScoredPostsService`) that returns the finalized ranked posts to the user. Additionally, it now handles ad blending and positioning, alongside various query and candidate hydrators.

### 3.2. Thunder (`thunder/`)
`Thunder` is the in-memory post store and real-time ingestion pipeline responsible for managing "in-network" content. It consumes post events from Kafka and maintains per-user stores for various post types. This allows for sub-millisecond lookups for posts from accounts a user follows, without the need to query an external database.

### 3.3. Phoenix (`phoenix/`)
`Phoenix` is the core machine learning component, primarily responsible for out-of-network retrieval and the overall ranking.
*   **Retrieval:** Uses a Two-Tower model to encode user features and posts into embeddings, finding relevant out-of-network posts via similarity search.
*   **Ranking:** Uses a Grok-based transformer to predict engagement probabilities for each candidate post, utilizing candidate isolation to ensure independent scoring. It recently added pre-trained model artifacts and a single entry point for end-to-end inference.

### 3.4. Candidate Pipeline (`candidate-pipeline/`)
This is a reusable framework (a Rust crate) designed for building recommendation pipelines. It defines traits for the various stages of the recommendation process (`Source`, `Hydrator`, `Filter`, `Scorer`, `Selector`, `SideEffect`) and executes sources and hydrators in parallel to ensure efficient processing.

### 3.5. Grox (`grox/`)
A newly introduced content-understanding pipeline service. `Grox` provides classifiers, embedders, and a task-execution engine designed to handle content understanding workloads such as spam detection, post-category classification, and enforcing policy rules (like PTOS).

## 4. Key Design Principles
*   **No Hand-Engineered Features:** Complete reliance on the Grok-based transformer for learning relevance.
*   **Candidate Isolation:** During ranking, posts are scored independently based on user context, preventing scores from depending on batch composition.
*   **Multi-Action Prediction:** The model predicts probabilities for a wide range of engagements (e.g., favorite, reply, dwell time) rather than a single relevance score.
*   **Composable Pipeline:** Uses the `candidate-pipeline` crate to maintain a flexible, parallelized, and easily extendable architecture.