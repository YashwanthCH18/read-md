# Microservices Deep Dive: The Ultimate Guide

This document explains **EVERYTHING** about your 4 microservices. It is the manual for your team.

---

## 1. Gateway Service (The Brain) - Person A
**Role**: The Boss. The Public Face. The Coordinator.
**Tech**: Python (FastAPI).
**Responsibility**: You talk to the User (Frontend) and order the other services around. You ensure data consistency.

### Endpoints (The Commands You Accept)

#### A. Node Management (CRUD)
*   **`POST /nodes`**
    *   **Goal**: "Create a new piece of knowledge."
    *   **Logic**:
        1.  Check if [embedding](file:///C:/Users/Yashwanth%20C%20H/.gemini/antigravity/brain/637e50bb-9ab4-434b-8330-54acd75d2ea0/ingestion/main.py#23-27) is provided. If not, call **Ingestion** (`POST /embed`) to get it.
        2.  Call **Vector Service** (`POST /vectors`) to save the vector.
        3.  Call **Graph Service** (`POST /nodes`) to save the metadata.
        4.  Return success only if *both* succeed.
*   **`GET /nodes/{id}`**
    *   **Goal**: "Get details about a specific node."
    *   **Logic**: Call **Graph Service** (`GET /nodes/{id}`). (Vector service isn't needed for reading properties).
*   **`PUT /nodes/{id}`**
    *   **Goal**: "Update a node."
    *   **Logic**:
        1.  If text changed, call **Ingestion** to get new vector.
        2.  Update **Vector Service** (new vector).
        3.  Update **Graph Service** (new properties).
*   **`DELETE /nodes/{id}`**
    *   **Goal**: "Delete a node."
    *   **Logic**:
        1.  Call **Vector Service** (`DELETE /vectors/{id}`).
        2.  Call **Graph Service** (`DELETE /nodes/{id}`).

#### B. Edge Management (Connecting Things)
*   **`POST /edges`**
    *   **Goal**: "Connect two nodes."
    *   **Logic**: Forward directly to **Graph Service** (`POST /edges`).
*   **`GET /edges/{id}`**
    *   **Goal**: "Read a connection."
    *   **Logic**: Forward directly to **Graph Service**.

#### C. Search (The Magic)
*   **`POST /search/hybrid`**
    *   **Goal**: "Find the best answer using both brains."
    *   **Logic**:
        1.  Get Query Vector from **Ingestion**.
        2.  Get Top Matches from **Vector Service**.
        3.  Get Graph Context for those matches from **Graph Service**.
        4.  **Math**: `Score = (VectorScore * alpha) + (GraphScore * beta)`.
        5.  Return ranked list.

#### D. Ingestion (The Pipeline)
*   **`POST /ingest/file`**
    *   **Goal**: "Process a raw file."
    *   **Logic**:
        1.  Send file to **Ingestion Service**.
        2.  Receive list of Chunks.
        3.  Loop through chunks and do the **Dual Write** (Save to Vector + Save to Graph).

---

## 2. Ingestion Service (The Worker) - Person B
**Role**: The Factory. The Translator.
**Tech**: Python (FastAPI) + `sentence-transformers` + `PyPDF2`.
**Responsibility**: CPU-heavy tasks. You turn "Dumb Text" into "Smart Vectors".

### Endpoints
*   **`POST /process/file`**
    *   **Input**: A PDF or Text file.
    *   **Logic**:
        1.  Extract text from PDF.
        2.  **Chunking**: Split text into 500-character blocks (overlapping by 50 chars).
        3.  **Embedding**: Run `model.encode(chunk_text)` to get 384 floats.
    *   **Output**: List of objects `{ text, vector, metadata }`.
*   **`POST /embed`**
    *   **Input**: A single string (e.g., a search query).
    *   **Logic**: Run `model.encode(text)`.
    *   **Output**: `{ vector: [...] }`.

---

## 3. Vector Service (The Librarian) - Person C
**Role**: The Storage (Similarity).
**Tech**: Python (FastAPI) + `qdrant-client`.
**Responsibility**: managing Qdrant. You answer "What is similar to this?"

### Endpoints
*   **`POST /vectors`**
    *   **Input**: List of points `{ id, vector, payload }`.
    *   **Logic**: `client.upsert(collection_name, points)`.
*   **`POST /search/vector`**
    *   **Input**: Query vector.
    *   **Logic**: `client.search(collection_name, query_vector, limit=k)`.
    *   **Output**: List of `{ id, score }`.
*   **`DELETE /vectors/{id}`**
    *   **Logic**: `client.delete(collection_name, points_selector=[id])`.

---

## 4. Graph Service (The Detective) - Person D
**Role**: The Storage (Structure).
**Tech**: Python (FastAPI) + `neo4j-driver`.
**Responsibility**: Managing Neo4j. You answer "How is this connected?"

### Endpoints
*   **`POST /nodes`**
    *   **Input**: `{ id, label, properties }`.
    *   **Logic**: Cypher `CREATE (n:Label {id: $id, ...})`.
*   **`POST /edges`**
    *   **Input**: `{ source, target, type }`.
    *   **Logic**: Cypher `MATCH (a {id: $s}), (b {id: $t}) CREATE (a)-[:TYPE]->(b)`.
*   **`GET /search/graph`**
    *   **Input**: `start_id`.
    *   **Logic**: Cypher `MATCH (n {id: $id})-[r]-(m) RETURN n, r, m`.
    *   **Output**: The local network around the node.

---

## 5. Scenarios (Putting it all together)

### Scenario 1: The "Knowledge Upload"
**User**: Uploads a PDF about "Photosynthesis".
1.  **Gateway** sends PDF to **Ingestion**.
2.  **Ingestion** returns 10 chunks. Chunk 1 says "Plants use sunlight...".
3.  **Gateway** generates `ID="CHUNK_001"`.
4.  **Gateway** tells **Vector Service**: "Save `CHUNK_001` with vector `[0.1, 0.5...]`".
5.  **Gateway** tells **Graph Service**: "Create Node `CHUNK_001` with text 'Plants use sunlight...'".
6.  **Gateway** tells **Graph Service**: "Create Edge from `FILE_PHOTOSYNTHESIS` to `CHUNK_001`".
**Result**: The knowledge is safely stored in both brains.

### Scenario 2: The "Smart Search"
**User**: Asks "How do plants eat?"
1.  **Gateway** asks **Ingestion** to embed "How do plants eat?".
2.  **Ingestion** returns vector `[0.1, 0.4...]`.
3.  **Gateway** asks **Vector Service**: "Find matches for `[0.1, 0.4...]`".
4.  **Vector Service** replies: "I found `CHUNK_001` (Score: 0.95)".
5.  **Gateway** asks **Graph Service**: "What is connected to `CHUNK_001`?".
6.  **Graph Service** replies: "It belongs to the 'Biology' document and is related to 'Sunlight'".
7.  **Gateway** sees the strong connection to 'Biology' and boosts the score.
8.  **Gateway** returns the answer to the user.

### Scenario 3: The "Correction"
**User**: "Wait, that chunk has a typo." (Updates Node).
1.  **Gateway** receives `PUT /nodes/CHUNK_001` with new text.
2.  **Gateway** asks **Ingestion** for a *new* vector for the fixed text.
3.  **Gateway** tells **Vector Service**: "Overwrite `CHUNK_001` with this new vector."
4.  **Gateway** tells **Graph Service**: "Update the text property of `CHUNK_001`."
**Result**: Both databases are instantly updated and synchronized.
