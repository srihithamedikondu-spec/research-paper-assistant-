# arXiv Research Assistant

A conversational agent that loads an arXiv paper, indexes its full text, and then answers questions about it, summarizes it, and quizzes you on it. Answers come from the paper's own text, and quiz answers are graded in Python code, not by the model.

## Features

| Capability - What it does 
| Load - Finds a paper on arXiv by its exact title, downloads the PDF, and indexes the full text 
| Ask -Answers specific questions using the most relevant passages from the paper 
| Summarize - Summarizes the whole paper: contribution, methods, and key findings 
| Quiz - Generates multiple-choice questions from the paper, then grades your answers and explains each one 

## How it works

```
paper title -> arXiv PDF -> text -> 1000-character chunks -> Gemini embeddings -> ChromaDB
                                                                                     |
user message -> LangChain agent (Gemini + conversation memory) -> picks a tool ------+
```

The agent has four tools, and it decides which one to call from each message:

| Tool - Purpose 

| `load_paper` - Fetch, parse, chunk, embed, and store a paper 
| `ask_paper` - Retrieve the top matching chunks and answer a question 
| `summarize_paper` - Read across all chunks and write a summary 
| `quiz_paper` - Generate a quiz, or grade the user's answers to the current quiz 

## Design decisions

- Honest failures.- Every tool returns an explicit error message when it fails, and the system prompt forbids answering from the model's general knowledge as a substitute. This fixed a real failure mode where the agent covered a broken tool call with a confident answer from memory.
- The answer key is never in the chat.- The quiz is saved in Python state and graded by a plain function that compares letters, so the model cannot leak the answers or grade from memory.
- Load, then ask.- After loading a paper, the agent stops and asks what to do next instead of chaining tools on its own.
- Safe state handling.- State is only updated after a load fully succeeds, so a failed load does not wipe a paper that was already loaded.
- Fresh index per paper.- The Chroma collection is recreated on every load, because persistent collections keep old data between runs.
- Whole-paper summaries.- Summaries read across all chunks, not just the top few matches.

## Example session

```
You: Can you summarize the paper Probalistic Explanations for Linear Models.
     tools: load_paper, summarize_paper
You: What is the method greedy feature selection?
     tools: ask_paper
You: Give me a quiz with 3 questions on this paper.
     tools: quiz_paper
You: 1-A, 2-B, 3-A
     tools: quiz_paper (grading)  ->  Score: 1/3, with an explanation for each question
```

The agent corrected the misspelled title before searching arXiv. The score came from the answer key stored in Python.

## Tech stack

- Agent and memory: LangChain (`create_agent`) and LangGraph (`InMemorySaver`)
- Indexing and retrieval: LlamaIndex with ChromaDB as the vector store
- Models: Gemini (`gemini-3.5-flash` for chat, `gemini-embedding-001` for embeddings)
- Data: `arxiv` for search, `pypdf` for PDF text extraction

## Getting started

1. Get a Gemini API key from [Google AI Studio](https://aistudio.google.com/).
2. Open the notebook in Google Colab (*File -> Open notebook -> GitHub*, then paste this repo's URL).
3. Add the key as a Colab secret named `GOOGLE_API_KEY` and enable notebook access.
4. Run the install cell, which uses the packages in `requirements.txt`.
5. Run the remaining cells from top to bottom, then start the chat cell.

Tips:
- Use the exact arXiv title when loading a paper.
- Ask for one thing at a time: a summary, a question, or a quiz.
- To answer a quiz, reply with letters, for example `1-A, 2-C, 3-B`.

## Limitations

- One paper and one user at a time, because the loaded paper is shared state.
- Conversation memory lives in RAM and resets when the runtime restarts.
- Papers are matched by exact title.
- Free-tier Gemini limits can cause 429 or 503 errors. Summaries are the heaviest request.

## Planned improvements

- Evaluations: retrieval hit rate on a hand-built question set, tool-selection checks, and unit tests for quiz grading
- Per-session state so several users can use it at once
- A FastAPI backend and a Streamlit interface
- Caching downloaded papers to avoid repeated arXiv requests
