# DocuBot Model Card

This model card is a short reflection on your DocuBot system. Fill it out after you have implemented retrieval and experimented with all three modes:

1. Naive LLM over full docs  
2. Retrieval only  
3. RAG (retrieval plus LLM)

Use clear, honest descriptions. It is fine if your system is imperfect.

---

## 1. System Overview

**What is DocuBot trying to do?**  
DocuBot is a documentation assistant designed to answer developer questions using local project documentation. It compares naive LLM generation, retrieval-only search, and retrieval-augmented generation (RAG) to determine how providing relevant evidence improves the reliability of AI-generated answers.



**What inputs does DocuBot take?**  
For example: user question, docs in folder, environment variables.

DocuBot takes developer questions, documentation stored in the docs/ folder, and a Gemini API key for LLM-supported modes.
**What outputs does DocuBot produce?**

DocuBot produces AI-generated answers in Naive LLM and RAG modes, or relevant documentation results in Retrieval Only mode. It should also refuse unsupported questions when meaningful evidence is unavailable.
---

## 2. Retrieval Design

**How does your retrieval system work?**  
Describe your choices for indexing and scoring.

- How do you turn documents into an index?
- How do you score relevance for a query?
- How do you choose top snippets?

>Indexing: The build_index() function reads the loaded documents, converts words to lowercase, removes basic punctuation, and creates an inverted index that maps words to the filenames containing them.

>Relevance Scoring: The score_document() function compares meaningful words from the user's question with words in each documentation section. It removes common words and counts matching words to calculate a relevance score. Additional weight is given to matching words in section headings.

>Selecting Top Snippets: The split_document() function divides documents into smaller sections using Markdown headings. The retrieve() function scores each section, filters out results below the minimum relevance threshold of two, sorts the remaining results from highest to lowest score, and returns the top three snippets by default.

**What tradeoffs did you make?**  
For example: speed vs precision, simplicity vs accuracy.

> I chose a simple keyword-based retrieval system because it is easier to understand, implement, and debug. However, keyword matching does not fully understand the meaning of a question, so it may miss relevant information when different words are used.

> Splitting documents into smaller sections improves precision and reduces unnecessary information, but it can also separate important details from their surrounding context.

---

## 3. Use of the LLM (Gemini)

**When does DocuBot call the LLM and when does it not?**  
Briefly describe how each mode behaves.

- Naive LLM mode:
- Retrieval only mode:
- RAG mode:

Naive LLM mode: Sends the user's question and the entire documentation collection to Gemini without retrieving specific snippets first.

Retrieval only mode: Uses Python to search and return relevant documentation sections without calling Gemini.

RAG mode: Retrieves relevant documentation snippets first and sends them to Gemini to generate an answer. If no snippets meet the relevance threshold, DocuBot refuses to answer without calling Gemini.



**Observation:** RAG provided a clearer answer with a documentation reference, but it was incomplete because the retrieval system did not supply the most relevant section. This demonstrates that RAG reduces unsupported generation but still depends on retrieval quality.

**What instructions do you give the LLM to keep it grounded?**  
Summarize the rules from your prompt. For example: only use snippets, say "I do not know" when needed, cite files.

> In RAG mode, Gemini is instructed to answer questions using only the retrieved documentation snippets. It should avoid making unsupported assumptions, acknowledge when there is insufficient information, and identify the documentation files used to support its answers.

DocuBot also checks whether relevant snippets are available before sending the question to Gemini. If no snippets meet the relevance threshold, it returns "I do not know based on these docs."

---

## 4. Experiments and Comparisons

Run the **same set of queries** in all three modes. Fill in the table with short notes.

You can reuse or adapt the queries from `dataset.py`.


| Query | Naive LLM | Retrieval Only | RAG | Notes |
|---|---|---|---|---|
| Where is the auth token generated? | Not helpful | Partially helpful | Partially helpful | Naive LLM reported insufficient information. Retrieval found authentication documentation, but RAG identified `/api/login` instead of the `generate_access_token()` function in `auth_utils.py`. |
| How do I connect to the database? | Harmful | Helpful | Helpful | Naive LLM generated generic PostgreSQL instructions. Retrieval and RAG correctly identified `DATABASE_URL`, SQLite, and PostgreSQL configuration from `DATABASE.md`. |
| Which endpoint lists all users? | Not yet tested | Not yet verified | Helpful | RAG correctly identified `GET /api/users` from `API_REFERENCE.md`, but omitted the admin-only requirement. |
| How does a client refresh an access token? | Not yet tested | Not yet verified | Helpful but incomplete | RAG identified `/api/refresh` from `AUTH.md` but omitted details about the existing token. |
| What is the weather on Mars? | Not yet tested | Helpful refusal | Helpful refusal | Both Retrieval Only and RAG correctly refused because the question was unrelated to the documentation. |
**What patterns did you notice?**  

- When does naive LLM look impressive but untrustworthy?  
- When is retrieval only clearly better?  
- When is RAG clearly better than both?
Naive LLM: Naive generation sometimes produced detailed but unsupported answers. For example, it generated generic database connection instructions that did not match the project's documented configuration. It also failed to identify the token-generation function despite the documentation containing that information.

Retrieval Only: Retrieval was more useful for finding information directly from project documentation. It identified the database configuration and authentication information without generating unsupported explanations. However, it sometimes selected sections related to the question, but those did not contain the most precise answer.

RAG: RAG combined retrieved documentation with clearer explanations. It correctly answered questions about database configuration, user endpoints, and token refreshing. It also refused an unrelated question about the weather on Mars. However, it produced an incomplete answer during token generation because the retrieval system did not prioritize the most relevant section of the documentation.

---

## 5. Failure Cases and Guardrails

**Describe at least two concrete failure cases you observed.**  
For each one, say:

- What was the question?  
- What did the system do?  
- What should have happened instead?

- Question 1 : How do I connect to the database?

What did the system do? The Naive LLM generated generic PostgreSQL connection instructions, including environment variables and JavaScript examples, instead of following the project's documentation.

What should have happened? The system should have identified the DATABASE_URL environment variable and explained that SQLite is used by default, while PostgreSQL can be configured.

- Question 2: Where is the auth token generated?

What did the system do? RAG identified the POST /api/login endpoint and cited API_REFERENCE.md, but did not identify the actual token-generation function.

What should have happened? RAG should have retrieved the Token Generation section from AUTH.md and identified the generate_access_token() function in auth_utils.py.

**When should DocuBot say “I do not know based on the docs I have”?**  
Give at least two specific situations.

DocuBot should refuse to answer in two situations:

1. When the question is unrelated to the available documentation. For example, when I asked about the weather on Mars, DocuBot correctly returned "I do not know based on these docs."

2. When the retrieved documentation does not provide enough meaningful evidence to answer the question reliably. Rather than generating unsupported information, DocuBot should acknowledge that the answer cannot be determined from the available documentation.

**What guardrails did you implement?**  
I implemented a minimum relevance threshold of two matching words to filter out weak retrieval results. I also limited retrieval to the top three most relevant sections of documentation.

If no snippets meet the relevance threshold, DocuBot returns "I do not know based on these docs" instead of generating an unsupported answer.



---

## 6. Limitations and Future Improvements

**Current limitations**  
List at least three limitations of your DocuBot system.

1. Keyword Matching: DocuBot relies on matching words between questions and documentation. It may miss relevant information when different words have similar meanings.

2. Retrieval Accuracy: DocuBot does not always select the most relevant section. For example, it identified the login endpoint rather than the generate_access_token() function when asked where the authentication token is generated.

3. Incomplete RAG Answers: RAG can still provide incomplete answers when the retrieved snippets lack important details. Its accuracy depends on the quality of the retrieval results.

**Future improvements**  
List two or three changes that would most improve reliability or usefulness.

1. Semantic Search: Implement semantic search to help DocuBot understand related words and retrieve information based on meaning rather than exact keyword matches.

2. Improve Relevance Scoring: Refine the scoring system to prioritize more relevant sections of documentation and reduce unnecessary results.


---

## 7. Responsible Use

**Where could this system cause real world harm if used carelessly?**  
Think about wrong answers, missing information, or over trusting the LLM.

DocuBot could cause real-world harm if developers rely on incorrect or incomplete answers without verifying the documentation. For example, incorrect authentication or database instructions could introduce security vulnerabilities, application errors, or data loss. Even though RAG uses retrieved documentation, it can still overlook important information. 

**What instructions would you give real developers who want to use DocuBot safely?**  
Write 2 to 4 short bullet points.

- Verify important answers against the original documentation before implementing changes.

- Test AI-generated code and configuration instructions instead of assuming they are correct.

---
