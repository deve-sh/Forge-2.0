# Autonomous LLM-based Workflow Generator and Executor

### Functional Requirements

- User should be able to type their desired action and the time at which they want it executed
- The prompt is to be converted using pre-defined blocks such as “run code in a sandbox”, “send a notification”, “make an api call”, “search for data on the web”, “eval using llm”, “function/tool: get prev run info”, “function/tool: get user info”, “if/else block” etc
- The user is asked for confirmation on whether it is what they wanted, and if yes, to confirm
- If the action can be achieved, the time at which it is to be executed is defined as a cron job (if repetitive) or as a straightforward one time cloud task entry.
- The action is then run on an isolated instance and its outcome and consistuent node outputs for debugging later.

We’re effectively building: [Forge - No Code API designer](https://app.notion.com/p/Forge-No-Code-API-designer-947c1eb14a34419e889fff6d1077e89e?pvs=21) ← And just adding a few blocks and a designer via natural language for it.

### DB Schema

We need the following for the blocks:

```sql
CREATE TABLE IF NOT EXISTS predefined_blocks (
   type enum (
	   "api_call",
	   "search_web",
	   "execute_code_in_sandbox",
	   "delay",
	   "get_prev_run_info",
	   "get_configured_secret",
	   "set_execution_variable",
	   "eval_using_llm",
	   "verification_step",
	   "get_user_info"
	 )
)
```

Then for each workflow that a user defines:

```sql
CREATE TABLE IF NOT EXISTS workflows (
	id uuid primary key,
	user uuid foreign key,
	team uuid foreign key,
	name string,
	active_version uuid foreign key,
	created_by uuid foreign key,
	updated_by uuid foreign key,
	created_at timestamp,
	updated_at timestamp,
);
```

for versioning of workflows to recover (With a 10 minute merge for any changes to a version, we don’t want to create multiple versions for the same workflow if it’s updated in 10 minutes time):

```sql
CREATE TABLE IF NOT EXISTS workflow_versions (
	id uuid primary key
	workflow uuid foreign key
	created_at timestamp
	updated_at timestamp
	created_by uuid foreign key
	updated_by uuid foreign key
	
	// callable -> url
	// cron -> repeated job
	invocation_type enum ("callable", "cron")
	// only for cron invocation_type
	schedule string | null
	
	// only applicable for callable types
	invocation_auth foreign key
);
```

we need the nodes and edges that define a traversal path (For each version, we store a set of nodes and edges via the react-flow schema):

```sql
CREATE TABLE IF NOT EXISTS nodes (
	id uuid primary key
	workflow uuid foreign key
	workflow_version uuid foreign key
  //	...meta using react-flow data
  // i.e: block type, block edge to, block metadata set by the agent or the user
)
```

Then for each execution, for both an LLM, we store output of a node’s execution which is exposed via the global get_node_exec_output function inside the exec environment:

```sql
CREATE TABLE IF NOT EXISTS node_exec_output (
	id uuid primary key
	...
);
```

Additional - Invocation auth:

```sql
CREATE TABLE IF NOT EXISTS invocation_auth (
    id uuid primary key,
    type enum ("api-key", "basic"),
    
    // single hashed string for api-key,
    // hash of encoded string for basic auth (username:password)
    // tradeoff is user can't see what the auth value is, which is okay
    // they can update it if they want to
    credential string
);
```

### High Level Architecture and flow

- Frontend:
    
    Everything will be painted via React flow’s nodes and edges framework via pre-defined blocks.
    
    - Each workflow and its version will be deployed either as an invokable via url or a cron job defined by the user’s natural language and
    - An LLM can build the workflow depending on the user’s prompt and ask the user for review via react-flow, any changes can be further prompted through. Use a setup like OpenAI’s SDK to plug into a cheaper model for free users and heavier models for paid users (That is a separate discussion, just the common SDK architecture should still work).
    - The user sends the final workflow to the backend for storing as a workflow version and setting up the cloud run function/job.
    - Each workflow needs a start and an end block, every branch needs to connect eventually to the end block.
    - The LLM can intelligently go through the workflow defined and identify any pitfalls, disconnected end nodes, circular infinite loops to block the user from saving and deploying the workflow (We might not even need LLMs for this, simple traversal can help).

- Backend:
    - Note that each workflow isn’t a separate cloud run function, we just have one `execute` function that is invoked with the name or id of the workflow (Via a custom subpath or subdomain that has the name of function and team-id/user-id defined, can be done via Google Cloud Domains or AWS Route 53). To start with, can be a simple `?workflow=<workflow-id>` on an exec function.
    
    For a repeating CRON Job, a simple cloud task or pub/sub mechanism can be setup that just pings this workflow at the defined cron intervals.
    - Each node’s execution has a function inside the execution environment of cloud function/cloud run function. I.e: `await execute_block(node: Node)` which takes `node.type` and `node.metadata` and executes it. Be it a delay timeout function or an if/else condition evaluation.
    - A simple LLM model like Haiku is configured to run for the llm_eval block. Each node of a complex type stores its output in the DB for debugging and retreival later.
    - Logs from each workflow execution are stored for 7 days for retreival, and deleted later.

## For the agents working on this repo

### Tech Stack

Frontend:
- A simple Client-side React app with a server-rendered home page for Agent and SEO (Allowed to use Next.js for it if needed).
- Use AEO principles for agent discovery.
- Auth has to be persisted.

Backend:
- A simple Express.js app is enough for readability and the scale we have to work with initially.

Runner:
- A Go based Dockerizable runner

Yet to define:

External Services to integrate with and build on top of:
- For auth: Supabase / Firebase 
- For Payments: Razorpay / Stripe


### Special Constraints and Instructions for LLMs

- To define: Metering and user-tiering
- To define: Limits of execution