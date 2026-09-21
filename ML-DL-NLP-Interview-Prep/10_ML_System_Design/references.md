# ML System Design — References

All links below were checked this session (fetched and content verified against the claim) unless explicitly marked otherwise.

## Books

- Chip Huyen, *Designing Machine Learning Systems* (O'Reilly, 2022) — the standard end-to-end reference; also directly useful for this topic's framing/architecture questions. Resources repo: https://github.com/chiphuyen/dmls-book
- Ali Aminian and Alex Xu, *Machine Learning System Design Interview* (ByteByteGo, 2023) — a 7-step framework plus 10 worked ML system design questions (recommendation, search ranking, ad click prediction, etc.). Purchase/details page (not URL-verified this session — verify the live purchase link before sharing, but the book's existence, authors, and ISBN 9781736049129 were corroborated across multiple independent bookseller listings).
- Chip Huyen, *Machine Learning Interviews* (free online book + companion repo, includes a section of open-ended ML system design questions and references her Stanford "Machine Learning Systems Design" course). https://github.com/chiphuyen/ml-interviews-book

## Papers

- Covington, Adams, Sargin, "Deep Neural Networks for YouTube Recommendations" (ACM RecSys 2016) — canonical reference for the two-stage candidate-generation + ranking architecture. PDF mirror: https://cseweb.ucsd.edu/classes/fa17/cse291-b/reading/p191-covington.pdf ACM record: https://dl.acm.org/doi/10.1145/2959100.2959190

## Official Docs

- PyTorch Distributed Overview — official documentation covering data parallelism (DDP) and model/tensor/pipeline parallelism (FSDP, TP, PP), directly relevant to the distributed-training section. https://docs.pytorch.org/tutorials/beginner/dist_overview.html

## Articles / Interview Prep

- Analytics Vidhya, "System Design for ML Interviews: 10 Real Problems Walked Through" (2026). https://www.analyticsvidhya.com/blog/2026/06/system-design-for-ml-interviews-real-problems-solved/
- ByteByteGo / Alex Xu ecosystem articles on ML system design interview structure (not URL-verified this session — the book above is verified to exist; search "bytebytego machine learning system design" for current article links before sharing).
- Shaped.ai, "Two-Tower Models for Recommendation Systems" — clear deep dive on the retrieval-stage architecture. https://www.shaped.ai/blog/the-two-tower-model-for-recommendation-systems-a-deep-dive

## Notes on What to Prioritize

Across sources, the same three worked problems recur constantly in real interviews: a recommendation system, a search/ranking system, and a fraud-detection system (occasionally swapped for ad click-through-rate prediction, which uses the same skeleton). Interviewers weight the structured framework (objective -> metric -> baseline -> architecture -> evaluation -> monitoring) as much as the specific architecture choice — practice narrating the framework out loud, not just knowing the two-stage retrieval-then-ranking pattern.
