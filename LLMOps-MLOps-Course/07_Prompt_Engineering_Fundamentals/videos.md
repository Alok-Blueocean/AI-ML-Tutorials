# Videos — Prompt Engineering Fundamentals

Curated external video resources to complement this module. Ranked by star rating (up to 5).
All titles below were verified to exist via direct search/fetch before inclusion; durations are
only listed where they could be confirmed, per the course's no-fabrication policy.

---

### ChatGPT Prompt Engineering for Developers
**Creator/Channel:** DeepLearning.AI, taught by Isa Fulford (OpenAI) and Andrew Ng (DeepLearning.AI)
**Format:** Short course, 9 video lessons + 7 hands-on notebooks (~1 hour 40 minutes total)
**Difficulty:** Beginner
**Rating:** ★★★★★

**Why it's worth watching:** This is the closest thing the field has to a canonical "official"
starting point — co-authored by the OpenAI staffer who helped define early best practices and
delivered by Andrew Ng, whose course pedagogy is unusually good at building intuition before
mechanics. It covers writing clear instructions, iterating systematically, summarizing/inferring/
transforming text, and structuring output — all directly relevant to this module's structured-
prompt section. Free to audit.

**Complements:** Part 2 (structured 5-section prompts, output formatting) and the general
discipline framing of "prompt engineering as systematic iteration, not guesswork."

**Link:** https://www.deeplearning.ai/short-courses/chatgpt-prompt-engineering-for-developers/

---

### Anthropic's Prompt Engineering Interactive Tutorial (video/notebook companion)
**Creator/Channel:** Anthropic (official GitHub-hosted interactive course, Claude 3 Haiku based)
**Difficulty:** Beginner → Advanced (9 chapters, difficulty increases per chapter)
**Rating:** ★★★★★

**Why it's worth watching (working through it):** Although distributed primarily as a Jupyter/
Google-Sheets interactive tutorial rather than a single video, Anthropic's own applied-AI team
built this as the definitive hands-on walkthrough of prompt structure, role assignment,
separating data from instructions, output formatting, chain-of-thought ("precognition"), and a
dedicated chapter on avoiding hallucinations — mapping almost one-to-one onto this module's three
transcript topics. Chapter 8 ("Avoiding Hallucinations") and Chapter 6 ("Precognition") are the
most directly relevant.

**Complements:** All three parts — context/structure in early chapters, hallucination reduction in
Chapter 8, and the appendix on tool use foreshadows module 17 (agents).

**Link:** https://github.com/anthropics/prompt-eng-interactive-tutorial

---

### State of GPT
**Creator/Channel:** Andrej Karpathy (talk given at Microsoft Build 2023; hosted on Karpathy's
own YouTube channel and Microsoft Developer channel)
**Difficulty:** Intermediate
**Rating:** ★★★★☆

**Why it's worth watching:** Karpathy walks through the LLM training pipeline (pretrain → SFT →
RLHF) and then pivots to a widely cited section on prompting strategies — few-shot prompting,
chain-of-thought, "let the model think," retrieval augmentation, and tool use — framed through the
lens of what the model actually is (a token-prediction engine) rather than folk heuristics. This
intuition ("why does chain-of-thought work at all") is exactly the kind of grounding this module
asks for before mechanics.

**Complements:** Part 1 (how context windows/tokens shape what a model can attend to) and the
conceptual bridge into chain-of-thought and ReAct-style tool use.

**Note:** Search "Andrej Karpathy State of GPT" — it circulates on both Karpathy's channel and
Microsoft's; exact runtime varies by re-upload, so no duration is claimed here.

---

### DSPy: Programming, not prompting, Foundation Models (talk/tutorial)
**Creator/Channel:** Omar Khattab and the Stanford NLP / DSPy team (conference talks and official
project walkthroughs, e.g. at Databricks/MLOps community events)
**Difficulty:** Advanced
**Rating:** ★★★★☆

**Why it's worth watching:** Directly supports this module's coverage of automatic prompt
optimization. DSPy's own maintainers present the core argument for why hand-tuned prompt strings
become a maintenance liability at scale, and how a compiler (MIPROv2, SIMBA, GEPA optimizers) can
search the space of instructions and few-shot demonstrations automatically against a metric —
turning "prompt engineering" into "prompt compiling." Useful as a preview of where the discipline
is heading, and as a contrast case against the manual 5-section structuring taught earlier in the
module.

**Complements:** The "when automatic optimization beats manual iteration" section; also a good
forward-reference for module 09 (statistical evaluation) since DSPy optimizers require a metric
function to optimize against.

**Link (project, with linked talks/videos):** https://github.com/stanfordnlp/dspy

---

## Notes on searching for more

Because YouTube's own catalog changes and re-uploads frequently, treat the above as anchors and
supplement with: OpenAI DevDay prompting/structured-outputs sessions (OpenAI's official YouTube
channel), and Anthropic's official "Applied AI" talks on prompt engineering (published on
Anthropic's YouTube channel and at anthropic.com/engineering). Verify current links before
sharing with learners, since course platforms periodically restructure URLs (as OpenAI did in
2026, moving developer docs from platform.openai.com to developers.openai.com).
