Extending dnd-character for a D&D Tools MCP Server
Goal: Transform the existing dnd-character library into a comprehensive D&D tool suite, exposed via an MCP (Model Context Protocol) server. This will allow an AI agent (e.g. in n8n) to use various D&D utilities through HTTP/SSE endpoints. The new features include: (1) encounter generation & balancing, (2) character sheet import, (3) combat flow management, and (4) utility spell simulations like Identify. Below is a high-level plan for implementing each component and bundling them into an MCP server.
Design Overview
Architecture: Build a microservice (e.g. with FastAPI or Flask) that wraps dnd-character library functionality and new extensions. Each feature becomes an MCP tool that the AI agent can invoke. The service will maintain an SSE endpoint (Server-Sent Events) per the MCP specification for n8n integration
docs.n8n.io
docs.n8n.io
. Through this SSE stream, the server will advertise available tools and handle tool invocation results. The tools themselves can be implemented as standard HTTP REST endpoints or functions internally, but MCP abstracts them to the agent. Integration with n8n: In a Docker Compose setup, run this service as a sidecar alongside n8n. The n8n MCP Client Tool node will connect to the service’s SSE endpoint (with appropriate auth if needed)
docs.n8n.io
. Once connected, the AI agent will see all exposed D&D tools and can call them by name. We must register each tool with a name, description, and expected input/output schema so the agent knows how to use them. Using the official Python MCP SDK
github.com
 can simplify creating the SSE server and tool registration, but a custom implementation with FastAPI and an SSE route (using an event stream) is also possible. Below, we outline each tool’s high-level implementation and how it extends dnd-character:
1. Encounter Generation & Balancing
Feature: Generate random combat encounters including monsters, NPCs, loot (weapons, armor, consumables), and possibly spells. Ensure the encounter is balanced for a given party level or difficulty setting.
Monster Generation: Leverage the library’s SRD monster data. The dnd-character library caches all 5e SRD monsters in a dictionary (SRD_monsters) for quick access
GitHub
. We can implement a function like generate_random_encounter(party_level, difficulty) that:
Determines an XP/CR budget based on party size & level and desired difficulty (easy/medium/hard/deadly). For example, use the DMG guidelines: sum the party’s XP thresholds and choose a total monster XP in the appropriate range
rpg.stackexchange.com
 (e.g. an adjusted XP of ~1000 might be “Hard” for a certain low-level party
rpg.stackexchange.com
).
Select monsters whose Challenge Rating (CR) and quantity fit the budget. This may involve using multipliers for multiple monsters as per DMG encounter building rules (e.g. 2 monsters → 1.5× XP, etc.).
Randomly pick monsters from SRD_monsters by CR range. For example, if aiming for ~500 XP total, maybe one CR 2 monster (450 XP) or a few lower CR monsters summing to ~500. Ensure variety by pulling from different types (undead, beasts, etc.) if desired.
Assemble the chosen monsters into an encounter list. Each monster can be instantiated via Monster(name) from the library
GitHub
, which yields a full stat block object. The tool can return a summary of each monster (name, HP, AC, etc.) and total difficulty rating.
NPC Generation: For friendly or neutral NPCs, use the character generation capabilities. The dnd-character library can create randomized characters easily – if ability scores are not provided, it rolls them automatically (4d6 drop lowest)
GitHub
. We can extend this to an NPC generator that:
Randomly chooses a class (or a simple “commoner” stat block if appropriate) and level. For instance, generate a level-appropriate ally or bystander. Example: npc = Character(name="Random NPC", classs=CLASSES['fighter'], level=2) – the library will roll stats and assign class features.
Assign a random race and alignment if needed (the library’s Character doesn’t explicitly handle race in core, but we can add simple attributes or utilize background for flavor). We might maintain a list of races and pick one to annotate the NPC’s description.
The NPC’s equipment can be auto-filled by the class’s starting gear (the library does this for class constructors, providing an inventory and player_options for gear
GitHub
). We can also randomize a few extra items or gold pieces for flavor.
Return the NPC’s key info (name, class/level, HP, AC, notable equipment).
Loot and Equipment: Use the library’s item database to add random treasure. The dnd-character library includes an equipment.Item class and caches the SRD equipment list
GitHub
. We can:
Pick a few random weapons, armor, or consumables from the SRD list. For example, choose a weapon by random index or by category (melee weapons vs ranged, etc.). The library likely has item indices (e.g., Item('longsword') yields a longsword object).
For magic items (not fully covered by SRD), either use any available open-source dataset or create placeholder magical items. We might generate simple magic effects (e.g. a +1 sword or a potion of healing – which is in the SRD).
Compile the loot list as part of the encounter output.
Spell Generation: Randomly include a scroll or an enemy spellcaster’s prepared spell. The library provides access to spells via dnd_character.spellcasting.SPELLS dict and helper functions
GitHub
GitHub
. We could pull a random spell of a certain level (or relevant to a generated monster/NPC – e.g. if an NPC wizard appears, list one of their spells). This is less critical for encounter balance, but adds flavor.
Balancing Approach: After generating, compute a rough difficulty:
Sum up the monsters’ XP (available in each monster’s data as xp or derived from CR
GitHub
).
Adjust for number of monsters (DMG multiplier). Compare to party’s threshold to label the encounter difficulty
rpg.stackexchange.com
.
The tool’s output can include “Difficulty: Medium (for 4x Level-3 party)” for transparency.
Example: For a party of four 3rd-level characters (threshold ~500 XP for medium), the tool might pick two hobgoblins (100 XP each) and one bugbear (200 XP). Adjusted XP = (200 + 2*100) * 1.5 (for 3 monsters) ≈ 600 XP
rpg.stackexchange.com
, which is a Medium-Hard border. The tool would list these monsters with stats and note the encounter is Medium difficulty.
2. Character Sheet Import & Parsing
Feature: Import D&D character sheets from PDF, HTML, or Markdown and create a Character object in code. This allows the AI agent to “read” a player’s character sheet and manipulate or reference it.
PDF Parsing: Many official and homebrew character sheets are PDFs (often with form fields). We can utilize a PDF reading library such as PyPDF (formerly pypdf/PyPDF2). In fact, the project already has a PDF parser that uses PdfReader from pypdf to extract form field values
GitHub
GitHub
. We would extend that approach:
Identify all relevant fields in the PDF (abilities, saves, skills, HP, AC, class, level, equipment, etc.). The existing code maps field names to character attributes via a FIELD_MAP dictionary
GitHub
GitHub
. This map will need adjustment per PDF form (different sheet formats have different field names).
Read the PDF’s field dictionary (reader.get_fields() in PyPDF) and pull values
GitHub
. Convert those to the types expected by dnd-character. For example, numeric strings to ints for ability scores, class name string to a class constructor (the parser uses a mapping of class names to the library’s class constructors
GitHub
).
Create a Character using the library: e.g. pc = Wizard(name="Gandalf", strength=10, intelligence=16, ...) – filling all stats from the sheet.
Post-process: assign equipment and spells. If the sheet lists items/spells, we look them up in the library’s data (e.g. use Item(item_name) for each listed equipment, and SPELLS[spell_name] for spells to add to spells_known).
Handle missing fields gracefully – the parser can fall back to defaults (the sample code creates a Fighter if it can’t parse class, as a safe default
GitHub
).
HTML/Markdown Parsing: If character sheets are provided in HTML (perhaps from D&D Beyond or another digital tool) or Markdown, the approach is to extract text and parse key stats. We can:
Use an HTML parser (BeautifulSoup, lxml, etc.) to scrape relevant sections by known tags or labels. For example, find elements containing “Strength” or specific class label, etc. Some sites provide structured JSON or XML exports which would be ideal, but if not, parsing text is needed.
For Markdown, assume a consistent format (like headings for each section). We could read the file line by line, detect lines like “Strength: 15”, etc. A simple regex or key lookup approach could map those into attributes.
Once the data is extracted, instantiate a Character object similarly to PDF: feed stats, class, etc. into Character(...) or appropriate class constructor.
Pay attention to multi-class or complex cases (the PDF parser example takes only the first class if “Class” field has “Fighter 1 / Wizard 2”
GitHub
). At high level, it’s fine to initially only support single-class for simplicity.
Output: The tool would output a structured representation of the character. Likely we can return a JSON (or Python dict) serialization of the character, since dnd-character objects can be converted to dicts easily (dict(object) yields a JSON-serializable dict
GitHub
). This includes all stats, making it easy for the AI agent to access any part (HP, spells, etc.).
Example: A user uploads Aragorn.pdf. The parser reads it and creates a Level 5 Ranger character in code. The MCP tool import_character_sheet returns a summary like:
{"name": "Aragorn", "class": "Ranger", "level": 5, "strength": 15, "dexterity": 14, "hp": 38, "ac": 16, "skills": {"Survival": true, "Athletics": true, ...}, "equipment": ["Longsword", "Dagger", "Bow"], ...}
The AI agent can then reason about this data (e.g., to check Aragorn’s HP or inventory during gameplay).
3. Combat Flow Management
Feature: Manage turn-based combat mechanics: initiative order, movement/jump distances, attack rolls, damage application, etc. Essentially, provide a “combat engine” tool to simulate parts of D&D combat so the AI can correctly apply rules.
Initiative Tracking: Implement a function start_combat(encounter_participants) that takes a list of combatants (PCs and monsters, likely using the Character and Monster objects). It will:
Roll initiative for each participant. Initiative = d20 + Dexterity modifier. The Character/Monster objects have a dexterity score; mod = (DEX-10)//2. Use Python’s random.randint(1,20) for the roll.
Sort the participants by initiative descending. Return an ordered list (or turn queue) of who acts first to last. We may include the roll totals for transparency.
Possibly assign an identifier or turn index for tracking. The tool could maintain a simple internal state (like a combat ID with the queue) if we want a follow-up “next_turn” tool. Alternatively, the agent can keep the sorted list and iterate through it on its own, calling other tools for actions.
Movement & Jump Calculations: Provide utility calculations for the agent to use when a player asks about movement constraints. For example:
Move distance: Simply read the speed attribute. Monster data includes a speed dict (e.g. “30 ft” walking speed)
GitHub
. For characters, speed might not be explicitly stored in the current library (unless we infer from race), but we can default to 30 feet for a typical humanoid or allow setting it manually. The tool get_move_options(entity) can return the speeds (walk, fly, climb, etc.) for that entity.
Jump distance: Apply the 5e rules. For instance, long jump = Strength score in feet with a 10-foot running start (half that from standing)
dndbeyond.com
. High jump = 3 + Strength modifier in feet with a running start (or just Strength mod feet if standing). So a character with STR 15 could long jump 15 ft (with run) or 7 ft (no run). We can implement calculate_jump(character) to output those distances. These are straightforward formulas from the rules that the agent can incorporate into answers. (If needed, include the effect of the Jump spell or Boots of Striding if those are in play, but at high level, basic cases suffice.)
Attack and Damage Simulation: Enable the agent to resolve an attack in mechanics:
Use the library’s data for attack bonuses and damage. In the dnd-character Monster objects, each action entry often has an attack_bonus and a damage range (e.g., “Hit: 5 (1d6+2) slashing damage”). For characters, one would use their weapon stats (we might need a method to get a character’s attack bonus: typically proficiency + ability mod, depending on weapon and class proficiency).
Implement a tool function like attack_roll(attacker, defender, weapon_or_action):
Roll d20, add appropriate attack bonus (for monsters: provided in data
GitHub
, for characters: compute from stats).
Compare to defender’s Armor Class (defender.armor_class is provided by the library for characters and monsters
GitHub
).
Determine hit or miss. If hit, roll damage dice. The library doesn’t directly roll dice, but we can parse the damage string (e.g. “1d6+2”) or leverage a dice-rolling utility if available. We can extend the library with a simple dice roller parser.
Subtract damage from defender’s HP (defender.hit_points for monsters
GitHub
, for characters use their hp attribute). Possibly track if someone is down (HP ≤ 0).
Return the outcome: e.g. “Zombie hits Aragorn for 5 slashing damage, Aragorn HP now 33/38” or “Goblin misses Gandalf.” The agent can use this to narrate combat results accurately.
We can use the example from the library’s README as a guide: it shows checking a zombie’s attack vs a Bard’s AC
GitHub
.
Turn Progression: If needed, maintain state between turns:
One approach: a CombatEncounter class to hold the initiative order and current turn index, plus references to all participants (with their evolving HP, etc.). The MCP server could store this in memory (e.g. in a dict keyed by an encounter ID).
Tools could include next_turn(encounter_id) to simply return who’s turn is next (advancing the index and maybe handling round-robin looping), and end_turn(encounter_id) or an automatic wrap when reaching end of list.
However, managing this state might be complex with multiple simultaneous combats or if the agent forgets the ID. An easier stateless approach is to let the agent handle ordering. For instance, the agent calls start_combat to get initiative sorted list, then on each turn it calls appropriate tools (attack_roll, move_character, etc.) providing the relevant entities. The server can compute results but doesn’t track whose turn it is – the agent (with its memory/context) can keep that logic.
Other Mechanics: Many other combat aspects (saving throws, area effects, conditions) could be included, but at high level we focus on movement and basic attacks as requested. We should ensure the tools are modular so more can be added later. For example, a apply_damage(target, amount) tool could be a generic way to adjust HP and report remaining HP, which can be used for spell damage or other effects too.
Example workflow: The agent asks the combat tool to roll initiative for 3 heroes and 2 monsters. The server returns an ordered list: [Goblin (init 18), Alice (init 15), Bob (init 12), Orc (init 11), Charlie (init 5)]. The agent then says “Goblin attacks Alice” and calls the attack tool with Goblin as attacker, Alice as defender. The server responds that the Goblin hit (roll 18+4 vs AC 14) for 6 damage, and Alice’s HP is reduced to 20. The agent uses this to narrate the outcome and decides the next action.
4. Utility Spell Simulation – e.g. "Identify"
Feature: Allow the AI agent to simulate casting utility spells that provide information. The prime example is Identify, which reveals magical item properties. We will create an identify_item(description, image?) tool. It accepts a textual description (and potentially an image of the item) and returns a detailed, “official-sounding” description of the item, as if the spell was cast successfully.
Integrating Search: In the typical use-case described, the agent might first use a general Search tool (another MCP tool or an internet query) to see if the item matches any known D&D items. For instance, if the player says “an empty whisky bottle containing a crackling, glowing blue crystal shard,” the agent might search D&D databases for “blue crystal shard bottle magic item”. We assume the agent can do that (via another tool), but our Identify tool can also assist by having access to a database of items:
Use the 5e SRD data for known items. The SRD includes common magical items like Potions, a few elemental gems, etc. If the item description appears to match something in the SRD (for example, a “Driftglobe” or “Lightning Bottle” if such existed), we fetch that item’s official description. The library’s cached data (via SRD("/api/equipment/...") or a similar endpoint) might contain basic item info. We may need an internal index of item descriptions to search keywords.
If a match is found (either via agent-provided item name or our keyword search), format the output with the item’s name and properties. For example: “Driftglobe – Wondrous item, uncommon. A small sphere of thick glass weighing 1 pound. If you are within 60 feet of it, you can speak its command word and cause it to emanate the Light spell…” (an authoritative description paraphrased or quoted from official text
rpg.stackexchange.com
). Always ensure the info is grounded in real D&D source (to avoid AI hallucination).
If no exact match: the tool should create a plausible new item. It will fabricate a name and magical properties that fit the description, in a balanced way:
Analyze keywords in the description (e.g. “bottle”, “whisky”, “crackling blue shard” suggests maybe a bottled lightning or a trapped elemental).
Decide an item type and rarity. Perhaps it’s a consumable if a bottle, or a wondrous item. For instance, “Bottle of Stormlight – when opened, releases a thunderous blast (once per day)”.
Generate properties: The AI agent itself could suggest something, but since our tool is non-AI code, we can use rule-based creativity or have a pre-written list of interesting effects to mix and match. Alternatively, allow the agent to supply some creative input and the tool just formats it.
Ensure the output sounds official – use the tone and structure of D&D item descriptions (mention rarity, attunement if needed, what the item does). This way the agent’s response to the player feels authentic.
Spell-Specific Logic: Although Identify is the example, consider extending to other utility spells:
Detect Magic: Could list schools of magic present on an item or in an area described.
Locate Object: Given a description, could return a hint of direction/distance if the object exists nearby (this is more situational).
Comprehend Languages: Could translate a given phrase or text from a D&D language.
These can be added similarly as separate tools. The Identify tool pattern (take an unknown detail and reveal information) is a template for others.
Using Official Sources: The prompt mentions “grounded in D&D official sources.” We should use open-source content (SRD) to stay within legal use. That includes all SRD items and spells (found via the 5e API cached in the library
GitHub
 and its JSON). For non-SRD content, we provide a creative but believable description rather than quoting proprietary text. The Identify tool can cite the source internally if needed (for our development/debugging), but the AI agent will likely relay the info in a narrative form to players.
Example: The agent calls identify_item with the input description. The tool searches and finds no exact match in the SRD. It then creates: “Bottle of Lightning (Wondrous Item – Uncommon): This appears to be a mundane whiskey bottle, but a crackling blue crystal shard inside pulses with electrical energy. When you uncork the bottle, lightning arcs outward. Once per short rest, you can remove the shard and speak a command word to cast the Lightning Bolt spell (Save DC 15). The shard then recharges inside the bottle at dawn. The bottle’s glass is magically reinforced to contain the volatile energy.” This output is richly flavored and could convince a player that Identify revealed a unique magical item.
MCP Server Implementation
With the above tools developed (as Python functions/classes extending the dnd-character module), we integrate them into an MCP server:
Server Framework: Use FastAPI (for async support) or Flask to define API routes. FastAPI is convenient for SSE using StreamingResponse for the event stream. We create:
An SSE endpoint (e.g. /mcp/stream) that on client connect will send an initial event containing the list of available tools and their metadata. The MCP Python SDK can handle this handshake, or we manually format a JSON like {"tools": [...tool definitions...]} as the first event.
Endpoints for each tool (if using HTTP POST calls from the client). However, MCP typically sends tool requests through the SSE connection itself (the client posts a request which the server then emits as an SSE event? or vice versa). Simpler: use the MCP SDK which abstracts this. For example, using the SDK you might define each tool as a Python function with a decorator, and the SDK handles routing calls to it via SSE.
Each tool function receives input (parameters from the agent’s call) and returns output (which the server sends back over SSE as a result event).
Tool Definitions: We should give each tool a clear name and description for the agent:
“GenerateEncounter”: Generates a random balanced encounter of monsters (and loot) given party details and desired difficulty.
“ImportCharacterSheet”: Parses a D&D 5e character sheet from a file (PDF/HTML/Markdown) into a Character object for reference.
“StartCombat”: Rolls initiative and returns turn order for given combatants; also provides utilities for movement and attack resolution. (We might split this into sub-tools: RollInitiative, AttackRoll, CalcJump, etc., so the agent can call them separately as needed.)
“IdentifyItem”: Simulates the Identify spell on an unknown item, returning a detailed description or identification.
(Plus any other spells or tools like DetectMagic if implemented similarly.)
Each tool will specify input parameters. For example, GenerateEncounter might accept JSON like {"party_level": 3, "party_size": 4, "difficulty": "Medium"} and optional terrain or theme; IdentifyItem might accept {"description": "text", "image": "url-or-null"}.
Testing with n8n: Once the server is running (e.g. on http://mcp-server:8000), configure n8n’s MCP client node:
Point the SSE Endpoint to http://mcp-server:8000/stream (or appropriate path).
If auth is used, set up Bearer token or None if internal network is fine
docs.n8n.io
.
Select the tools to expose (likely all of them for our use-case).
Then, in the AI agent’s workflow, the agent will have these tools in its toolbox. We can test prompts like “Generate an encounter suitable for a level 5 party” and see if the agent calls the tool correctly, or “(Player provides character sheet) – the agent should call ImportCharacterSheet with the file, etc. Debug any input/output format issues.
Serialization & Response Format: Ensure all tool outputs are serialized to text or JSON that the agent can parse. The MCP spec generally transmits everything as text streams (often JSON strings). For example, our Identify tool should probably output a textual description (which the agent will directly use in its answer), whereas something like ImportCharacterSheet might output a JSON that the agent might summarize. We might decide to have ImportCharacterSheet tool itself return a summary string of the character (for easier consumption), and only provide raw stats if requested. This is a design choice – since the agent is quite capable of reading JSON, either approach can work.
Error Handling: We must handle errors gracefully, sending error messages back that the agent can understand. For instance, if the PDF parser fails or an unsupported format is uploaded, return a message like “Could not parse character sheet – unknown format” so the agent can respond appropriately. Similarly, if a tool is given unreasonable inputs (e.g., encounter generation with negative party level), return a validation error.
Performance Considerations: The dnd-character library on first use will fetch data from the 5e API and cache it
GitHub
. We should initialize that at server startup (perhaps call a few library functions to warm the cache) so that the first agent request doesn’t incur a delay. Also, consider that image processing (if we ever analyze the image for Identify) would require an OCR or image recognition step – likely out of scope, as the agent can describe the image in text to the tool.
In summary, by extending the dnd-character library with these capabilities and exposing them via an MCP-compatible server, we enable an AI agent to conduct rich D&D interactions. The agent will be able to balance encounters, parse character data, run combat scenarios, and magically identify items with authority, all by delegating to these structured tools. This design keeps the heavy rules logic in code (for accuracy) while letting the AI focus on storytelling and decision-making, resulting in a powerful Dungeon Master assistant. Sources:
dnd-character library usage examples
GitHub
GitHub
 (monster attacks, spell access)
D&D 5e rules references for balancing and movement (DMG encounter difficulty
rpg.stackexchange.com
, jump distance
dndbeyond.com
)
Project’s PDF parser for character sheets (utilizing PyPDF)
GitHub
GitHub
n8n MCP integration docs (tool exposure and SSE config)
docs.n8n.io
docs.n8n.io
Citations

MCP Client Tool node documentation | n8n Docs

https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/

MCP Client Tool node documentation | n8n Docs

https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/

GitHub - modelcontextprotocol/servers: Model Context Protocol Servers

https://github.com/modelcontextprotocol/servers
GitHub
monsters.py

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd-character/dnd_character/monsters.py#L7-L14

dnd 5e 2014 - How Exactly Do I Balance Combat Encounters In D&D With The Rules From The DMG? - Role-playing Games Stack Exchange

https://rpg.stackexchange.com/questions/206838/how-exactly-do-i-balance-combat-encounters-in-dd-with-the-rules-from-the-dmg

dnd 5e 2014 - How Exactly Do I Balance Combat Encounters In D&D With The Rules From The DMG? - Role-playing Games Stack Exchange

https://rpg.stackexchange.com/questions/206838/how-exactly-do-i-balance-combat-encounters-in-dd-with-the-rules-from-the-dmg
GitHub
README.md

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd-character/README.md#L34-L41
GitHub
README.md

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd-character/README.md#L130-L138
GitHub
README.md

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd-character/README.md#L58-L67
GitHub
README.md

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd-character/README.md#L62-L70
GitHub
README.md

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd-character/README.md#L80-L88
GitHub
README.md

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd-character/README.md#L101-L109
GitHub
monsters.py

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd-character/dnd_character/monsters.py#L40-L48

dnd 5e 2014 - How Exactly Do I Balance Combat Encounters In D&D With The Rules From The DMG? - Role-playing Games Stack Exchange

https://rpg.stackexchange.com/questions/206838/how-exactly-do-i-balance-combat-encounters-in-dd-with-the-rules-from-the-dmg
GitHub
pdf_character_parser.py

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd_chatbot/pdf_character_parser.py#L1-L9
GitHub
pdf_character_parser.py

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd_chatbot/pdf_character_parser.py#L133-L141
GitHub
pdf_character_parser.py

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd_chatbot/pdf_character_parser.py#L29-L38
GitHub
pdf_character_parser.py

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd_chatbot/pdf_character_parser.py#L62-L70
GitHub
pdf_character_parser.py

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd_chatbot/pdf_character_parser.py#L88-L93
GitHub
pdf_character_parser.py

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd_chatbot/pdf_character_parser.py#L134-L142
GitHub
pdf_character_parser.py

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd_chatbot/pdf_character_parser.py#L142-L149
GitHub
README.md

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd-character/README.md#L140-L148
GitHub
monsters.py

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd-character/dnd_character/monsters.py#L28-L36

Jump Distance - Rules & Game Mechanics - D&D Beyond Forums

https://www.dndbeyond.com/forums/dungeons-dragons-discussion/rules-game-mechanics/107450-jump-distance?srsltid=AfmBOoqwIfU6P2NO-afrjjEVMW3HinNRG5BWAM1X9sd8Dmdw5fVj7uUW
GitHub
monsters.py

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd-character/dnd_character/monsters.py#L24-L32
GitHub
SRD.py

https://github.com/jereimers/CC_one-shot/blob/5d642a9deb3d9929ef6e7f913900c5446aa4f654/dnd-character/dnd_character/SRD.py#L81-L89

MCP Client Tool node documentation | n8n Docs

https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/