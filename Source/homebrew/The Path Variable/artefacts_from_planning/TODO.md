### Revised and Prioritized Steps for Minimum Viable ColmCast Discord Chatbot

**Phase 1: Core Infrastructure & Basic Discord-n8n-RAG Flow**

1.  **Integrate `colmcast_service` into `devstack`:**
    *   Add the `colmcast_service` as a new service in `devstack/docker-compose.yml`.
    *   Ensure it's on the `cc-net` network so n8n can communicate with it.
    *   Configure its `Dockerfile` and `requirements.txt` for the FastAPI app.
    *   Expose its port (e.g., 8000) within the Docker network.
2.  **Verify `colmcast_service` RAG Functionality:**
    *   Ensure `colmcast_service/build_index.py` can successfully build the FAISS index. (This is a prerequisite for RAG).
    *   Test the `/query_rules` endpoint of the `colmcast_service` directly to confirm it returns relevant passages.
3.  **Set up n8n Discord Integration:**
    *   **Discord Bot Registration**: Confirm a Discord Bot is registered and a Bot Token is available.
    *   **Connect Discord to n8n**: In n8n, install Discord nodes (Trigger, Send). Configure Discord credentials using the Bot Token.
4.  **Create Basic Discord-n8n-RAG Workflow in n8n:**
    *   **Trigger**: Use a Discord Message Create trigger (for DMs to the bot or specific channel messages).
    *   **Action (Call `colmcast_service` for RAG)**: Use an HTTP Request node in n8n to call the `colmcast_service`'s `/query_rules` endpoint, passing the user's message as the query.
    *   **Action (Call OpenAI via `colmcast_service`)**: The `colmcast_service/chat_utils.py` already handles calling OpenAI with RAG context. We need to ensure `chat_utils.handle_message` is exposed via an endpoint in `main.py` that n8n can call.
    *   **Response**: Use a Discord Send node to send the response from `colmcast_service` back to Discord.

**Phase 2: Streamlining `colmcast_service` for MVP**

1.  **Refactor `colmcast_service/main.py`**:
    *   Add a new endpoint (e.g., `POST /chat`) that takes a user message and user ID.
    *   This endpoint should call `chat_utils.handle_message` and return its response.
2.  **Simplify `colmcast_service/chat_utils.py` for MVP**:
    *   Temporarily remove or bypass the character creation state machine logic (riddle, persona selection, PDF parsing) within `handle_message`. The `dnd-character` library will not be actively used for character generation in the MVP.
    *   Adjust the `SYSTEM_PROMPT` to focus purely on being a D&D assistant that uses RAG, without the character creation guidance or riddle.
    *   Ensure `PLAYER_PROFILES` and `CHARACTER_SESSIONS` are handled gracefully, perhaps by initializing a default "generic" profile if none exists, or by simply not using character-specific data for the MVP.
3.  **Remove Standalone Discord Bot**:
    *   The `colmcast_service/discord_chatbot.py` will no longer be needed as n8n will handle Discord integration. This file can be removed or commented out.

**Phase 3: Testing and Refinement**

1.  **End-to-End Testing**:
    *   Test the full flow: Discord message -> n8n -> `colmcast_service` (RAG + OpenAI) -> n8n -> Discord reply.
    *   Verify that RAG context is correctly retrieved and used by OpenAI.
    *   Confirm the bot responds in its persona (as defined by the simplified `SYSTEM_PROMPT`).
2.  **Error Handling & Logging**:
    *   Ensure robust error handling and logging are in place for all components (n8n, `colmcast_service`).

Later:
---update dnd-character library with extended races, classes, backgrounds, etc. found in RAGdb (esp. Unearthed Arcana)