# StudyBuddy — AI Learning Partner

An AI-powered study assistant that quizzes you on your own lecture materials using RAG and LLM feedback.

## How it works

1. Upload your lecture notes or slides (PDF, Word, or plain text)
2. StudyBuddy extracts the main topics automatically
3. Select a topic — StudyBuddy asks you a comprehension question based on your material
4. Explain the concept in your own words
5. Get instant feedback and targeted follow-up questions until the topic is mastered

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/your-username/study-buddy.git
cd study-buddy
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. Configure your API key

Copy the example env file and fill in your credentials:

```bash
cp .env.example .env
```

Then open `.env` and set your values:

```
API_KEY=your-api-key-here
BASE_URL=https://chat.kiconnect.nrw/api/v1
MODEL=OpenAI-GPT-5-Mini
```

> If you are using a different OpenAI-compatible API, replace `BASE_URL` and `MODEL` accordingly.

### 4. Run the app

Open your terminal (Command Prompt or PowerShell) and navigate to the project folder:

```bash
cd "C:\path\to\study-buddy"
```

For example, if you cloned it to your D drive:

```bash
cd "D:\study-buddy"
```

Then run:

```bash
streamlit run app.py
```

The app will open automatically in your browser at `http://localhost:8501`.

## Supported file types

- PDF (`.pdf`)
- Word (`.docx`)
- Plain text (`.txt`)

```mermaid
flowchart TB
    subgraph FE ["Streamlit Frontend"]
        APP["app.py<br/>state machine · chat · sidebar<br/>dialogs · model selector · usage"]
        AUI["auth_ui.py<br/>login / signup / logout"]
    end

    subgraph CORE ["Application Core"]
        AG["agent/study_agent.py<br/>StudyBuddyAgent"]
        PR["agent/prompts.py<br/>prompt templates"]
        MO["agent/models.py<br/>model catalogue"]
        DL["tools/document_loader.py<br/>PDF · DOCX · TXT → text"]
        RG["tools/rag.py<br/>chunking · embeddings · FAISS"]
    end

    subgraph AUTH ["Auth & Persistence"]
        AU["auth.py<br/>bcrypt · login sessions"]
        HI["history.py<br/>study sessions · document meta"]
        SC[("schema.sql<br/>users · sessions<br/>study_sessions<br/>hidden_topics · llm_usage")]
    end

    subgraph EXT ["External Services"]
        LLM["OpenAI-compatible<br/>LLM endpoint"]
        DB[("Supabase<br/>PostgreSQL")]
        ST["all-MiniLM-L6-v2<br/>sentence-transformer"]
    end

    APP --> AG
    APP --> AUI
    APP --> HI
    AUI --> AU
    AU --> DB
    HI --> DB
    HI --> SC
    AG --> PR
    AG --> LLM
    AG --> RG
    AG --> MO
    RG --> ST
    DL --> AG
    DB -. actual store for .-> SC
```

## 2. Application State Machine

```mermaid
stateDiagram-v2
    [*] --> no_document
    no_document --> document_open : upload / select document
    document_open --> questioning : pick topic
    questioning --> questioning : follow-up
    questioning --> feedback : evaluated
    questioning --> mastered : all concepts covered
    feedback --> questioning : next question
    feedback --> document_open : back to topics
    feedback --> questioning : restart topic
    mastered --> document_open : back to topics
    mastered --> questioning : restart topic
    document_open --> no_document : delete active document
```

## 3. Processing Pipeline

```mermaid
flowchart LR
    A["Upload<br/>PDF · DOCX · TXT"] --> B["Parse<br/>document_loader.py"]
    B -->|"text < 50 chars"| X["Reject: Unparseable"]
    B --> C["Chunk<br/>600 chars · 80 overlap"]
    C --> D["Embed<br/>all-MiniLM-L6-v2"]
    D --> E[("FAISS<br/>IndexFlatIP")]
    E --> F["Topic extraction<br/>LLM"]
    F --> G[("topics_json<br/>stored with document")]
    G --> H["Student picks a topic"]
    H --> I["Concept extraction<br/>LLM · 3–5 key concepts"]
    I --> J[("concepts_json<br/>mastery checklist")]
    J --> K["Question generation<br/>LLM"]
    K --> L["Student answers"]
    L --> M["Evaluation<br/>LLM · JSON"]
    M --> N{"All concepts covered?"}
    N -->|"no"| O["Follow-up<br/>or retry"]
    O --> K
    N -->|"yes"| P["Topic mastered"]
```

## 4. Authentication Flow

```mermaid
sequenceDiagram
    actor U as User
    participant UI as auth_ui.py
    participant A as auth.py
    participant DB as Postgres

    U->>UI: enter email + password
    UI->>A: authenticate(identifier, password)
    A->>DB: SELECT user by email OR username
    DB-->>A: row or None

    alt row is None
        A->>A: bcrypt check against dummy hash
        A-->>UI: LoginError
        UI-->>U: Incorrect email or password.
    else row exists
        A->>A: bcrypt checkpw(password, hash)
        alt password ok
            A-->>UI: user dict
            UI->>A: create_session(user_id)
            A->>DB: INSERT INTO sessions
            UI->>U: set query param, toast, rerun
        else wrong password
            A-->>UI: LoginError
            UI-->>U: Incorrect email or password.
        end
    end
```

## 5. Persistence (ER Diagram)

```mermaid
erDiagram
    USERS ||--o{ SESSIONS : has
    USERS ||--o{ STUDY_SESSIONS : has
    USERS ||--o{ HIDDEN_TOPICS : has
    USERS ||--o{ LLM_USAGE : has
    STUDY_SESSIONS ||--o{ LLM_USAGE : "made during"

    USERS {
        int id PK
        text email UK
        text username UK
        text password_hash
        text preferred_model
        timestamp created_at
    }
    SESSIONS {
        text token PK
        int user_id FK
        timestamp expires_at
        timestamp created_at
    }
    STUDY_SESSIONS {
        int id PK
        int user_id FK
        text document_name
        text document_text
        text topic
        text topics_json
        text concepts_json
        text conversation_json
        text mistakes_json
        text current_concept
        text current_question
        int follow_up_count
        timestamp created_at
        timestamp updated_at
    }
    HIDDEN_TOPICS {
        int id PK
        int user_id FK
        text document_name
        text topic
        timestamp created_at
    }
    LLM_USAGE {
        int id PK
        int user_id FK
        int session_id FK
        text model
        int prompt_tokens
        int completion_tokens
        timestamp created_at
    }
```

## 6. Multi-Model Support

```mermaid
flowchart LR
    USER["User"] -->|"selects model"| SEL["Sidebar<br/>Model selector"]
    SEL -->|"persist"| DB[("users.preferred_model")]
    DB -->|"restore on login"| AGENT["StudyBuddyAgent<br/>agent.model"]
    AGENT -->|"evaluate"| MAIN["Main evaluation<br/>updates progress"]
    AGENT -->|"regenerate"| RQ["New question<br/>from another model"]
    AGENT -->|"second opinion"| SO["Second opinion<br/>on last answer"]
    MAIN --> USAGE[("llm_usage")]
    RQ --> USAGE
    SO --> USAGE
```

## 7. CI Pipeline

```mermaid
flowchart LR
    DEV["git push / MR"] --> CI["GitLab CI"]
    CI --> INST["pip install<br/>(cached)"]
    INST --> PY["pytest tests/"]
    PY -->|"pass"| OK["Pipeline green"]
    PY -->|"fail"| NOK["Pipeline red"]
```
