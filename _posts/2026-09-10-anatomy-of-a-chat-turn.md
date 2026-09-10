---
layout: post
title: "Anatomy of a Chat Turn"
date: 2026-09-10
summary: "A couple weeks before the Summit I wrote about why I still build chatbots. This is the part I promised: what it actually looked like under 30,000 people, and everything that broke along the way."
description: "A trace-by-trace tour of the Databricks architecture behind Brickbot, the Data + AI Summit chat assistant, and the real-time fixes it took to run under load."
image: /2026/anatomy-of-a-chat-turn/image.jpg
tags: [databricks, agent]
---

<style>
article .lede{font-size:1.12em;color:#1d1d1f}
article h3{font-size:1.05em;margin-top:1.8em;color:#1d1d1f}
.q-head{display:flex;gap:12px;align-items:baseline;margin-top:2.2em}
.q-num{color:#d32f2f;font-weight:700;font-size:1.5em;line-height:1;font-variant-numeric:tabular-nums}
.q-head h2{margin:0;border:none;padding:0}
figure{margin:2em 0}
figure img{border:1px solid #e3e3e6;border-radius:8px;display:block}
figcaption{font-family:"SF Mono","Fira Code",Menlo,monospace;font-size:.76em;color:#6e6e73;margin-top:10px;line-height:1.5}
figcaption b{color:#d32f2f;font-weight:600}
.callout{background:#f7f7f8;border-left:3px solid #d32f2f;border-radius:0 8px 8px 0;padding:14px 18px;margin:1.8em 0}
.callout p{margin:.2em 0}
</style>
<p class="lede">In <a href="https://www.nickkarpov.com/2026/why-i-still-build-chatbots-in-2026/">the last post</a> I mapped Brickbot onto the Databricks platform — ingestion here, governance there, a Genie agent for the hard questions — and then I made a promise: that I'd come back and write about what each piece looked like <em>in practice</em>, where it broke, what I'd change. That post was the map. The Summit was the territory, and the territory had opinions.</p>

  <p>The best way I've found to show any of this is to stop describing the architecture and just watch a single request go through it. Every reply Brickbot gives is the visible tip of a chain — retrieval, a model, some governed tools, memory. So here's what I'll do: ask it four questions, open the <strong>trace</strong> of each one in MLflow, and follow it <em>backward</em> from the answer down to the raw data. And at every layer, I'll tell you what happened when the doors opened and thirty thousand people showed up at once.</p>

  <figure>
    <img src="/2026/anatomy-of-a-chat-turn/fig01.jpg" alt="Brickbot answering a question in the chat UI">
    <figcaption><b>Fig 1.</b> The whole product is a chat box. After the Summit it opens with “DAIS 2026 wrapped — how can I help you look back?”, and those starter questions are, more or less, the four we're about to trace. Everything interesting is behind them — which is also where things went wrong.</figcaption>
  </figure>

  <div class="q-head"><div class="q-num">1</div><h2>"Give me a session on real-time streaming"</h2></div>

  <p>Start with the easy one. Brickbot searches, hands back a couple of picks, done. Now open the trace of that turn — and before we chase it anywhere, just look at its shape:</p>

  <figure>
    <img src="/2026/anatomy-of-a-chat-turn/fig02.jpg" alt="An MLflow trace span tree showing the turn structure">
    <figcaption><b>Fig 2.</b> One turn in MLflow: <b>memory → prompt.load → model → tool → model</b>. This shape is the whole article.</figcaption>
  </figure>

  <p>Here's the thing worth internalizing before anything else: <strong>every turn has this exact shape.</strong> Memory, prompt, model, one tool, model again. Only the tool in the middle changes from question to question. Once you see it, a chatbot stops being a mysterious box and becomes a very short, very legible pipeline — which, not coincidentally, is why I could fix so much of it mid-conference without redeploying anything. Two of these steps are the same on every single turn, so let me deal with them first, because they're also where the two best stories live.</p>

  <h3>The prompt (and the four days I spent editing it live)</h3>

  <p>The system prompt isn't in the code. It's loaded at request time from the <strong><a href="https://mlflow.org/docs/latest/genai/prompt-registry/">MLflow Prompt Registry</a></strong> — one for anonymous visitors, one for authenticated users, an FAQ variant, each versioned, each with a <code>@production</code> and <code>@local</code> alias. To change Brickbot's behavior you don't ship anything; you cut a new version and move the alias, and the app picks it up on the next turn.</p>

  <figure>
    <img src="/2026/anatomy-of-a-chat-turn/fig03.jpg" alt="Prompt Registry showing version 44 with production and local aliases">
    <figcaption><b>Fig 3.</b> The authenticated prompt's version history — and the timestamps are the story: a stack of new versions across June 15, then four more before 9:10 on the morning of the 16th. Version 36, 9:03 that morning, is us teaching Brickbot about products announced on the keynote stage minutes earlier.</figcaption>
  </figure>

  <p>This was, without exaggeration, the most valuable thing I had all week. The authenticated prompt landed a new version nearly every day of the event, and almost every one started the same way: I'd be scrolling the live traces, catch an answer I hated, and patch the prompt from the API a few minutes later. When people started coaxing it into revealing internal details about how it worked, that was a prompt version. When it kept giving up on questions it actually had the tools to answer, that was a prompt version. The morning of the keynotes, when Databricks announced a pile of new products, we taught Brickbot about them <em>from the audience</em> — another version, live. None of it touched a deploy pipeline. If you take one thing from this post: your prompt is a versioned artifact with a production alias, not a string literal in your repo. That single decision is the difference between "we'll fix it next release" and "fixed, thirty seconds ago."</p>

  <h3>The model (and the Day 1 incident I don't love talking about)</h3>

  <p>The model is Claude Haiku, served through the <strong><a href="https://docs.databricks.com/aws/en/ai-gateway">Unity AI Gateway</a></strong> as a plain OpenAI-compatible endpoint. About an hour into Day 1, that endpoint started throwing <code>429</code>s. Thirty thousand people is, it turns out, more concurrency than our happy path assumed, and suddenly a chunk of turns were failing in front of a live audience.</p>

  <figure>
    <img src="/2026/anatomy-of-a-chat-turn/fig04.jpg" alt="Unity AI Gateway dashboard: queries, tokens, errors, and latency spiking on Day 1">
    <figcaption><b>Fig 4.</b> The Day-1 spike, from the Gateway dashboard. Requests going vertical (top-left), the error count following them straight up (bottom-left) — and the detail that made me wince, top-right: input tokens exploding while the <b>cached-tokens line never left the floor</b>.</figcaption>
  </figure>

  <p>Two fixes, same afternoon. First, the boring-but-critical one: a graceful failure so a rate-limited turn degraded to an apology instead of a spinner of death. Second, the fun one — because the model is <em>just</em> a URL, a token, and a name, I stood up <strong>cross-account failover</strong>: if the primary account 429s, retry the exact same call against a second Databricks account. That's a few hours of work, not a rewrite, precisely because there's nothing bespoke about the model endpoint. And because everything is traced, I could actually <em>see</em> the failover firing and confirm it was working instead of hoping. The other thing that dashboard showed me was that flat green line: we were paying full input price on every turn because prompt caching wasn't on. Turning it on — <code>cache_control</code> over the static part of the system prompt — got the cached line off the floor and cut input cost by about two-thirds, which is a very good thing to fix on the exact afternoon your traffic goes vertical. The lesson generalizes to any agent: put a gateway between you and the model, treat the model as swappable from day one, and watch the caching line — because the day you need any of that is not a day you'll want to be writing a client.</p>

  <p>The <strong>memory</strong> step at the very top — the first thing that happens, before the model even thinks — I'm going to leave sitting there for now. We come back to it in question four, and it's a better story with the setup.</p>

  <h3>Now the tool — and the thread down to the data</h3>

  <p>In this trace the tool is <code>search_sessions</code>, and the one call underneath it is a <code>POST /sql/statements</code>. The "search" is a query against an <strong><a href="https://docs.databricks.com/aws/en/ai-search/ai-search">AI Search</a></strong> index (what Databricks called Vector Search until this year) — hybrid, so it does classic keyword (BM25) and vector similarity at once. That mattered more than it sounds: it meant hybrid search was a drop-in for the keyword search we already trusted, not a rewrite that begged us to relearn what "relevant" means.</p>

  <figure>
    <img src="/2026/anatomy-of-a-chat-turn/fig05.jpg" alt="Create AI Search index dialog with Hybrid selected">
    <figcaption><b>Fig 4.</b> Hybrid index: full-text and vector embeddings together. It's also exposed as an MCP server, so other apps can reuse it without touching our code.</figcaption>
  </figure>

  <p>But an index is built <em>from</em> a table, so the honest question is: which one, and can I trust it? Click <strong>lineage</strong> and the platform just tells you — the index came from a silver table, which came from a raw table, which came from a job. I didn't set any of this up by hand; it's there because every object is governed.</p>

  <figure>
    <img src="/2026/anatomy-of-a-chat-turn/fig06.jpg" alt="Unity Catalog lineage graph from index back to source table">
    <figcaption><b>Fig 5.</b> Lineage, for free: index ← silver ← raw ← job. The unglamorous hero of the whole system.</figcaption>
  </figure>

  <p>Follow that graph one hop further and you land on the job: a <strong><a href="https://docs.databricks.com/aws/en/jobs/">Lakeflow</a></strong> pipeline on a 15-minute schedule, bronze (the API response, stored faithfully) then silver (trimmed to what we actually serve). "Every 15 minutes" sounds like a throwaway detail until you remember conference data is alive — a room gets reassigned, a session gets pulled. In fact one of our stranger constraints lived right here: session rooms were embargoed until midnight the night before the show, so the silver layer simply withheld the room column until the clock passed it. That's the kind of rule that's a nightmare in application code and a one-line filter in a pipeline.</p>

  <figure>
    <img src="/2026/anatomy-of-a-chat-turn/fig07.jpg" alt="The refresh job in Jobs and Pipelines">
    <figcaption><b>Fig 6.</b> The refresh job behind the index — bronze then silver, every 15 minutes, so a stale room never outlives its correction by more than a quarter hour.</figcaption>
  </figure>

  <p>And what does the job call to get the data? Not a hand-rolled API client — <strong><a href="https://www.databricks.com/product/unity-catalog">Unity Catalog functions</a></strong> (<code>get_sessions</code>, <code>get_speakers</code>, <code>get_exhibitors</code>), each a thin, governed wrapper around <code>http_request</code>.</p>

  <figure>
    <img src="/2026/anatomy-of-a-chat-turn/fig08.jpg" alt="The get_sessions UC function wrapping http_request">
    <figcaption><b>Fig 7.</b> <code>get_sessions</code> — a governed SQL function around an HTTP call. A grantable object, not a snippet buried in a repo.</figcaption>
  </figure>

  <p>Those functions all point at one registered <strong>HTTP Connection</strong>, which is where the credential lives — permissioned like any other catalog object, invisible to the code. In previous years the "connection" to our event vendor was an API key on my laptop and a bus factor of one. Now it's an object I can grant to exactly who should have it. (The part that <em>did</em> bite us, for the record: the vendor's API names its fields <code>startDateTime</code> and <code>endDateTime</code>, and for weeks we read <code>startTime</code> and quietly got null. Governed or not, you still have to read the docs.)</p>

  <figure>
    <img src="/2026/anatomy-of-a-chat-turn/fig09.jpg" alt="The RainFocus HTTP connection in Unity Catalog">
    <figcaption><b>Fig 8.</b> The source: one governed connection to the external event platform. Grant it once; nothing leaks into code.</figcaption>
  </figure>

  <div class="callout">
    <p>So one "find me a session" traced the whole way down: <strong>trace → AI Search → lineage → Lakeflow → UC function → HTTP connection → the vendor API</strong>. Every hop a governed object. That's the backbone. The next three questions just swap the tool in the middle — so they'll go faster.</p>
  </div>

  <div class="q-head"><div class="q-num">2</div><h2>"How many distinct companies are speaking, and break the sessions down by day and time"</h2></div>

  <p>Search is great at "find me one." It's useless at "count the distinct companies" or "group these by morning and afternoon." So the model reaches for a different tool — <code>ask_genie</code> — and hands the hard part to a <strong><a href="https://www.databricks.com/product/ai-bi/genie">Genie Agent</a></strong>, a sub-agent that writes and runs SQL over the same tables. This is the one genuinely new capability this year, and it's why Brickbot could suddenly answer a whole class of question it used to just apologize for. And it doesn't hide the hand-off — Brickbot surfaces the sub-agent's entire reasoning right in the chat: what it understood, the data it picked, its plan, and the SQL it ran.</p>

  <figure>
    <img src="/2026/anatomy-of-a-chat-turn/fig10.jpg" alt="Brickbot chat exposing Genie thinking: understood question, data sources, plan, and SQL">
    <figcaption><b>Fig 10.</b> Brickbot surfaces the Genie sub-agent's whole reasoning inline — what it understood, the data it chose, the plan, and the SQL. And that SQL reads a <b>MEASURE</b> from the <b>metrics</b> namespace, not the raw tables.</figcaption>
  </figure>

  <p>Look at the SQL in that panel: it isn't scanning the raw tables, it's reading <code>MEASURE(distinct_companies)</code> straight from <code>brickbot2026.metrics</code>. Those are the agent's <strong>metric views</strong>, and they're the part worth copying. The questions we knew would get asked — counts, distinct companies, sessions by day and time — are defined once as measures and materialized, so the agent reads a number instead of re-deriving a join every time someone asks. It's also where hard-won semantics live: morning is before noon, afternoon after, and — because the model kept confidently inventing one — there is explicitly no <em>evening</em>.</p>

  <figure>
    <img src="/2026/anatomy-of-a-chat-turn/fig11.jpg" alt="Genie agent configuration showing metric-view measures">
    <figcaption><b>Fig 9.</b> The Genie agent's sources: silver tables plus metric views. Measures like <b>distinct_companies</b> are why "how many companies" doesn't turn into a fresh SQL adventure on every request.</figcaption>
  </figure>

  <p>Two things went sideways here and both are instructive. First, time. On Day 1 Genie was reporting session times seven hours off — it was helpfully "converting" our timestamps, which were already Pacific, into UTC. Live fix. Time zones are the tar pit of every scheduling agent; the only thing that saved us was, again, seeing it in a trace within minutes. Second, throughput. Genie has a per-workspace ceiling, and we approached it. We got the ceiling raised, and I added a small rate limiter so that if we ever went over, Brickbot would quietly fall back to plain search instead of queueing behind a slow analytical query — and I taught the prompt to keep our internal capacity numbers to itself, because a helpful bot will absolutely tell a curious attendee exactly how it's provisioned if you let it. The general pattern under all of this: when a sub-problem needs a different kind of reasoning, give it to a specialized sub-agent instead of cramming it into one prompt, and put a semantic layer in front of it so "how many customers" means the same thing every time it's asked.</p>

  <div class="q-head"><div class="q-num">3</div><h2>"What's on my schedule?"</h2></div>

  <p>The first two questions were anonymous — it didn't matter who was asking. This one is different, and the difference is the whole ballgame for anyone building agents that act for real people. The tool is <code>get_my_schedule</code>, another governed UC function, shaped like the search ones but taking an <strong>attendee ID</strong> and calling the vendor's personal-schedule endpoint.</p>

  <figure>
    <img src="/2026/anatomy-of-a-chat-turn/fig12.jpg" alt="The personal-schedule UC function definition">
    <figcaption><b>Fig 10.</b> The personal-schedule function — parameterized and governed. Notice what's not here: any place where the model gets to decide whose schedule it's allowed to read.</figcaption>
  </figure>

  <p>My rule, and I'd die on this hill: <strong>the model never touches authorization.</strong> By the time <code>get_my_schedule</code> runs, we've already resolved who the user is — that happens in the one piece of this whole system that isn't Databricks, a thin app that validates the attendee and injects their identity into the prompt before the model runs a single token. The LLM is handed "you are talking to attendee X" as settled fact. It never decides it, so it can never be talked out of it. If you let a language model reason about who it's allowed to act as, you've built a very charming security hole, and prompt injection will find it. Resolve identity outside the model, pass it in, and keep the model out of the trust decision entirely.</p>

  <p>One nice touch and one ugly surprise. The nice touch: the vendor's schedule comes back thin, so we enrich it in place with a <code>LEFT JOIN</code> to our sessions table and hand the model real titles, rooms, and times instead of bare IDs — the kind of thing that's trivial when your operational call and your analytics live in the same place. The ugly surprise: the vendor's schedule endpoint sometimes lagged, so a freshly-favorited session would briefly return nothing, and Brickbot would cheerfully tell someone their schedule was empty. We added a model-in-the-loop retry for exactly that case. Real systems are held together with small, specific patches like this, and you only find them by watching real traffic.</p>

  <figure>
    <img src="/2026/anatomy-of-a-chat-turn/fig13.jpg" alt="SQL showing the schedule enriched with a left join to sessions">
    <figcaption><b>Fig 11.</b> The enrichment — the vendor's thin schedule left-joined to our sessions table, in plain SQL.</figcaption>
  </figure>

  <div class="q-head"><div class="q-num">4</div><h2>"What do you know about me — and file a feature request for me"</h2></div>

  <p>Ask two things at once and the model just chains two tools in a single turn. Both of these are about memory, in two different senses, and together they close the loop I left open back in question one.</p>

  <p><code>list_my_memories</code> reads the <strong>managed agent memory</strong> — a durable, semantic profile of what you care about. And here's the callback: <em>that is the exact store the trace quietly loaded at the very top of every turn.</em> The memory step in Fig 2 that I told you to hold onto? That's Brickbot reading your profile before it thinks, so it can be useful from your first message instead of your fifth. Now we're just asking it out loud.</p>

  <p><code>submit_feature_request</code> writes — and in the trace you can watch it reach for a Postgres credential and hit <strong><a href="https://www.databricks.com/product/lakebase">Lakebase</a></strong>, the operational store where the app's own state lives: raw <code>chat_turns</code>, the <code>feature_requests</code> people file, a little <code>user_ui_state</code> so a first-time visitor gets different suggestions than a regular. This tool didn't exist when the doors opened. Halfway through Day 1 I noticed people were telling Brickbot what they <em>wished</em> it could do — and it was just being warmly useless about it, nodding along. So we shipped a tool to capture those as real, queryable feature requests, and taught the prompt to offer one when it hit a wall. Some of the best product feedback we got all year is sitting in that table.</p>

  <figure>
    <img src="/2026/anatomy-of-a-chat-turn/fig14.jpg" alt="Lakebase tables: chat_turns, feature_requests, user_ui_state">
    <figcaption><b>Fig 12.</b> Lakebase — the operational Postgres, right next to the analytics. This is the feature_requests table the tool started filling on Day 1: reminders, waitlists, calendar export, a venue map — plus the occasional bit of ops feedback, like the training lunch that ran out early.</figcaption>
  </figure>

  <div class="callout">
    <p>The symmetry I like: <strong>the trace is how <em>you</em> watch the agent; the operational store is how <em>it</em> remembers.</strong> Two ledgers of the same conversation — one for running it, one for debugging it. Any serious agent ends up needing both, plus a semantic profile on top for the "remembers you" feeling.</p>
  </div>

  <h2>One trace becomes sixty thousand</h2>

  <p>I've been opening single traces this whole time. Pull all the way back and the same experiment shows the shape of the entire event — thirty thousand attendees, north of sixty thousand conversations — a quiet trickle for weeks, then the wall of traffic when the show starts.</p>

  <figure>
    <img src="/2026/anatomy-of-a-chat-turn/fig15.jpg" alt="MLflow usage chart showing the conference-day spike and error rate">
    <figcaption><b>Fig 13.</b> Every request, by day. Application errors stayed near zero across the whole event — though plenty of answers were <em>semantically</em> wrong, which is the harder and more interesting number to move.</figcaption>
  </figure>

  <p>I want to be honest about that gap, because it's the one people gloss over. "Errors near zero" means the app almost never threw. It does not mean the answers were right. A confidently wrong session recommendation doesn't register as an error anywhere — it just quietly disappoints someone. The traces are how you go hunting for those, and every single fix in this post — the failover, the time-zone bug, the empty-schedule retry, a dozen prompt versions — started as something I noticed scrolling through them. The observability wasn't a reporting afterthought. It was the control panel I ran the conference from.</p>

  <h2>What I actually took away</h2>

  <p>The first is that <strong>governed and observable is what makes real-time possible.</strong> Everything that let me fix Brickbot while it was live in front of thirty thousand people came down to the same property: every piece was an object I could see and change on its own. A bad answer was a trace. The fix was a new prompt version and a moved alias. A rate-limit meltdown was a second endpoint and a retry. None of it was a deploy. That's not a nice-to-have when you're operating something live — it's the whole difference between a system you run and a system that runs you.</p>

  <p>The second is how <em>little</em> of this is application code. The hard parts — retrieval, analytics, authorization, memory, tracing — all live in the platform as governed primitives. The thing that stitches them into a chatbot is a thin shim, a few hundred lines. Which means the shim is almost beside the point: the same primitives, wired a little differently, are a data-quality agent, or an ops copilot, or a nightly enrichment job. The chatbot is just the version of it I had to keep alive in public for four days. If you're building anything agentic, the interesting question isn't "how do I build the app" — it's "which of these primitives do I get to not build."</p>

  <p>There's more I haven't gotten to — the memory design deserves its own post, and I still owe some thoughts on measuring semantic quality. But that's the field guide for now. If you want the fast version, the whole thing above is one trace, opened five times.</p>

  <p>If you're building something agentic, the move is to pick one of these primitives and point it at your own problem — stand up a <a href="https://www.databricks.com/product/ai-bi/genie">Genie Agent</a> on your own tables, or point <a href="https://docs.databricks.com/aws/en/ai-search/ai-search">AI Search</a> at your own docs, and see how little is left for you to build.</p>
