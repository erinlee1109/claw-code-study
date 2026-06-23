# Intro 

- We ALL know: 
	- Talking to LLM costs tokens
	- Context management is important
	- Start a new chat when AI's performance degrades

- But exactly why and how? 


- What I will NOT cover
	- What is MCP, Skills
	- How to build an agentic workflow

- What I will cover
	- Agent architecture
	- How harnesses like Cursor and Claude Code run
	- How to use tokens efficiently 



# LLMs are Stateless

- Has no memory of our interaction
- Cannot run code. Cannot see files

- LLMs receive all conversation history in the current session, every time
- Example: 

	**Dialogue**

	```
	User: "What's the time complexity of binary search?"
	Model: "Binary search runs..."
	User: "What's the weather in San Jose today?"
	```

	**Request (HTTP)**

	```json
	POST /v1/messages HTTP/1.1
	Host: api.anthropic.com

	{
	  "model": "claude-sonnet-4-6",
	  "max_tokens": 1024,
	  "messages": [
	    {
	      "role": "user",
	      "content": "What's the time complexity of binary search?"
	    },
	    {
	      "role": "assistant",
	      "content": "Binary search runs in O(log n) time, since it halves the search space on each comparison."
	    },
	    {
	      "role": "user",
	      "content": "What's the weather in San Jose today?"
	    }
	  ]
	}
	```

- Notice the ENTIRE first exchange gets resent. 
- Nothing is "remembered" server-side by the model.



# LLMs need State Management

- Done by harness wrappers like Cursor, Claude Code, ChatGPT website
- What needs to be done? 
	- LLMs receive a context window 
		- Limited by maximum context window. Capped by total number of tokens it can process at once. 

- Several words exist to refer to this
	- Harness
	- Agent runtime
	- Agent loop
	- Client (Anthropic)
	- Orchestrator 
	- Scaffold

- What is harness: Code sitting between model <> real world 
	- Claude Code -> harness 
	- Claude Agent SDK -> build your OWN harness


## Agent Execution Loop

```
while not done:
    1. assemble context (system prompt + history + files + tool results)
    2. call the model
    3. parse output: text response, or a tool-call request?
    4. if tool-call: execute it locally, capture result, go to 1
    5. if text/done: show to user, wait for next input
```



# Tool Use

- LLMs cannot "run" anything
- Instead, outputs structured text (Tool name + JSON args)
- Harness (like Cursor) executes bash, read_file, edit_file, etc.
- Send the result back to LLM 


## Tool Schema 

- Sent to the model every single call, NOT once per session
- What tools exist, what they do, and what arguments they need 
- Example: 

```json
{
  "name": "read_file",
  "description": "Reads the contents of a file at a given path",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": { "type": "string", "description": "Absolute file path" }
    },
    "required": ["path"]
  }
}
```

- Recommended practice: Turn off MCPs you don't need 


## Tool Use Request (model's output)

- Model decides it needs to read a file
- Model requests tool use "read_file"
- `tool_use_id` links the result back to this request 

- Example: 

```json
{
  "type": "tool_use",
  "id": "toolu_01ABC",
  "name": "read_file",
  "input": { "path": "/data/notes.txt" }
}
```


## Harness executes 

- Harness: code sitting between model <> real world

- Sees `tool_use` in model's response 
- Opens the file on disk

```javascript
fs.readFile('/data/notes.txt')
```

- Append `tool_result` to context window 

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01ABC",
  "content": "Meeting notes: discuss Q3 roadmap, budget review..."
}
```

- Harness sends the ENTIRE context window to model 
	
	Wait, that sounds so inefficient! Yes, we will discuss caching in a bit 


## Model's Final Response 

- Converts tool result and other findings into natural language
- Sent back to harness
- Harness displays 

	"The file contains meeting notes — looks like it covers the Q3 roadmap and a budget review."


## Is it done yet? 

- Model has no concept of "done"
- Model may need to use a series of tools
- How can harness decide if the work is done?
- Every model response includes the field: `stop_reason` 

```
stop_reason: "tool_use" — model needs more info. LOOP. 
stop_reason: "end_turn" — model has finished 
...
``` 



# Context Building 

## Different Contexts  

- System prompt 
	- Tool definitions (tools, MCPs)
	- Behavioral instructions

- Skills (dynamically loaded)
	- Name + description always sent in HTTP call to model 
	- Model outputs `tool_use` block
	- Harness sends the entire text `tool_result`
	- REMINDER: If your skill is too long, it can take up a lot of context window

- Memory
	- CLAUDE.md
	- .cursor/rules

- File context 
	- Cursor: RAG. Pre-indexing
	- Claude Code: Explore filesystem live (`grep`, `glob`, `read`)
		- `glob`: pattern-match filenames. `**/*test*.py`
		- `grep`: text search across repo
		- `read`: open specific file

- Conversation history
- Tool result


## Assembling Context 

- Serialized into a JSON blob. 
	- Model can only consume text/token 

- Ordering is important 
	- Static first (system prompt, tools, etc)
	- New conversation after 
	- Why? For caching


## More on Cursor's Codebase Indexing 

- Codebase can be large. Not enough context window. 
- RAG Philosophy: Embed once, retrieve only what's relevant, when it's relevant

- RAG index: How it works 
	- Chunking
		- Code is split into chunks (functions, classes), not whole files. 

	- Embedding
		- Each chunk passes through an embedding model
		- Places semantically similar code near in vector space

	- Vector index
		- Embeddings stored in vector DB for fast search

	- Incremental re-indexing
		- After editing files, update only the changed chunks 


- RAG vs. Live Exploration
	- RAG: fast, cheap. stale until re-indexed. 
	- Live exploration: accurate, up-to-date. expensive. 
	- Some use both



# Safety

## Permissions
- Harness is the gatekeeper 
- Enforces permission gate before all `tool_use` calls
	- Allow 
	- Ask user
	- Block


- Hooks: Code that runs on trigger to automatically enforce constraints 
	- Common hooks
		- `PreToolUse`: block call, modify input
		- `PostToolUse`: inspect result 

	- Example, "never touch `.env`"
		- Model can't follow rules 
		- Model outputs `tool_use` request to read the `.env` file 
		- Harness's `PreToolUse` blocks it 


## Containment

- Philosophy: Never trust the model. Build a system to contain the consequences. 

- Bad input (malformed, not malicious)
	- e.g. `{"encoding": "utf-9"}`
	- schema validation catches it, error returned as `tool_result`

- Bad input (malicious/destructive)
	- e.g. `rm -rf /`
	- contained via sandboxing (Docker/gVisor), resource limits (`ulimit`, `timeout`, no network by default)



# Performance / Cost Engineering

## Cache
- Caching makes answers faster. Compute is more efficient. 
- Avoid re-computing the same prefix

- What is cached? 
	- System prompts
	- Tools
	- Skills
	- Conversation history

- Where is it cached? `KV Cache`
	 - Stored in remote GPU as mathematical representation: Key-Value States/Embeddings
	 - Cache is unique to specific model. Each model caches differently

- What is `TTL` (Time To Live)
	- 5 min ~ 1 hour
	- Every time you send a message, `TTL` resets and keeps the cache
	- After 5 min, clears the cache

- What does it mean to me? 

	- If you return to the conversation after TTL: 
		- Model will re-compute the entire context window
		- Higher token cost
		- Slower answer 


## Attention Dilution "Lost in the Middle"

- Caused by too many tokens

- LLM's recall accuracy
	- beginning (system prompts, rules) -> good
	- middle (conversation history) -> bad
	- end (my latest question) -> good 

- Can lead to hallucination or confusion 



## Sub-agents

- Fresh context window for a sub-task
- Sub-agent returns summary to parent
- Good for parent's context.
- Expensive 



# Recommended Practice

## For Better Outcomes

- Context is scarce
- Avoid a large AGENTS.md / CLAUDE.md
- Too much guidance == No guidance
- Share a table of contents, not encyclopedia. 


## To Save Tokens 
- More and more companies discussing "return on tokens"

1. Manage your context window
	- Turn off MCPs / tools you don't need
	- Don't install SKILLS "just in case"


2. Practice context hygiene
	- Compact
		- At 60% context 
		- After a milestone
	
	- Start fresh
		- Spent 10 turn debugging an error with failed solutions
		- Irrelevant topic


3. Avoid breaks longer than 5min (cache TTL)
	- run /compact before stepping away
	OR
	- Summarize / document before leaving mid-task 
	- Start a new session with summary



# More Resources

- https://openai.com/index/harness-engineering/ 
- https://deepwiki.com/andrew-kramer-inno/claude-code-source-build/1-overview
