---
layout: post
title: "How we gave one AI agent 53 tools without making it dumber"
description: "A story about attention, and the three architectures it took to figure out that scaling AI agents isn't about picking tools, it's about managing what the model pays attention to."
date: 2026-04-13
permalink: /scaling-ai-agent-53-tools/
tags: [ai, agents, llm, langgraph, architecture, tool-use, engineering]
image: /assets/posts/scaling-ai-agent-53-tools/01-hero.png
---

*A story about attention, and the three architectures it took to figure that out.*

<figure style="margin:3rem 0;">
<svg viewBox="0 0 620 210" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="fig1-title" style="width:100%;height:auto;display:block;">
  <title id="fig1-title">The attention funnel: 53 tools exist in the registry, but only ~14 reach the model per turn.</title>

  <text x="60" y="38" font-family="'DM Sans',sans-serif" font-size="10" fill="#88857C" letter-spacing="1.6">THE REGISTRY · 53 TOOLS</text>
  <g fill="#18160F" transform="translate(60,62)">
    <circle cx="0"   cy="0"  r="3"/><circle cx="18"  cy="0"  r="3"/><circle cx="36"  cy="0"  r="3"/><circle cx="54"  cy="0"  r="3"/><circle cx="72"  cy="0"  r="3"/><circle cx="90"  cy="0"  r="3"/><circle cx="108" cy="0"  r="3"/><circle cx="126" cy="0"  r="3"/><circle cx="144" cy="0"  r="3"/>
    <circle cx="0"   cy="18" r="3"/><circle cx="18"  cy="18" r="3"/><circle cx="36"  cy="18" r="3"/><circle cx="54"  cy="18" r="3"/><circle cx="72"  cy="18" r="3"/><circle cx="90"  cy="18" r="3"/><circle cx="108" cy="18" r="3"/><circle cx="126" cy="18" r="3"/><circle cx="144" cy="18" r="3"/>
    <circle cx="0"   cy="36" r="3"/><circle cx="18"  cy="36" r="3"/><circle cx="36"  cy="36" r="3"/><circle cx="54"  cy="36" r="3"/><circle cx="72"  cy="36" r="3"/><circle cx="90"  cy="36" r="3"/><circle cx="108" cy="36" r="3"/><circle cx="126" cy="36" r="3"/><circle cx="144" cy="36" r="3"/>
    <circle cx="0"   cy="54" r="3"/><circle cx="18"  cy="54" r="3"/><circle cx="36"  cy="54" r="3"/><circle cx="54"  cy="54" r="3"/><circle cx="72"  cy="54" r="3"/><circle cx="90"  cy="54" r="3"/><circle cx="108" cy="54" r="3"/><circle cx="126" cy="54" r="3"/><circle cx="144" cy="54" r="3"/>
    <circle cx="0"   cy="72" r="3"/><circle cx="18"  cy="72" r="3"/><circle cx="36"  cy="72" r="3"/><circle cx="54"  cy="72" r="3"/><circle cx="72"  cy="72" r="3"/><circle cx="90"  cy="72" r="3"/><circle cx="108" cy="72" r="3"/><circle cx="126" cy="72" r="3"/><circle cx="144" cy="72" r="3"/>
    <circle cx="0"   cy="90" r="3"/><circle cx="18"  cy="90" r="3"/><circle cx="36"  cy="90" r="3"/><circle cx="54"  cy="90" r="3"/><circle cx="72"  cy="90" r="3"/><circle cx="90"  cy="90" r="3"/><circle cx="108" cy="90" r="3"/><circle cx="126" cy="90" r="3"/>
  </g>

  <g stroke="#18160F" stroke-width="1.5" fill="none">
    <line x1="240" y1="107" x2="340" y2="107"/>
    <polyline points="332,102 340,107 332,112"/>
  </g>
  <text x="290" y="96"  font-family="'DM Sans',sans-serif" font-size="10" fill="#88857C" text-anchor="middle" letter-spacing="1.4">MIDDLEWARE</text>
  <text x="290" y="126" font-family="'DM Sans',sans-serif" font-size="9"  fill="#88857C" text-anchor="middle" font-style="italic">active_context filter</text>

  <text x="380" y="64" font-family="'DM Sans',sans-serif" font-size="10" fill="#88857C" letter-spacing="1.6">MODEL ATTENTION · 14</text>
  <g fill="#B8882A" transform="translate(380,86)">
    <circle cx="0" cy="0"  r="4"/><circle cx="22" cy="0"  r="4"/><circle cx="44" cy="0"  r="4"/><circle cx="66" cy="0"  r="4"/><circle cx="88" cy="0"  r="4"/>
    <circle cx="0" cy="22" r="4"/><circle cx="22" cy="22" r="4"/><circle cx="44" cy="22" r="4"/><circle cx="66" cy="22" r="4"/><circle cx="88" cy="22" r="4"/>
    <circle cx="0" cy="44" r="4"/><circle cx="22" cy="44" r="4"/><circle cx="44" cy="44" r="4"/><circle cx="66" cy="44" r="4"/>
  </g>
</svg>
<figcaption style="text-align:center;font-size:0.78rem;color:#88857C;font-style:italic;margin-top:0.8rem;">Fig. 1: the same agent holds 53 tools in its registry, but on any given turn the middleware shows it only the handful that matter.</figcaption>
</figure>

<nav class="toc" markdown="1">
<p class="toc-title">Contents</p>
<ol>
  <li><a href="#setup">The setup</a></li>
  <li><a href="#react">Attempt one · a plain ReAct agent</a></li>
  <li><a href="#graph">Attempt two · the distributed state graph</a></li>
  <li><a href="#attention">Attempt three · stop being clever, start managing attention</a></li>
  <li><a href="#scoping">Part one · tool scoping</a></li>
  <li><a href="#prompt">Part two · the three-layer prompt</a></li>
  <li><a href="#nav">The LLM drives its own navigation</a></li>
  <li><a href="#takeaway">What I actually want you to take away</a></li>
  <li><a href="#sources">Sources</a></li>
</ol>
</nav>

---

## The setup {#setup}

Early 2025. I was building a chat assistant for a hospitality company I'll call **Harbor**. Airport lounges, a loyalty card program, gift cards, a B2B portal for travel agencies. Four products, four kinds of customer, one chat window.

The brief: *"One AI. Handles everything. Like talking to a person who actually works here."*

"Everything" meant 53 distinct backend operations across five completely different user journeys. If you've ever tried to wire that many tools into a 2025-era LLM, you already know where this is going.

---

## Attempt one · a plain ReAct agent {#react}

We started the obvious way. One agent, all 53 tools, good prompt, let the model reason.

On a real workload, it collapsed. You have to remember what early-2025 models were like. Impressive in a short loop, two or three tool calls and done, but push them into longer chains with 50+ tools and they fell apart. They'd hallucinate tool names. They'd forget the user was a distributor and reach for customer-facing tools. They'd call the same thing twice with slightly different arguments.

None of this was the model being dumb. It was the model being asked to do something the reasoning budget of that generation couldn't sustain. So we did what everyone else was doing: we retreated into a graph.

---

## Attempt two · the distributed state graph {#graph}

The industry consensus at the time was that if ReAct couldn't handle your complexity, you broke the problem into a graph of smaller agents. Each node a specialist with three or four tools. A router up top. State flowing between them.

For a while it worked better. Then real conversations hit it.

A user in booking mode would ask about their loyalty points. The router would hand them to the membership specialist, which had no idea a half-finished booking was sitting in state. Control would come back and the booking flow would be in a subtly broken state. We patched. We added edges. We added a supervisor layer.

Six weeks in, the graph looked like a subway map drawn by someone having a bad week. The uncomfortable truth took us too long to admit: **the graph wasn't a better architecture. It was scaffolding around a weak foundation.** We hadn't solved the reasoning problem; we'd buried it under so many layers that when it failed, we couldn't find it. We called it a night and decided to tear it down.

<span class="recap">Two architectures, two different kinds of complexity, both collapsing under the same underlying problem.</span>

<figure style="margin:3rem 0;">
<svg viewBox="0 0 620 260" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="fig2-title" style="width:100%;height:auto;display:block;">
  <title id="fig2-title">Three architectures we tried, from chaotic ReAct to tangled state graph to scoped attention.</title>

  <line x1="206.67" y1="40" x2="206.67" y2="200" stroke="#88857C" stroke-width="0.5" stroke-dasharray="2 3"/>
  <line x1="413.33" y1="40" x2="413.33" y2="200" stroke="#88857C" stroke-width="0.5" stroke-dasharray="2 3"/>

  <g transform="translate(103,110)">
    <g stroke="#18160F" stroke-width="0.8" fill="none" opacity="0.75">
      <line x1="16" y1="0"    x2="72" y2="0"/>
      <line x1="14.8" y1="6.2" x2="66.6" y2="27.7"/>
      <line x1="11.3" y1="11.3" x2="50.9" y2="50.9"/>
      <line x1="6.2"  y1="14.8" x2="27.7" y2="66.6"/>
      <line x1="0"    y1="16"  x2="0"    y2="72"/>
      <line x1="-6.2" y1="14.8" x2="-27.7" y2="66.6"/>
      <line x1="-11.3" y1="11.3" x2="-50.9" y2="50.9"/>
      <line x1="-14.8" y1="6.2" x2="-66.6" y2="27.7"/>
      <line x1="-16" y1="0"    x2="-72" y2="0"/>
      <line x1="-14.8" y1="-6.2" x2="-66.6" y2="-27.7"/>
      <line x1="-11.3" y1="-11.3" x2="-50.9" y2="-50.9"/>
      <line x1="-6.2" y1="-14.8" x2="-27.7" y2="-66.6"/>
      <line x1="0"    y1="-16" x2="0"    y2="-72"/>
      <line x1="6.2"  y1="-14.8" x2="27.7" y2="-66.6"/>
      <line x1="11.3" y1="-11.3" x2="50.9" y2="-50.9"/>
      <line x1="14.8" y1="-6.2" x2="66.6" y2="-27.7"/>
    </g>
    <g fill="#18160F">
      <circle cx="72" cy="0"    r="2"/><circle cx="66.6" cy="27.7" r="2"/><circle cx="50.9" cy="50.9" r="2"/><circle cx="27.7" cy="66.6" r="2"/>
      <circle cx="0"  cy="72"   r="2"/><circle cx="-27.7" cy="66.6" r="2"/><circle cx="-50.9" cy="50.9" r="2"/><circle cx="-66.6" cy="27.7" r="2"/>
      <circle cx="-72" cy="0"   r="2"/><circle cx="-66.6" cy="-27.7" r="2"/><circle cx="-50.9" cy="-50.9" r="2"/><circle cx="-27.7" cy="-66.6" r="2"/>
      <circle cx="0"  cy="-72"  r="2"/><circle cx="27.7" cy="-66.6" r="2"/><circle cx="50.9" cy="-50.9" r="2"/><circle cx="66.6" cy="-27.7" r="2"/>
    </g>
    <circle cx="0" cy="0" r="16" fill="#F7F5F0" stroke="#18160F" stroke-width="1.5"/>
    <text x="0" y="3" font-family="'DM Sans',sans-serif" font-size="8" fill="#18160F" text-anchor="middle">agent</text>
  </g>

  <g transform="translate(310,110)" stroke="#18160F" fill="none">
    <g stroke-width="1">
      <line x1="-60" y1="-40" x2="-10" y2="-15"/>
      <line x1="-10" y1="-15" x2="50"  y2="-45"/>
      <line x1="-10" y1="-15" x2="-30" y2="35"/>
      <line x1="-30" y1="35"  x2="30"  y2="20"/>
      <line x1="30"  y1="20"  x2="50"  y2="-45"/>
      <line x1="-60" y1="-40" x2="-30" y2="35"/>
      <line x1="50"  y1="-45" x2="60"  y2="45"/>
      <line x1="30"  y1="20"  x2="60"  y2="45"/>
      <line x1="-10" y1="-15" x2="60"  y2="45"/>
    </g>
    <g fill="#F7F5F0" stroke-width="1.2">
      <rect x="-72" y="-48" width="24" height="16" rx="2"/>
      <rect x="-22" y="-23" width="24" height="16" rx="2"/>
      <rect x="38"  y="-53" width="24" height="16" rx="2"/>
      <rect x="-42" y="27"  width="24" height="16" rx="2"/>
      <rect x="18"  y="12"  width="24" height="16" rx="2"/>
      <rect x="48"  y="37"  width="24" height="16" rx="2"/>
    </g>
  </g>

  <g transform="translate(516,110)">
    <g fill="#88857C" opacity="0.35">
      <circle cx="72" cy="0"   r="2"/><circle cx="62.4" cy="36" r="2"/><circle cx="36" cy="62.4" r="2"/><circle cx="0" cy="72" r="2"/>
      <circle cx="-36" cy="62.4" r="2"/><circle cx="-62.4" cy="36" r="2"/><circle cx="-72" cy="0" r="2"/><circle cx="-62.4" cy="-36" r="2"/>
      <circle cx="-36" cy="-62.4" r="2"/><circle cx="0" cy="-72" r="2"/><circle cx="36" cy="-62.4" r="2"/><circle cx="62.4" cy="-36" r="2"/>
    </g>
    <g stroke="#B8882A" stroke-width="1.2" fill="none">
      <line x1="15" y1="0"    x2="36" y2="0"/>
      <line x1="10.6" y1="10.6" x2="25.5" y2="25.5"/>
      <line x1="0"  y1="15"   x2="0"  y2="36"/>
      <line x1="-10.6" y1="10.6" x2="-25.5" y2="25.5"/>
      <line x1="-15" y1="0"   x2="-36" y2="0"/>
      <line x1="-10.6" y1="-10.6" x2="-25.5" y2="-25.5"/>
      <line x1="0"  y1="-15"  x2="0"  y2="-36"/>
      <line x1="10.6" y1="-10.6" x2="25.5" y2="-25.5"/>
    </g>
    <g fill="#B8882A">
      <circle cx="36" cy="0"     r="3.5"/><circle cx="25.5" cy="25.5" r="3.5"/><circle cx="0" cy="36" r="3.5"/><circle cx="-25.5" cy="25.5" r="3.5"/>
      <circle cx="-36" cy="0"    r="3.5"/><circle cx="-25.5" cy="-25.5" r="3.5"/><circle cx="0" cy="-36" r="3.5"/><circle cx="25.5" cy="-25.5" r="3.5"/>
    </g>
    <circle cx="0" cy="0" r="15" fill="#F7F5F0" stroke="#18160F" stroke-width="1.5"/>
    <text x="0" y="3" font-family="'DM Sans',sans-serif" font-size="8" fill="#18160F" text-anchor="middle">agent</text>
  </g>

  <text x="103" y="222" font-family="'DM Sans',sans-serif" font-size="10" fill="#18160F" text-anchor="middle" letter-spacing="1.4">ATTEMPT 1</text>
  <text x="103" y="238" font-family="'DM Sans',sans-serif" font-size="9"  fill="#88857C" text-anchor="middle" font-style="italic">ReAct · all 53 visible</text>

  <text x="310" y="222" font-family="'DM Sans',sans-serif" font-size="10" fill="#18160F" text-anchor="middle" letter-spacing="1.4">ATTEMPT 2</text>
  <text x="310" y="238" font-family="'DM Sans',sans-serif" font-size="9"  fill="#88857C" text-anchor="middle" font-style="italic">State graph · tangled specialists</text>

  <text x="516" y="222" font-family="'DM Sans',sans-serif" font-size="10" fill="#18160F" text-anchor="middle" letter-spacing="1.4">ATTEMPT 3</text>
  <text x="516" y="238" font-family="'DM Sans',sans-serif" font-size="9"  fill="#88857C" text-anchor="middle" font-style="italic">Scoped · 8–18 at a time</text>
</svg>
<figcaption style="text-align:center;font-size:0.78rem;color:#88857C;font-style:italic;margin-top:0.8rem;">Fig. 2: three architectures. The first failed from too much signal at once; the second from too many moving parts around it; the third works because the agent sees only what it needs.</figcaption>
</figure>

---

## Attempt three · stop being clever, start managing attention {#attention}

Between starting the project and tearing the graph down, two things had changed.

The first was that the models had gotten quietly better. Not dramatically. Just enough that a ReAct loop which had collapsed six months earlier now held together much longer.

The second was that I'd started reading the emerging research on *why* large toolsets hurt agents, and the answer wasn't "the model is bad at choosing." It was subtler and more useful.

**Tool descriptions compete with the user's message for the model's attention.** Every tool definition you put in the prompt is a block of tokens the model has to process, weigh, and mostly ignore. A 2025 paper on RAG-MCP showed that in typical MCP deployments, *72% of the agent's context window is consumed by tool definitions before any actual work begins*. Almost three-quarters of the model's working memory spent on descriptions it will never touch on this turn. ([Gan & Sun, 2025](https://arxiv.org/abs/2505.03275))

<figure style="margin:3rem 0;">
<svg viewBox="0 0 620 180" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="fig3-title" style="width:100%;height:auto;display:block;">
  <title id="fig3-title">A typical MCP agent spends 72 percent of its context window on tool definitions before any user work begins.</title>
  <defs>
    <pattern id="hatch" patternUnits="userSpaceOnUse" width="6" height="6" patternTransform="rotate(45)">
      <line x1="0" y1="0" x2="0" y2="6" stroke="#18160F" stroke-width="1"/>
    </pattern>
  </defs>

  <text x="60" y="38" font-family="'DM Sans',sans-serif" font-size="10" fill="#88857C" letter-spacing="1.6">AGENT CONTEXT WINDOW</text>

  <rect x="60" y="58" width="500" height="36" fill="none" stroke="#18160F" stroke-width="1.5"/>
  <rect x="60" y="58" width="360" height="36" fill="url(#hatch)"/>
  <rect x="420" y="58" width="140" height="36" fill="#B8882A" opacity="0.92"/>
  <line x1="420" y1="58" x2="420" y2="94" stroke="#18160F" stroke-width="1.5"/>

  <text x="240" y="117" font-family="'DM Sans',sans-serif" font-size="10" fill="#18160F" text-anchor="middle" letter-spacing="0.4">72% · tool definitions</text>
  <text x="490" y="117" font-family="'DM Sans',sans-serif" font-size="10" fill="#18160F" text-anchor="middle" letter-spacing="0.4">28% · actual work</text>

  <text x="310" y="150" font-family="'DM Sans',sans-serif" font-size="10" fill="#88857C" text-anchor="middle" font-style="italic">Almost three-quarters of the model's working memory, spent on descriptions</text>
  <text x="310" y="164" font-family="'DM Sans',sans-serif" font-size="10" fill="#88857C" text-anchor="middle" font-style="italic">it never touches on this turn. (RAG-MCP, 2025)</text>
</svg>
<figcaption style="text-align:center;font-size:0.78rem;color:#88857C;font-style:italic;margin-top:0.8rem;">Fig. 3: if you have to pay attention to a menu before you can read the question, you will get the question wrong.</figcaption>
</figure>

When the RAG-MCP authors replaced the all-tools-at-once approach with retrieval, showing the model only the tools relevant to the current query, tool-selection accuracy went from **13.62% to 43.13%** on their benchmark. More than triple. Same model. Same tools. The only thing that changed was how much of the model's attention was spent on noise.

The broader literature tells the same story. Recent benchmarks show per-tool accuracy as high as 96% in isolation collapsing to under 15% in large-toolset multi-turn settings. As one recent survey put it: *"A well-scoped agent with 8 tools that all apply to its task outperforms a general agent with 80 tools in almost every benchmark that matters."* ([Ziółkowski, 2026](https://gziolo.pl/2026/04/09/research-architecting-tools-for-ai-agents-at-scale/))

So we stopped thinking of our job as "help the model choose between 53 tools" and started thinking of it as "**protect the model's attention so it only ever has to choose between the 6 to 18 that actually matter right now.**"

<span class="recap">The frame shifted from "pick the right tool" to "defend the attention budget." Everything that followed came from that reframing.</span>

---

## The architecture, in one sentence

**One agent. All 53 tools in the registry. But on any given turn, the model only sees the tools relevant to what the user is currently doing, and only the prompt text that applies to the current scope.**

Two moving parts: a tool-scoping layer and a three-layer prompt.

---

## Part one · tool scoping {#scoping}

We split Harbor into five **contexts**: booking (~25 tools), gift cards (6), account (12), distributor (18), membership (16). Three tools are always available on top of any context: `load_context`, `get_knowledge`, and `logout`. Those three are the agent's navigation kit.

Enforcement is a middleware that runs before every model call. Seven lines do the real work:

```python
def _scope_tools(self, request, state):
    active_context = state.get("active_context")
    if active_context and active_context in self.TOOL_SCOPES:
        allowed = self.TOOL_SCOPES[active_context]
        tools = [t for t in (request.tools or []) if t.name in allowed]
        return request.override(tools=tools)
    return request
```

The hidden tools are not in the function-calling schema at all. There's nothing to get confused by, and more importantly, **nothing taking up attention.**

This same pattern is showing up independently across the industry. GitHub's official MCP server ships a `--dynamic-toolsets` mode that starts with exactly three meta-tools (`list_available_toolsets`, `get_toolset_tools`, `enable_toolset`) and lets the LLM activate whole tool groups on demand. ([Ziółkowski, 2026](https://gziolo.pl/2026/04/09/research-architecting-tools-for-ai-agents-at-scale/)) Different domain, different terminology, exactly the same idea: start narrow, let the model widen itself when it needs to.

---

## Part two · the three-layer prompt {#prompt}

Tool scoping alone isn't enough. The model also needs to know *what it could do* if it switched, or it'll never think to switch. So we split the system prompt into three layers:

<figure style="margin:3rem 0;">
<svg viewBox="0 0 620 340" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="fig4-title" style="width:100%;height:auto;display:block;">
  <title id="fig4-title">The three-layer prompt: a thin capability index and a thicker core are always loaded; one of five context modules is hot-swapped per turn.</title>

  <text x="60" y="36" font-family="'DM Sans',sans-serif" font-size="10" fill="#88857C" letter-spacing="1.6">EVERY MODEL CALL SEES ONE STACK</text>

  <rect x="120" y="60"  width="280" height="42" fill="none" stroke="#18160F" stroke-width="1.5" rx="2"/>
  <text x="260" y="80"  font-family="'DM Sans',sans-serif" font-size="11" fill="#18160F" text-anchor="middle">Capability index</text>
  <text x="260" y="94"  font-family="'DM Sans',sans-serif" font-size="9"  fill="#88857C" text-anchor="middle" font-style="italic">5 lines · what the agent could do, across every context</text>

  <text x="106" y="86"  font-family="'DM Sans',sans-serif" font-size="9" fill="#88857C" text-anchor="end" letter-spacing="1">ALWAYS</text>

  <rect x="120" y="114" width="280" height="72" fill="none" stroke="#18160F" stroke-width="1.5" rx="2"/>
  <text x="260" y="140" font-family="'DM Sans',sans-serif" font-size="11" fill="#18160F" text-anchor="middle">Always-on core</text>
  <text x="260" y="156" font-family="'DM Sans',sans-serif" font-size="9"  fill="#88857C" text-anchor="middle" font-style="italic">Reasoning framework, voice, boundaries,</text>
  <text x="260" y="168" font-family="'DM Sans',sans-serif" font-size="9"  fill="#88857C" text-anchor="middle" font-style="italic">security, who the agent is</text>

  <text x="106" y="154" font-family="'DM Sans',sans-serif" font-size="9" fill="#88857C" text-anchor="end" letter-spacing="1">ALWAYS</text>

  <rect x="120" y="198" width="280" height="96" fill="none" stroke="#B8882A" stroke-width="2" rx="2"/>
  <text x="260" y="228" font-family="'DM Sans',sans-serif" font-size="11" fill="#18160F" text-anchor="middle">Switchable module</text>
  <text x="260" y="244" font-family="'DM Sans',sans-serif" font-size="9"  fill="#88857C" text-anchor="middle" font-style="italic">Detailed playbook for the current context</text>
  <text x="260" y="256" font-family="'DM Sans',sans-serif" font-size="9"  fill="#88857C" text-anchor="middle" font-style="italic">only · hot-swapped per turn</text>

  <text x="106" y="244" font-family="'DM Sans',sans-serif" font-size="9" fill="#88857C" text-anchor="end" letter-spacing="1">ONE</text>
  <text x="106" y="256" font-family="'DM Sans',sans-serif" font-size="9" fill="#88857C" text-anchor="end" letter-spacing="1">OF FIVE</text>

  <g font-family="'DM Sans',sans-serif" font-size="9" fill="#88857C">
    <line x1="400" y1="214" x2="440" y2="214" stroke="#B8882A" stroke-width="1.2"/>
    <rect x="440" y="206" width="90" height="16" fill="#B8882A" opacity="0.15" stroke="#B8882A" stroke-width="1"/>
    <text x="485" y="217" text-anchor="middle" fill="#18160F">booking</text>

    <line x1="400" y1="234" x2="440" y2="234" stroke="#88857C" stroke-width="0.8" stroke-dasharray="2 2"/>
    <rect x="440" y="226" width="90" height="16" fill="none" stroke="#88857C" stroke-width="0.8"/>
    <text x="485" y="237" text-anchor="middle">gift cards</text>

    <line x1="400" y1="254" x2="440" y2="254" stroke="#88857C" stroke-width="0.8" stroke-dasharray="2 2"/>
    <rect x="440" y="246" width="90" height="16" fill="none" stroke="#88857C" stroke-width="0.8"/>
    <text x="485" y="257" text-anchor="middle">account</text>

    <line x1="400" y1="274" x2="440" y2="274" stroke="#88857C" stroke-width="0.8" stroke-dasharray="2 2"/>
    <rect x="440" y="266" width="90" height="16" fill="none" stroke="#88857C" stroke-width="0.8"/>
    <text x="485" y="277" text-anchor="middle">distributor</text>

    <line x1="400" y1="294" x2="440" y2="294" stroke="#88857C" stroke-width="0.8" stroke-dasharray="2 2"/>
    <rect x="440" y="286" width="90" height="16" fill="none" stroke="#88857C" stroke-width="0.8"/>
    <text x="485" y="297" text-anchor="middle">membership</text>
  </g>

  <text x="310" y="322" font-family="'DM Sans',sans-serif" font-size="9" fill="#88857C" text-anchor="middle" font-style="italic">load_context() swaps the bottom layer · the top two never change</text>
</svg>
<figcaption style="text-align:center;font-size:0.78rem;color:#88857C;font-style:italic;margin-top:0.8rem;">Fig. 4: two always-loaded layers establish identity and possibility; one swappable layer carries the current playbook.</figcaption>
</figure>

1. **A concise capability index**, always loaded. A short list of everything the agent can do across every context: "You can book lounges, manage existing bookings, buy or redeem gift cards, manage partner accounts, top up loyalty cards." Five lines. Tells the model the universe of possibilities without drowning it in detail.

2. **The always-on core**, always loaded. Core behavior: how to think, how to talk, what it can't do, security boundaries. The same for every context. This is who the agent *is*, regardless of what it's currently doing.

3. **The switchable module**, hot-swapped per context. The detailed playbook for the current scope. Booking gets flight codes and passenger types. Distributor gets commission periods and billing extracts. Neither has to know about the other.

Every model call gets *one* overview + *one* set of behavior rules + *one* detailed module. Not five modules fighting for attention. One.

The index in layer 1 matters more than it looks. Without it, an agent stuck in the gift-card scope has no idea it could also help with loyalty cards, so it never calls `load_context` when it should. The index is a tiny attention cost that unlocks the entire navigation system.

---

## The non-obvious bit · the LLM drives its own navigation {#nav}

`load_context` isn't a routing decision our code makes. It's a tool the model calls. When a user in booking mode says *"actually, can I buy a gift card?"*, the model decides to call `load_context("gift_cards")`. State updates. On the next turn, the middleware swaps the prompt module and filters the tools, and the model finds itself in a new room with new capabilities.

From the user's side: nothing happened. From our side: we wrote zero routing logic. The model navigates itself, because we gave it the ability and told it when to use it.

This is the opposite of the state graph. In the graph, *we* decided what transitions were allowed. Here, we decide what tools and instructions live in each room, and the model walks between rooms on its own. Adding a new context is a dict entry, a prompt module, and a list of tools. About an hour of work, versus two weeks of node surgery in the old system.

<span class="recap">The router isn't gone. It moved inside the model.</span>

---

## What I actually want you to take away {#takeaway}

Every architecture in this post was a workaround for a limitation in the process of disappearing.

The graph existed because ReAct couldn't handle 53 tools in early 2025. The scoping architecture exists because even today, attention degrades when the menu gets too long. Both are compromises. Both are scaffolding around something the model, *at this moment*, can't quite do.

Here's the thing nobody writing about AI architecture wants to say out loud: **the models are eating our compromises.** The graph layer we spent four months on in mid-2025 is something a 2026-class model no longer needs. The scoping trick I'm writing about right now is something a 2027-class model probably won't need either. Every workaround you ship today has a half-life. You're building trellises for a plant growing faster than you can build.

This isn't a reason to stop building them. You need the trellis now. But it should change how you hold the work. Don't fall in love with your architecture. Don't let a clever middleware layer become part of your identity. The whole point of scaffolding is to be removed when the thing it's holding up can stand on its own.

Build honestly. Solve the problem you actually have with the models you actually have. Keep the scaffolding as light and removable as possible. Every line of architecture is a bet that the limitation it's working around will still be there tomorrow. Most of those bets will lose. That's fine. That's what progress looks like from the inside.

---

> Scaling agents isn't about picking tools. It's about managing attention. Give the model a small menu, a concise sense of what else is out there, and the ability to walk itself between rooms, and it will stay where it belongs. For now.

Hold it lightly. The next model is coming.

---

## Sources {#sources}

- [RAG-MCP: Mitigating Prompt Bloat in LLM Tool Selection via Retrieval-Augmented Generation (Gan & Sun, arXiv 2505.03275)](https://arxiv.org/abs/2505.03275). The 13.62% → 43.13% accuracy jump, and the finding that tool definitions consume ~72% of agent context windows in typical MCP deployments.
- [Research: Architecting Tools for AI Agents at Scale, by Grzegorz Ziółkowski (April 2026)](https://gziolo.pl/2026/04/09/research-architecting-tools-for-ai-agents-at-scale/). The 8-vs-80 tools observation, GitHub's `--dynamic-toolsets` meta-tool pattern, and the STRAP tool-consolidation pattern.
- [MCP Tool Overload: Why More Tools Make Your Agent Worse (Nebula, DEV.to)](https://dev.to/nebulagg/mcp-tool-overload-why-more-tools-make-your-agent-worse-5a49). An accessible walkthrough of the same phenomenon for readers who want a less academic entry point.
