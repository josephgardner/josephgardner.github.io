---
layout: default
title: Agent Session Protocol
permalink: /agent-session-protocol/
wide: true
body_class: protocol-landing-page
description: "A lightweight continuity protocol for long-running, evidence-led agent work."
image: /public/continue-from-boot-social.png
image_type: image/png
image_alt: "Agent Session Protocol continuity loop from BOOT through STATE, QUEUE, NEXT, work, and checkpoint"
image_width: 1200
image_height: 630
---

<nav class="protocol-nav" aria-label="Protocol navigation">
  <a class="protocol-home" href="/">← AlgoPlay</a>
  <a href="https://gist.github.com/josephgardner/5eb3359d1342e08d60ceba1993983cfe">Protocol source ↗</a>
</nav>

<main class="protocol-launch">
  <section class="protocol-hero">
    <p class="protocol-eyebrow">Agent Session Protocol · V3</p>
    <div class="protocol-hero-grid">
      <div>
        <h1>Continue <span>the work.</span></h1>
        <p class="protocol-lede">A lightweight continuity protocol for agent work that outlasts a single context window, session, or repository.</p>
        <div class="protocol-actions">
          <a class="protocol-button protocol-button-primary" href="/continue-from-boot/">Read Continue from Boot</a>
          <a class="protocol-button protocol-button-secondary" href="https://gist.github.com/josephgardner/5eb3359d1342e08d60ceba1993983cfe">Open the protocol</a>
        </div>
      </div>
      <aside class="protocol-terminal" aria-label="The fresh-session prompt">
        <div class="protocol-terminal-label">One prompt, every fresh session</div>
        <div class="protocol-terminal-code"><span>&gt;</span> continue from boot</div>
        <p>Recover the procedure, current truth, durable priority, and one bounded next step.</p>
      </aside>
    </div>
  </section>

  <p class="protocol-statement">Continuity means knowing what is true, what was decided, and what comes next.</p>

  <section class="protocol-section">
    <div class="protocol-section-heading">
      <h2>Fresh context. Durable progress.</h2>
      <p>The protocol keeps small, differently-changing truths in separate files instead of asking a new agent to reconstruct the project from chat history.</p>
    </div>
    <div class="protocol-cards">
      <article class="protocol-card"><span class="protocol-card-index">01 · RECOVER</span><h3>BOOT</h3><p>The repeatable procedure for entering a repository and verifying its handoff.</p></article>
      <article class="protocol-card"><span class="protocol-card-index">02 · ORIENT</span><h3>STATE</h3><p>Machine-readable truth about the active task, decisions, and current checkpoint.</p></article>
      <article class="protocol-card"><span class="protocol-card-index">03 · PRIORITIZE</span><h3>QUEUE</h3><p>The durable task that owns attention across many bounded sessions.</p></article>
      <article class="protocol-card"><span class="protocol-card-index">04 · CONTINUE</span><h3>NEXT</h3><p>The one authorized slice of work, with evidence gaps and scope fences.</p></article>
    </div>
  </section>

  <section class="protocol-section">
    <div class="protocol-section-heading">
      <h2>A queue is not a session.</h2>
      <p>Long-lived priorities and immediate authority are different things. The protocol makes that distinction explicit.</p>
    </div>
    <ol class="protocol-flow">
      <li><span class="protocol-flow-number">01</span><strong>BOOT</strong><small>Recover the procedure.</small></li>
      <li><span class="protocol-flow-number">02</span><strong>STATE</strong><small>Load current truth.</small></li>
      <li><span class="protocol-flow-number">03</span><strong>QUEUE</strong><small>Name the durable task.</small></li>
      <li><span class="protocol-flow-number">04</span><strong>NEXT</strong><small>Bound the next action.</small></li>
      <li><span class="protocol-flow-number">05</span><strong>CHECKPOINT</strong><small>Commit evidence coherently.</small></li>
    </ol>
  </section>

  <section class="protocol-section">
    <a href="https://gist.githubusercontent.com/josephgardner/5eb3359d1342e08d60ceba1993983cfe/raw/28c287d39f87f86d2f93442209602b86fff7bf2b/PROTOCOL_EXPLAINER.svg"><img class="protocol-visual" src="https://gist.githubusercontent.com/josephgardner/5eb3359d1342e08d60ceba1993983cfe/raw/28c287d39f87f86d2f93442209602b86fff7bf2b/PROTOCOL_EXPLAINER.svg" alt="Agent Session Protocol continuity loop"></a>
    <p class="protocol-visual-caption"><a href="https://gist.githubusercontent.com/josephgardner/5eb3359d1342e08d60ceba1993983cfe/raw/28c287d39f87f86d2f93442209602b86fff7bf2b/PROTOCOL_EXPLAINER.svg">View the full-size explainer</a> · <a href="https://gist.github.com/josephgardner/5eb3359d1342e08d60ceba1993983cfe">Open the current protocol</a></p>
  </section>

  <section class="protocol-origin">
    <h2>Built in FastImg. Extracted for the next project.</h2>
    <p>The first version was a resume file in FastImg. It became separate recovery, state, next-work, and evidence artifacts; the reusable protocol followed. The implementation story explains what those files had to survive.</p>
    <a href="/continue-from-boot/">Read the implementation story →</a>
  </section>
</main>
