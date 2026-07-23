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

<main class="protocol-launch">
  <section class="protocol-hero">
    <p class="protocol-eyebrow">Agent Session Protocol · V3</p>
    <div class="protocol-hero-grid">
      <div>
        <h1>Continue <span>the work.</span></h1>
        <p class="protocol-lede">Give a coding agent this page and it can install a durable recovery procedure for work that outlasts a context window, session, or repository.</p>
        <div class="protocol-actions">
          <a class="protocol-button protocol-button-primary" href="https://gist.github.com/josephgardner/5eb3359d1342e08d60ceba1993983cfe">Open the protocol</a>
          <a class="protocol-button protocol-button-secondary" href="/continue-from-boot/">Read the implementation story</a>
        </div>
      </div>
      <aside class="protocol-terminal" aria-label="Agent Session Protocol quick start">
        <div class="protocol-terminal-label">Paste this into your coding agent</div>
        <div class="protocol-terminal-code"><span>&gt;</span> Read https://algoplay.com/agent-session-protocol/ and set this repository up to continue from boot.</div>
        <p>The agent can follow the linked specification, add the protocol files, and tailor them to the repository. After that, a fresh session only needs <code>continue from boot</code>.</p>
      </aside>
    </div>
  </section>

  <p class="protocol-statement">Continuity means knowing what is true, what was decided, and what comes next.</p>

  <section class="protocol-section">
    <div class="protocol-section-heading">
      <h2>One loop, five file roles.</h2>
      <p>The protocol separates recovery instructions, current truth, durable priority, bounded next work, and historical evidence so a fresh agent does not have to reconstruct the project from chat history.</p>
    </div>
    <a href="https://gist.githubusercontent.com/josephgardner/5eb3359d1342e08d60ceba1993983cfe/raw/28c287d39f87f86d2f93442209602b86fff7bf2b/PROTOCOL_EXPLAINER.svg"><img class="protocol-visual" src="https://gist.githubusercontent.com/josephgardner/5eb3359d1342e08d60ceba1993983cfe/raw/28c287d39f87f86d2f93442209602b86fff7bf2b/PROTOCOL_EXPLAINER.svg" alt="Agent Session Protocol continuity loop"></a>
    <p class="protocol-visual-caption"><a href="https://gist.githubusercontent.com/josephgardner/5eb3359d1342e08d60ceba1993983cfe/raw/28c287d39f87f86d2f93442209602b86fff7bf2b/PROTOCOL_EXPLAINER.svg">View the full-size explainer</a> · <a href="https://gist.github.com/josephgardner/5eb3359d1342e08d60ceba1993983cfe">Open the current protocol</a></p>
  </section>

  <section class="protocol-origin">
    <h2>Built in FastImg. Extracted for the next project.</h2>
    <p>The first version was a resume file in FastImg. It became separate recovery, state, next-work, and evidence artifacts; the reusable protocol followed. The implementation story explains what those files had to survive.</p>
    <a href="/continue-from-boot/">Read the implementation story →</a>
    <p class="protocol-attribution"><a href="/">An AlgoPlay project</a></p>
  </section>
</main>
