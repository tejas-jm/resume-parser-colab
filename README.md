
# 🤖 Secure, Layout-Aware Resume Parser

This project provides a robust, end-to-end pipeline for parsing resumes (`.pdf`, `.docx`, `.doc`) into a clean, structured JSON format.

It intelligently handles complex layouts, anonymizes all Personally Identifiable Information (PII) before processing, and uses a Large Language Model (Gemini) to extract and categorize information with high accuracy. The output is then validated against a strict schema (Pydantic) and securely "re-hydrated" with the original data.

## ✨ Core Features

  * **Broad File Support:** Parses `.pdf`, `.docx`, and legacy `.doc` files.
  * **Layout-Aware Parsing:** Uses `unstructured` with a "hi-res" (OCR) strategy to understand document layout, columns, and text blocks, preventing data jumbling.
  * **🔒 Secure PII Anonymization:** Leverages **Presidio** to find and mask all sensitive data (names, emails, phone numbers, custom links) *before* it's sent to any API.
  * **🧠 Intelligent Extraction:** Uses Google's **Gemini** model via LangChain to understand the context of the resume and extract data relationally (e.g., knowing *which* job description belongs to *which* company).
  * **✅ Strict Schema Validation:** Employs **Pydantic** to define a "perfect" JSON schema and automatically validate the LLM's output, ensuring 100% data consistency.
  * **💧 Secure Re-hydration:** The final step securely swaps the anonymized masks (e.g., `<PERSON_1>`) back to their original values (e.g., "Alex Johnson") after processing, so the final JSON is complete.

-----

## 🚀 Workflow

The entire process is a secure, multi-step pipeline:

```mermaid
graph TD
    A[Start: User Uploads File (.pdf, .doc, .docx)] --> B{Step 1: Parse File};
    B --> C[get_file_text() using 'unstructured' hi-res];
    C --> D{Step 2: Anonymize Text};
    D --> E[anonymize_and_get_lookup() using Presidio];
    E --> F[AnalyzerEngine finds PII (PERSON, EMAIL, etc.)];
    F --> G[AnonymizerEngine creates unique masks (e.g., <PERSON_1>)];
    G --> H[Output 1: anonymized_text];
    G --> I[Output 2: lookup_dict {mask: original_value}];
    
    H --> J{Step 3: Parse with LLM};
    J --> K[LangChain chain.invoke() with anonymized_text];
    K --> L[Gemini Model generates JSON string];
    
    L --> M{Step 4: Validate Schema};
    M --> N[PydanticOutputParser validates JSON against ResumeSchema];
    N -- Valid --> O[parsed_masked_model (Pydantic Object)];
    N -- Invalid --> P[Error: Validation Failed];
    
    O --> Q{Step 5: Re-hydrate Data};
    I --> Q;
    Q --> R[rehydrate_model() swaps masks with original values];
    R --> S[final_resume_model (Pydantic Object)];
    
    S --> T{Step 6: Final Output};
    T --> U[Print final_resume_model as clean JSON];
```

-----

## 🛠️ Tech Stack

  * **LLM & Orchestration:** Google Gemini, LangChain
  * **File Parsing:** `unstructured[all-docs]` (with Detectron2 for OCR)
  * **PII Anonymization:** `presidio-analyzer`, `presidio-anonymizer`
  * **Schema Validation:** `pydantic`
  * **System Dependencies:** `libreoffice`, `antiword` (for `.doc` support)

-----

## 🏃‍♂️ How to Run (Google Colab)

This project is designed to run perfectly in a Google Colab notebook.

### 1\. 🔑 Set Up Your Google API Key

For this notebook to work, you need to provide a Google AI Studio API key.

1.  **Get your key:**

      * Go to [Google AI Studio](https://aistudio.google.com/app/apikey).
      * Click **"Create API key"** and copy the key.

2.  **Store it in Colab Secrets:**

      * In your Colab notebook, click the **"Key" icon** (🔑) in the left-hand sidebar.
      * Click **"Add new secret"**.
      * For the **Name**, enter: `GOOGLE_API_KEY`
      * For the **Value**, paste your API key.
      * Make sure the **"Notebook access"** toggle is turned **on**.

### 2\. 📦 Install Dependencies

Run the first cell in the notebook to install all required Python packages and system libraries (`libreoffice`, `antiword`).

### 3\. 🧠 Pre-load Models

Run the second cell to pre-load the `unstructured` OCR and layout models. This makes the first parse much faster.

### 4\. 🚀 Run the Pipeline

Execute the final "Run" cell. It will ask you to upload a resume file. The script will then perform all 5 steps of the workflow and print the final, clean JSON at the end.

-----

## 🧩 Code Breakdown

### `get_file_text()`

Uses `unstructured.partition` with the `strategy="hi_res"`. This function automatically detects the file type (`.pdf`, `.doc`, `.docx`) and uses an OCR model to analyze the document's visual layout, preventing text from different columns from being mixed.

### `anonymize_and_get_lookup()`

Uses a custom-configured Presidio `AnalyzerEngine`. We use a clean `RecognizerRegistry` to *only* search for specific PII (PERSON, EMAIL, PHONE\_NUMBER, GITHUB\_LINK, LINKEDIN\_LINK). This prevents false positives. The function returns two things:

1.  The fully anonymized text (e.g., `"My name is <PERSON_1>"`)
2.  A `lookup_dict` (e.g., `{"<PERSON_1>": "Alex Johnson"}`)

### `ResumeSchema (Pydantic)`

A set of Pydantic models that define the *exact* target JSON structure. This includes all nested objects like `Contact`, `Experience`, and `Skills`. This schema is used to both instruct the LLM and validate its output.

### `chain`

A LangChain pipeline that:

1.  Takes the `anonymized_text`.
2.  Inserts it into a `PromptTemplate`.
3.  Sends the prompt to the **Gemini** model.
4.  Takes the model's raw JSON string output.
5.  Runs it through a **`PydanticOutputParser`** to validate it and convert it into a Python object.

### `rehydrate_model()`

A recursive function that walks through the Pydantic object, finds any string that matches a mask in the `lookup_dict`, and replaces it with the original, sensitive value.

-----

## 🎯 Example JSON Output

The final, re-hydrated output will look like this:

```json
{
  "name": "Alex Johnson",
  "title": "Full Stack Engineer",
  "contact": {
    "email": "alex.johnson@email.com",
    "phone": "+1 (555) 123-4567",
    "address": "123 Main St, Anytown, USA 12345",
    "website": "alexjohnson.dev",
    "linkedin": "linkedin.com/in/alex-j-dev",
    "github": "github.com/AlexJDev"
  },
  "summary": null,
  "education": [
    {
      "institution": "State University",
      "degree": "B.S. in Computer Science",
      "dates": "Aug 2019 - May 2023",
      "score": "3.8 GPA",
      "location": null
    }
  ],
  "experience": [
    {
      "company": "Tech Solutions Inc.",
      "role": "Software Engineer Intern",
      "dates": "May 2022 - Aug 2022",
      "duration": null,
      "location": null,
      "responsibilities": [
        "Developed new features for a client-facing web application using React and Node.js.",
        "Collaborated with senior developers in an Agile environment to fix bugs and improve code quality."
      ]
    }
  ],
  "skills": {
    "languages": ["JavaScript", "Python", "SQL"],
    "frameworks": ["React", "Node.js", "Express"],
    "developer_tools": ["Git", "GitHub", "Docker"],
    "databases": ["PostgreSQL", "MongoDB"],
    "other": ["Agile Methodologies", "Cloud Computing"]
  }
}
```