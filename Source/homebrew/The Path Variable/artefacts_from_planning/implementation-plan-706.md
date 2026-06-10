Implementation Plan: From Slack Bot to Discord Automation
Based on the above evaluation, here is a phased implementation plan to realize ColmCast Chatbot v2:
Phase 1: Environment Setup & Basics
Set up Repositories: Create a new repository (or branch) for ColmCast v2 infrastructure. Organize the code into n8n_workflows/, colmcast_service/ (Python code), etc. Include a README outlining how to build and run.
Dockerize n8n: Using the n8n-hosting example, write a docker-compose.yml with an n8n service. Configure it to use a local volume for /home/node/.n8n (to persist data). Initially, bring up n8n and ensure you can access the UI (likely at http://localhost:5678). Set an admin user/password.
Discord Bot Registration: Create a Discord Bot through the Developer Portal. Give it a name (e.g., “ColmCastBot”). Generate a Bot Token and save it. Enable Privileged Gateway Intents if needed (we might not need presence/member intents for this use case). Add a slash command (in the Developer Portal or via code) for a simple test, like /ping (for initial connectivity testing). Invite the bot to a test server.
Connect Discord to n8n: Install the community Discord Trigger & Send nodes in n8n (if not using webhooks). Configure the Discord credentials in n8n using the Bot Token. Create a simple workflow in n8n: Discord Trigger (for /ping) → Respond with Discord node (“Pong!”). Test this on the Discord server by typing /ping. Adjust any permission issues until the bot responds successfully.
Basic Conversation Workflow: Implement a basic version of the DM chat workflow. For now, it can ignore RAG and the Python service:
Trigger: (Temporary solution) Use a Discord slash command like /ask with a text parameter, to simulate a DM message. (Later we will handle actual DMs).
Action: Use an OpenAI node (or HTTP request) to send a prompt to GPT-4. For testing, a simple system prompt “You are CC, a witty AI.” and user prompt from the slash command text is fine.
Response: Take GPT output, and use Discord Send node to reply (ephemeral or channel message).
Verify that the round-trip works: the model responds and the message appears on Discord.
Phase 2: Python Game Logic Service
Service Scaffold: In colmcast_service/, set up a basic Flask or FastAPI app. Implement one test endpoint, e.g., GET /hello that returns “world” to verify connectivity. Write a Dockerfile and build the image (docker build -t colmcast-api .). Update docker-compose.yml to include this service, and link it with n8n’s network.
Integrate dnd-character: Copy or install the dnd-character library into the service. Ensure the container has all needed dependencies (faiss, numpy, etc.). In the app, add an endpoint /create_character that internally uses dnd_character.Character or class constructors to create a new character. For start, it can create a random Level 1 character with default parameters. Test this by calling the endpoint (e.g., via curl inside the container or an n8n HTTP node) – it should return some character data (JSON or text).
Load FAISS Index: Move the existing FAISS index files (from storage/faiss_index in the Slack bot) into the service container (consider adding them via Dockerfile or mounting a volume). In the app startup, load the index using LlamaIndex as done in rag_retriever.py
GitHub
. Implement a POST /query_rules that accepts a question and returns the retrieved context (the concatenated top 3 passages, for example)
GitHub
. Test this endpoint with a sample question (“What is a concentration spell?”) and see that it returns a relevant snippet (e.g. from the Player’s Handbook rules on concentration).
Extend Character Generation: Expand /create_character to accept parameters – class, level, maybe race/background. Utilize the library to generate appropriately. For example, if class is specified, instantiate that class from dnd_character.classes. If level is given, set it and call .level_up() if needed to reach that level. Include equipment. This will use SRD data for now – we’ll incorporate custom data later.
Persistent Storage (if needed): Decide how to store created characters or selected pregens. A simple approach: the service can keep them in memory (in a dict mapping user ID to Character object). But to persist across restarts, use either a tiny SQLite (since we already have SQLAlchemy via library if needed) or allow n8n to request the full data and store it on its side (not ideal). Perhaps implement endpoints: /characters (GET all or POST new) and /characters/{id} for fetch/update. We can generate a character ID when creating. For the one-shot, an in-memory might suffice, but persistence is safer.
Phase 3: Feature Parity Implementation
Player Profile Integration: Import or recreate the player_profiles.json from Slack bot
GitHub
 into our new system. We can either embed that data into the prompt for GPT (like v1 did) or have the Python service manage some of it. Given the persona prompt already covers a lot, we might store profiles in n8n (in a static data structure or in the Python service as a dict) and retrieve relevant info to append to system prompt or as context. Implement a lookup such that when user X messages, the workflow fetches X’s profile (e.g., “Jordan Lumberpond, Bard, NG, famous musician…” from the JSON) and inserts a line about it into the prompt. This ensures CC’s responses remain tailored
GitHub
.
Lore Data: Similarly, port lore.json (if it contains specific world lore or hints). Make it accessible to the bot – maybe also via the vector index if it’s textual, or as a short context that’s always included. Ensure CC doesn’t reveal secrets unless appropriate (the persona instructions already cover that).
In-Character Chat Workflow: Now assemble the full Chat workflow in n8n:
Trigger: Use a Discord Message Create trigger for DMs to the bot (if using the community trigger, or if not available, fall back to using a slash command like /cc in a server channel that denotes an IC question).
Actions:
Identify the user and their character (if we have a mapping from Discord user to character).
Formulate the system prompt. This will combine: the core persona (from Slack but updated if needed), the player’s profile snippet, possibly a summary of recent conversation (we can retrieve last few messages from an in-memory store or Discord history API), and a placeholder for rule context.
If the message seems to be a rules question or contains a D&D term, call the /query_rules on our service to get relevant info. We could use a simple heuristic or always query it and let the LLM decide to use it. In v1, they always pulled some context (similarity_top_k=3)
GitHub
. We can do the same: send the user’s message to /query_rules, get up to 3 passages, and include them in the prompt as “D&D Rulebook Excerpts:\n<text>”.
Send the composed prompt (system + excerpts + conversation history + user query) to OpenAI (GPT-4) via the OpenAI node.
Take the response, log it (maybe to a file or simply to the n8n execution log), then send it back to the user on Discord.
Testing: Try a sample DM: “Hey CC, can my artificer use a longsword effectively?” Expect the bot to respond with some in-character snark plus the mechanical answer (e.g., explaining artificers are proficient with martial weapons if they have that background, etc., drawn from context).
Dice Roll Workflow: Implement the /roll slash command workflow:
Trigger: Discord slash command /roll with a string parameter (like “1d20+5”).
Action: Use either a small JS function node or call a Python endpoint to parse and roll. For simplicity, we might incorporate a tiny dice library in Python service (there are many on PyPI, or we write a quick parser using regex).
The Python service returns the result (e.g., “Rolled 1d20+5: 13 + 5 = 18”). The n8n workflow then replies with that string in Discord.
Test with various inputs, including edge cases (“3d6”, “d%, etc.). This ensures players have a handy roller. (Even if they roll their own physical dice, it’s nice for CC to do some rolls for secret checks or NPCs.)
Character Management Commands: Implement selection of pregenerated characters:
Perhaps a command /list_characters that lists the available pregens (names and brief intro). This can output an embed with each character’s emoji and summary
GitHub
GitHub
.
/choose_character name: The workflow finds the chosen pregen (we can store preset characters in a JSON or the Python service could have them predefined in a module). It then records this choice (e.g., in a database or in n8n’s memory) tying the Discord user to that character. It confirms to the user (“You have chosen to play Stephanie Efforts, the Artificer.”).
We’ll have to have the stats ready: we might pre-create these characters using the library and store their JSON stat blocks. If time permits, create a script to generate them at level 10 with appropriate gear and manually adjust anything to fit their concept (like custom magic items or specific ability scores to match their archetype). Load these into the Python service (perhaps as part of its startup, from a file).
Once a user has chosen a character, the chat workflow can automatically pull their character’s info for context. For example, we might add to the system prompt: “(Player’s character: Level 10 Artificer with 18 INT, carrying an Arcane Focus Drone)” or even have CC know that info to incorporate in replies (“Ah, a clever Artificer like you, with all your gadgets… [in-character banter]”).
Game-Specific Mechanics: Implement any known special commands needed for our scenario:
If there’s a puzzle where players might ask CC for hints, perhaps a command /hint triggers a workflow that either uses some logic or randomly picks a clue from a list.
If CC is meant to “resist” giving direct answers (as a character), we might implement a threshold – e.g., require a Persuasion roll. In that case, a workflow could intercept certain questions and require a dice roll outcome before proceeding to call GPT (this could be a fun integration: the bot might say “CC eyes you warily, uncertain if he should reveal more…”, then prompt the player to roll persuasion).
Ensure safety nets: we should decide how to handle if GPT produces something inappropriate or lore-breaking. Possibly keep the moderation strategy from v1 (not directly mentioned, but presumably relying on GPT-4’s moderation or our careful prompt). If something goes awry, since we’re there as DM, we can always step in – but as a backup, we could add a check: the workflow can scan GPT’s output for forbidden content or major spoilers (we can maintain a list of true campaign secrets that should not be revealed, and if the answer contains them, either redact or re-roll with a stricter prompt).
Phase 4: Testing and Iteration
Alpha Test: Gather a small group of testers (could be just the DM and one player or friend) to run through a mini-conversation on Discord. Test every command:
DM chat in character, ask both simple and complex questions (lore, rules).
Roll some dice.
Try to break things (ask multi-part questions, spam a few commands concurrently, etc.) to see if the system queues them properly.
Evaluate CC’s persona consistency and adjust the system prompt if needed. For instance, if CC starts revealing too much or too little, tweak the instructions (the system prompt from Slack had detailed do’s and don’ts
GitHub
 – ensure those are all carried over).
Check the timing – Discord interactions might need some adjustments (maybe using ephemeral messages for certain replies).
Beta Test – Full Group: Do a session zero or a prologue with the actual player group in Discord. Let them introduce their characters and maybe have a light conversation with CC. Monitor the logs for any errors or slow points. This will also train players on using the commands. Gather feedback: Are the commands intuitive? Is CC responding too slowly or quickly? Are there requests for additional features (e.g., “Can CC show us an image?” – maybe we integrate an image generation for fun via a future stable diffusion if desired; or “Can CC track our HP?” – maybe a feature to log HP changes).
Tweak & Tune: Based on testing feedback:
Adjust temperature or response length from GPT if needed (maybe CC should be more concise or more poetic).
Fix any bugs (e.g., a crash in Python service if an unknown class is requested – add graceful handling).
Expand homebrew support if testers tried something and got an “I don’t know that” – for example, if a tester asked about a subclass we forgot to add, go ahead and add it.
Update documentation with any new decisions (if we decided to cut a feature or add something last-minute, record it).
Go-Live Preparation: Before the real session:
Performance Check: Rebuild the FAISS index if new content was added (run build_index.py on all the final PDFs and ensure the index file is updated). The sourcebooks loaded
github.com
 include a huge trove, which is great – but confirm the index size is manageable and that memory usage is within container limits. Possibly allocate more RAM to the container if needed.
Backup: Take a snapshot of the entire setup (export workflows JSON, backup volumes, etc.).
Logging: If wanting to record the session, have a plan – maybe a workflow that appends all CC’s messages to a log file or Google Doc. This can be done by adding a side-action in the main chat workflow (n8n can do multiple actions in parallel: one to respond, one to log). Alternatively, Discord’s chat log can serve as record (just ensure nothing gets lost).
Deploy on Production environment: If tests were on a local machine, now deploy the stack to the cloud server intended for the game. Set up domain if needed for webhooks, ensure all environment variables are correctly set on the server.
Run a final smoke test on the production environment’s Discord server to ensure everything survived the move (usually it should if environment is same, but checking API keys, etc.).
Session Running: During the actual game sessions, monitor the system:
Keep an eye on resource usage (with GPT-4 calls and vector searches, CPU should be fine, memory mostly for index ~ a few hundred MB, network usage minimal except OpenAI).
If any issue arises mid-session (bot not responding, etc.), have a fallback – e.g., be ready to restart a container. Docker-compose makes this quick; n8n will resume and catch up on any missed webhooks (or the slash command can be reissued).
Use the maintainability features we set: if the DM wants to tweak CC’s behavior live (say CC should become more cryptic), we can edit the system prompt in n8n on the fly and save – the next responses will use the new prompt immediately. This dynamic control is a big win for v2.
Post-Session Review: After each game, review the interaction logs:
Were there questions CC couldn’t handle? (Add those answers or context next time.)
Did any rules answers seem incorrect? (Might indicate missing or wrong data in our knowledge base – update that content or adjust the retrieval query.)
Are players engaging well with the bot, or are they unsure of how to use it? (This may suggest adding more guidance or simplifying certain interactions.)
Iterate on the workflows or service as needed for subsequent sessions.