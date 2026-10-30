# 📚 BookBuddy

A self-learning study assistant that answers questions from your textbook.
Pick a subject and chapter, ask a question, and BookBuddy finds the relevant
pages of the book and explains the answer, with related YouTube videos.

This is a personal learning project, based on
[project-studypal-study-assistant](https://github.com/sarprk/project-studypal-study-assistant).
Currently supported: **Class 12 Biology (NCERT)**.

## How it works

BookBuddy uses RAG (Retrieval-Augmented Generation):

1. **Prepare (once):** chapter PDFs are split into chunks, turned into
   embeddings with a HuggingFace model, and stored in Chroma vector databases.
2. **Ask:** your question is matched against those chunks, and the 3 most
   relevant ones are sent with the question to an LLM on Groq
   (`openai/gpt-oss-120b`), which writes the answer.

```
Textbook PDFs ──► chunks ──► embeddings ──► Chroma DB
                                               │
Your question ──► find 3 matching chunks ──────┘
                          │
                          ▼
            Groq LLM writes the answer ──► Streamlit UI (+ YouTube links)
```

## Project structure

```
src/
  main.py               Streamlit app
  vectorize_script.py   Builds the vector databases from the PDFs
  vectorize_book.py     PDF loading, chunking and embedding helpers
  chatbot_utility.py    Lists chapters for a subject
  get_yt_video.py       YouTube video search
  env_template.txt      Template for src/.env
data/                   Textbook PDFs (not in git)
vector_db/              Whole-book vector database (generated, not in git)
chapters_vector_db/     Per-chapter vector databases (generated, not in git)
```

## Setup

Requires Python 3.11.

1. **Install dependencies**

   ```bash
   python -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
   ```

2. **Add your Groq API key.** Get a free key from
   [console.groq.com](https://console.groq.com/keys), then:

   ```bash
   cp src/env_template.txt src/.env
   # edit src/.env and set GROQ_API_KEY
   ```

3. **Add the textbook PDFs.** The PDFs are not included in this repo because
   NCERT textbooks are copyrighted. Download the Class 12 Biology chapters from
   [ncert.nic.in/textbook.php](https://ncert.nic.in/textbook.php) and save them as:

   ```
   data/class_12/biology/1. Sexual Reproduction in Flowering Plants.pdf
   data/class_12/biology/2. Human Reproduction.pdf
   ...
   data/class_12/biology/13. Biodiversity and its Conservation.pdf
   ```

   File names must start with the chapter number and a dot; that's how the
   app sorts chapters.

4. **Build the vector databases** (takes a few minutes):

   ```bash
   python src/vectorize_script.py
   ```

   Running this again adds duplicate chunks to existing databases. Delete
   `vector_db/` and `chapters_vector_db/` first if you need to rebuild.

5. **Run the app**

   ```bash
   streamlit run src/main.py
   ```

## Troubleshooting

- **`Cannot set non-stroke color ... got [/'P36']`** while vectorizing: a
  harmless pdfminer warning about colour patterns in the PDF. Text extraction
  still works.
- **`model_not_found` from Groq:** Groq retires models from time to time.
  Pick a current chat model from the
  [Groq models list](https://console.groq.com/docs/models) and update it in
  `src/main.py`.
