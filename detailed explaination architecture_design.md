# Vector + Graph Native Database - Deep Dive Architecture

## 1. The "Squad" Structure & Connections
This system is built by **4 People**, each owning one "Box". The boxes connect via **HTTP REST APIs**.

*   **Person A (Gateway)**: The "Boss". You define the rules. Everyone listens to you. You talk to everyone.
*   **Person B (Ingestion)**: The "Worker". You take raw stuff and make it useful. Only Person A talks to you.
*   **Person C (Vector)**: The "Librarian". You organize books by similarity. Only Person A talks to you.
*   **Person D (Graph)**: The "Detective". You connect clues. Only Person A talks to you.

### How They Connect (The "Wiring")
All services run in Docker. They talk using internal URLs (defined in [docker-compose.yml](file:///C:/Users/Yashwanth%20C%20H/.gemini/antigravity/brain/637e50bb-9ab4-434b-8330-54acd75d2ea0/docker-compose.yml)):
*   Gateway can reach Ingestion at `http://ingestion:8001`
*   Gateway can reach Vector at `http://vector_service:8003`
*   Gateway can reach Graph at `http://graph_service:8004`

---

## 2. Service Contracts (The "Language")
To make this work, you must agree on exactly what JSON you send to each other.

### Contract 1: Gateway -> Ingestion (Person A -> Person B)
**Goal**: "Here is a file, give me chunks and vectors."
*   **Request** (`POST /process/file`): Multipart Form Data (The PDF file).
*   **Response** (What Person B must return):
    ```json
    [
      {
        "text": "Artificial Intelligence is...",
        "vector": [0.12, -0.05, ...],  // 384 floats
        "metadata": { "page": 1, "source": "paper.pdf" }
      },
      ...
    ]
    ```

### Contract 2: Gateway -> Vector Service (Person A -> Person C)
**Goal**: "Save these vectors" OR "Find similar vectors."
*   **Upsert Request** (`POST /vectors`):
    ```json
    {
      "points": [
        {
          "id": "uuid-1234",
          "vector": [0.12, ...],
          "payload": { "text": "...", "source": "..." }
        }
      ]
    }
    ```
*   **Search Request** (`POST /search/vector`):
    ```json
    { "vector": [0.12, ...], "top_k": 10 }
    ```
*   **Search Response** (What Person C must return):
    ```json
    [ { "id": "uuid-1234", "score": 0.95 }, ... ]
    ```

### Contract 3: Gateway -> Graph Service (Person A -> Person D)
**Goal**: "Save these nodes" OR "Traverse the graph."
*   **Create Node Request** (`POST /nodes`):
    ```json
    {
      "id": "uuid-1234", // MUST match the ID sent to Vector Service
      "label": "Chunk",
      "properties": { "text": "...", "source": "..." }
    }
    ```
*   **Traversal Request** (`GET /search/graph`):
    *   Query Param: `?start_id=uuid-1234`
*   **Traversal Response** (What Person D must return):
    ```json
    {
      "nodes": [ ... ],
      "relationships": [ { "source": "uuid-1234", "target": "uuid-5678", "type": "RELATED_TO" } ]
    }
    ```

---

## 3. Detailed Scenarios (Step-by-Step)

### Scenario A: The "Ingestion" Pipeline (Uploading Data)
*User uploads a PDF about "SpaceX".*

1.  **Gateway (Person A)** receives the PDF.
    *   *Action*: "I need to process this."
    *   *Call*: Sends PDF to **Ingestion (Person B)**.
2.  **Ingestion (Person B)** works.
    *   *Action*: Extracts text "SpaceX launches rockets...", splits it into 5 chunks.
    *   *Action*: Calculates vectors for each chunk.
    *   *Return*: Returns list of 5 chunks with vectors to Gateway.
3.  **Gateway (Person A)** coordinates the "Dual Write".
    *   *Action*: Generates 5 UUIDs (e.g., `id_1`, `id_2`...).
    *   *Call*: Sends `id_1` + [vector](file:///C:/Users/Yashwanth%20C%20H/.gemini/antigravity/brain/637e50bb-9ab4-434b-8330-54acd75d2ea0/storage/main.py#143-159) to **Vector Service (Person C)**.
    *   *Call*: Sends `id_1` + [text](file:///C:/Users/Yashwanth%20C%20H/.gemini/antigravity/brain/637e50bb-9ab4-434b-8330-54acd75d2ea0/gateway/main.py#24-46) to **Graph Service (Person D)**.
    *   *Wait*: Waits for both C and D to say "OK".
4.  **Result**: The system now remembers "SpaceX" semantically (Vector) and structurally (Graph).

### Scenario B: The "Hybrid Search" (The Complex One)
*User asks: "What rockets does SpaceX use?"*

1.  **Gateway (Person A)** receives query.
    *   *Action*: Needs a vector for the query.
    *   *Call*: Sends text "What rockets..." to **Ingestion (Person B)**.
    *   *Return*: B returns `query_vector`.
2.  **Gateway (Person A)** asks the Librarian (Vector Search).
    *   *Call*: Sends `query_vector` to **Vector Service (Person C)**.
    *   *Action (Person C)*: Qdrant finds top 10 matches (e.g., `id_1` (SpaceX), `id_5` (Starship)).
    *   *Return*: C returns `[id_1, id_5]`.
3.  **Gateway (Person A)** asks the Detective (Graph Search).
    *   *Action*: "Okay, `id_1` is a match. What is connected to it?"
    *   *Call*: Sends `id_1` to **Graph Service (Person D)**.
    *   *Action (Person D)*: Neo4j looks at [(id_1)-[:USES]->(Rocket)](file:///C:/Users/Yashwanth%20C%20H/.gemini/antigravity/brain/637e50bb-9ab4-434b-8330-54acd75d2ea0/storage/main.py#35-39). Finds "Falcon 9".
    *   *Return*: D returns "Falcon 9" node and connection strength.
4.  **Gateway (Person A)** thinks (Ranking).
    *   *Logic*: "Vector said `id_1` is 90% match. Graph confirmed it's connected to 'Rocket'. This is a great answer."
    *   *Calculation*: `0.9 * 0.5 + 1.0 * 0.5 = 0.95`.
5.  **Result**: Gateway returns the combined answer to the user.

### Scenario C: "Delete a Node" (Maintenance)
*User wants to delete the "SpaceX" document.*

1.  **Gateway (Person A)** receives `DELETE /nodes/id_1`.
2.  **Gateway (Person A)** ensures consistency.
    *   *Call*: Tells **Vector Service (Person C)**: "Delete `id_1`". (C removes it from Qdrant).
    *   *Call*: Tells **Graph Service (Person D)**: "Delete `id_1`". (D removes it from Neo4j).
3.  **Result**: Data is gone from both places. No "ghost" data left behind.

---

## 4. Why this is the BEST for 4 People
*   **Person A** writes the logic in [main.py](file:///C:/Users/Yashwanth%20C%20H/.gemini/antigravity/brain/637e50bb-9ab4-434b-8330-54acd75d2ea0/storage/main.py) of Gateway. You don't care *how* vectors are stored, you just send them.
*   **Person B** plays with PyTorch and Transformers in `ingestion/`. You don't care about databases.
*   **Person C** learns Qdrant API in `vector_service/`. You don't care about the text content.
*   **Person D** writes Cypher queries in `graph_service/`. You don't care about embeddings.

**Everyone has a clear job. The "Contracts" (JSON format) are the only thing you need to agree on before coding.**
