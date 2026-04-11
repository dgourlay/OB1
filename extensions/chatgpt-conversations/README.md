# ChatGPT Conversation History

## Why This Matters

You had a long conversation with ChatGPT six months ago where you made a critical architecture decision. You remember the conclusion but not the reasoning. Or you spent a week comparing products, weighing tradeoffs across multiple sessions, and now you need that analysis again. Your ChatGPT export has it all — but it's buried in a multi-gigabyte JSON file with thousands of conversations. This extension makes your entire ChatGPT history searchable and browsable through your Open Brain agent, with pyramid summaries that let you scan timelines or dive deep into any conversation.

## What It Does

A semantic search and date-based browsing interface over your imported ChatGPT conversations. The ChatGPT Import recipe extracts pyramid summaries at five detail levels (8-word labels through 128-word full summaries), and this extension exposes two MCP tools to query them: browse by date/type/topic, or search by meaning.

## What You'll Learn

- Semantic vector search with Supabase `pgvector` and an RPC match function
- Pyramid summaries for progressive disclosure (same pattern used in local-rag)
- Date-range and metadata filtering on structured conversation data
- Building MCP tools that query pre-populated data (vs. CRUD extensions)

## Prerequisites

- Working Open Brain setup ([guide](../../docs/01-getting-started.md))
- Supabase CLI installed and linked to your project
- ChatGPT Import recipe run with the `--store-conversations` flag (`recipes/chatgpt-conversation-import/`)
- `pgvector` extension enabled in your Supabase project (enabled by default on new projects)

## Credential Tracker

Copy this block into a text editor and fill it in as you go.

> **Already have your Supabase credentials from the [Setup Guide](../../docs/01-getting-started.md)?** You just need the same Project URL, Secret key, and Project ref.

```text
CHATGPT CONVERSATIONS -- CREDENTIAL TRACKER
--------------------------------------

SUPABASE (from your Open Brain setup)
  Project URL:           ____________
  Secret key:            ____________
  Project ref:           ____________

GENERATED DURING SETUP
  Default User ID:       ____________
  MCP Access Key:        ____________  (same key for all extensions)
  MCP Server URL:        ____________
  MCP Connection URL:    ____________

--------------------------------------
```

---

![Step 1](https://img.shields.io/badge/Step_1-Create_the_Database_Tables-00897B?style=for-the-badge)

![1.1](https://img.shields.io/badge/1.1-Create_the_Table_and_Match_Function-555?style=for-the-badge&labelColor=00897B)

In your Supabase SQL Editor (`https://supabase.com/dashboard/project/YOUR_PROJECT_ID/sql/new`), paste and Run:

<details>
<summary>📋 <strong>SQL: chatgpt_conversations table + match_conversations RPC</strong> (click to expand)</summary>

```sql
-- Conversation history with pyramid summaries
CREATE TABLE chatgpt_conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    chatgpt_id TEXT UNIQUE,
    title TEXT,
    create_time TIMESTAMPTZ,
    update_time TIMESTAMPTZ,
    model_slug TEXT,
    message_count INTEGER,
    conversation_type TEXT,

    -- Pyramid summaries
    summary_8w TEXT,
    summary_16w TEXT,
    summary_32w TEXT,
    summary_64w TEXT,
    summary_128w TEXT,

    -- Searchable metadata
    key_topics TEXT[],
    people_mentioned TEXT[],

    -- Export-native metadata
    voice TEXT,
    gizmo_id TEXT,
    gizmo_type TEXT,
    conversation_origin TEXT,
    conversation_url TEXT,

    -- Processing metadata
    content_hash TEXT,
    import_batch TEXT,
    processed_at TIMESTAMPTZ DEFAULT now(),

    -- Embedding of the 128w summary
    embedding vector(1536),

    user_id UUID REFERENCES auth.users(id) DEFAULT auth.uid()
);

-- Indexes
CREATE INDEX idx_chatgpt_conv_embedding ON chatgpt_conversations
    USING hnsw (embedding vector_cosine_ops);
CREATE INDEX idx_chatgpt_conv_type ON chatgpt_conversations (conversation_type);
CREATE INDEX idx_chatgpt_conv_create_time ON chatgpt_conversations (create_time DESC);
CREATE INDEX idx_chatgpt_conv_topics ON chatgpt_conversations USING GIN (key_topics);

-- Semantic search RPC
CREATE OR REPLACE FUNCTION match_conversations(
    query_embedding vector(1536),
    match_threshold float DEFAULT 0.5,
    match_count int DEFAULT 10,
    filter jsonb DEFAULT '{}'::jsonb
) RETURNS TABLE (
    id uuid,
    title text,
    conversation_type text,
    summary text,
    conversation_url text,
    create_time timestamptz,
    similarity float
) LANGUAGE plpgsql AS $$
BEGIN
    RETURN QUERY
    SELECT
        c.id,
        c.title,
        c.conversation_type,
        c.summary_128w AS summary,
        c.conversation_url,
        c.create_time,
        1 - (c.embedding <=> query_embedding) AS similarity
    FROM chatgpt_conversations c
    WHERE 1 - (c.embedding <=> query_embedding) > match_threshold
    ORDER BY c.embedding <=> query_embedding
    LIMIT match_count;
END;
$$;
```

</details>

![1.2](https://img.shields.io/badge/1.2-Grant_Table_Permissions-555?style=for-the-badge&labelColor=00897B)

New query -- paste and Run:

<details>
<summary>📋 <strong>SQL: Grant service_role access</strong> (click to expand)</summary>

```sql
grant select, insert, update, delete on table public.chatgpt_conversations to service_role;
```

</details>

> [!IMPORTANT]
> This step is required. Supabase does not grant full table permissions to `service_role` by default on new projects. Without this, your MCP server will return "permission denied" errors.

![1.3](https://img.shields.io/badge/1.3-Verify-555?style=for-the-badge&labelColor=00897B)

✅ **Done when:** Table Editor shows `chatgpt_conversations` with all columns, and the `match_conversations` function appears under Database > Functions.

---

![Step 2](https://img.shields.io/badge/Step_2-Deploy_the_MCP_Server-00897B?style=for-the-badge)

Follow the [Deploy an Edge Function](../../primitives/deploy-edge-function/) guide using these values:

| Setting | Value |
|---------|-------|
| Function name | `chatgpt-conversations-mcp` |
| Download path | `extensions/chatgpt-conversations` |

> [!TIP]
> If you already deployed the core Open Brain server, this process is identical — just with a different function name and download path.

✅ **Done when:** `supabase functions list` shows `chatgpt-conversations-mcp` as `ACTIVE`.

---

![Step 3](https://img.shields.io/badge/Step_3-Connect_to_Your_AI-00897B?style=for-the-badge)

Follow the [Remote MCP Connection](../../primitives/remote-mcp/) guide to connect this extension to Claude Desktop, ChatGPT, Claude Code, or any other MCP client.

| Setting | Value |
|---------|-------|
| Connector name | `ChatGPT Conversations` |
| URL | Your **MCP Connection URL** from the credential tracker |

✅ **Done when:** Your AI client shows the extension's tools in its available tools list.

---

![Step 4](https://img.shields.io/badge/Step_4-Test_It-00897B?style=for-the-badge)

Try these prompts in your AI client:

1. **"List my last 10 conversations"** — should return recent conversations with 32w summaries showing titles, dates, and types
2. **"Search my conversations for architecture decisions"** — should return semantically relevant conversations with 64w summaries
3. **"What was I working on in January?"** — should return conversations from that month with enough detail to jog your memory

> [!CAUTION]
> If any prompt returns an error, check the Edge Function logs in your Supabase dashboard (Edge Functions > `chatgpt-conversations-mcp` > Logs) before troubleshooting further.

> [!NOTE]
> No results? This extension queries data populated by the ChatGPT Import recipe. You must run `import-chatgpt.py --store-conversations` before these tools will return anything.

✅ **Done when:** All test prompts return expected results with pyramid summaries at the appropriate detail level.

---

## Available Tools

### `list_conversations`

Browse conversations by date, type, or topic. No embedding required — uses metadata filters and date ranges.

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `after` | string (ISO date) | null | Conversations created after this date |
| `before` | string (ISO date) | null | Conversations created before this date |
| `type` | string | null | Filter by conversation_type (e.g., `product_research`, `technical_architecture`, `business_strategy`) |
| `topic` | string | null | Filter by key_topics (array contains match) |
| `limit` | number | 20 | Max results returned |
| `detail` | string | `"32w"` | Pyramid summary level: `8w`, `16w`, `32w`, `64w`, `128w` |

**Example prompts:**

```
Show my tech conversations from July 2025
```

```
List my last 10 strategy conversations
```

```
What conversations did I have about hiring in Q1?
```

### `search_conversations`

Semantic search over conversation summaries. Embeds your query and matches against the 128w summary embeddings using cosine similarity.

**Note:** This tool requires `OPENROUTER_API_KEY` to generate query embeddings at search time. If you already set this key when deploying the core Open Brain MCP server, it's shared across all edge functions automatically. If not, set it with: `supabase secrets set OPENROUTER_API_KEY=sk-or-v1-...`

`list_conversations` does NOT need this key (it uses SQL filters only).

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `query` | string | (required) | What to search for |
| `limit` | number | 10 | Max results returned |
| `threshold` | number | 0.5 | Minimum similarity score (0 to 1) |
| `detail` | string | `"64w"` | Pyramid summary level to return |

**Example prompts:**

```
Find conversations about RFID
```

```
Search for MVP discussions
```

```
What conversations mention patent filing?
```

---

## Progressive Disclosure

Both tools accept a `detail` parameter that controls how much summary text is returned per conversation. This follows the pyramid summary pattern: short labels for scanning, full summaries for deep reading.

| Level | Words | Use Case | Example |
|-------|-------|----------|---------|
| `8w` | ~8 | Timeline labels, quick scanning | "High chair comparison for hardwood floors" |
| `16w` | ~16 | One-sentence summaries, card titles | "Compared high chairs for hardwood floors, chose Nuna ZAAZ for stability and easy cleaning" |
| `32w` | ~32 | Card previews (default for `list_conversations`) | Key details with enough context to decide if you want to read more |
| `64w` | ~64 | Short paragraphs (default for `search_conversations`) | Decisions, reasoning, and alternatives considered |
| `128w` | ~128 | Full summaries, deep context | Complete summary with people, context, decisions, and tradeoffs. This is what gets embedded for semantic search. |

The defaults are tuned for each tool's primary use case: `list_conversations` defaults to `32w` because you are scanning a timeline, while `search_conversations` defaults to `64w` because you are evaluating specific matches.

---

## Database Schema

The extension uses a single table and one RPC function:

**`chatgpt_conversations`** — One row per imported ChatGPT conversation. Stores the original title, creation/update timestamps, model used, message count, LLM-classified conversation type, five pyramid summary levels, topic and people arrays, export-native metadata (voice, custom GPT, origin), and a 1536-dimensional embedding of the 128w summary.

**`match_conversations()`** — Supabase RPC function for semantic search. Takes a query embedding, similarity threshold, and match count. Returns conversations ranked by cosine similarity with their title, type, summary, URL, and creation time.

Key indexes:

- **HNSW** on the embedding column for fast approximate nearest-neighbor search
- **B-tree** on `create_time DESC` for date-range browsing
- **B-tree** on `conversation_type` for type filtering
- **GIN** on `key_topics` for topic array containment queries

---

## Cross-Extension Integration

### Thoughts to Conversations

Each thought extracted by the ChatGPT Import recipe includes a `chatgpt_conversation_id` in its metadata JSONB. This links back to the `chatgpt_conversations` table. Your agent can follow this link to provide source attribution: "You made this decision during a conversation on January 15th — here is the full context."

**Example workflow:**

1. You search your Open Brain thoughts and find: "Chose event-driven architecture for the notification service"
2. The thought's metadata contains `chatgpt_conversation_id`
3. Your agent uses `search_conversations` or `list_conversations` to pull up the full conversation summary
4. You get the reasoning, alternatives considered, and a link back to the original ChatGPT conversation

### With Professional CRM (Extension 5)

Conversations that mention people (stored in `people_mentioned`) can surface when preparing for meetings. Before a call with a colleague, your agent can search conversations mentioning them to recall past discussions and decisions made together.

---

## Example Queries

These are natural-language prompts you can use with your AI client:

```
What was I working on last month?
```

```
Find my conversations about architecture decisions
```

```
List conversations mentioning PRFAQ
```

```
Search for discussions about hiring
```

```
Show me all product research conversations from 2025
```

```
What voice conversations did I have about the MVP?
```

```
Find conversations where I discussed Redis
```

```
List my strategy conversations from the last 3 months
```

---

## Expected Outcome

After completing this extension, you should be able to:

1. Browse your ChatGPT conversation history by date range, topic, or conversation type
2. Search conversations semantically — find discussions by meaning, not just keywords
3. Control the level of detail returned, from 8-word labels to 128-word full summaries
4. Follow links from extracted thoughts back to their source conversations
5. Open the original ChatGPT conversation via the stored URL

Your agent will be able to answer questions like:
- "What was I researching in October?"
- "Find conversations where I made hiring decisions"
- "Show me my most recent technical architecture discussions"
- "What did I discuss about the product launch strategy?"
- "List all conversations about the RFID project"

---

## Troubleshooting

For common issues (connection errors, 401s, deployment problems), see [Common Troubleshooting](../../primitives/troubleshooting/).

**Extension-specific issues:**

**"No results returned" for any query**
- This extension queries data populated by the ChatGPT Import recipe. Run `import-chatgpt.py --store-conversations` first. Without `--store-conversations`, the recipe writes thoughts but does not populate the `chatgpt_conversations` table.

**"Permission denied for table chatgpt_conversations"**
- You skipped Step 1.2. Go back and run the `GRANT` SQL to give `service_role` access.

**"relation 'chatgpt_conversations' does not exist"**
- The schema SQL from Step 1.1 was not run successfully. Re-run it in the Supabase SQL Editor.

**"function match_conversations does not exist"**
- The RPC function was not created. Re-run the schema SQL — the `CREATE FUNCTION` statement is included at the bottom.

**`search_conversations` returns no results but `list_conversations` works**
- `search_conversations` requires `OPENROUTER_API_KEY` to generate query embeddings at search time. `list_conversations` uses SQL filters and doesn't need it. Set the key: `supabase secrets set OPENROUTER_API_KEY=sk-or-v1-...`. If the core Open Brain MCP server already uses this key, it's shared across all edge functions.

**`list_conversations` returns no results but data exists in Supabase**
- The edge function filters by `user_id`. If conversations were imported without the `USER_ID` env var (rows have `user_id = NULL`), the filter may exclude them. The extension matches both the `DEFAULT_USER_ID` and NULL rows, but verify `DEFAULT_USER_ID` is set correctly in your edge function secrets.

**Conversations show empty summaries**
- The conversations were imported before `--store-conversations` was implemented, or the LLM failed to generate summaries for some conversations. Re-run the import with `--store-conversations` to regenerate.

**Search returns results but they seem irrelevant**
- The similarity `threshold` defaults to 0.5. Try lowering it to 0.3 for broader results, or raising it to 0.7 for stricter matches. If all results score low, the query may not match any conversation topics in your history.

> **Tool hygiene:** This extension adds 2 MCP tools to your AI's context window. As you build more extensions, the total tool count grows — and with it, the context cost and risk of your AI picking the wrong tool. See the [MCP Tool Audit & Optimization Guide](../../docs/05-tool-audit.md) for strategies on auditing, merging, and scoping your tools.
