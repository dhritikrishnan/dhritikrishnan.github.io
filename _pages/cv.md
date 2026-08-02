---
layout: cv-layout
title: "CV"
seo_title: "CV - Dhriti Krishnan"
description: "CV of Dhriti Krishnan: MS in Computational Data Science at Carnegie Mellon, ML research intern at Blue Origin, researcher with NVIDIA Research and CMU's TEEL Lab, with publications at ACL SRW and AIED."
permalink: /cv/
author_profile: false
published: true
redirect_from:
  - /resume
---

<div class="cv-page">

  <!-- Header -->
  <div class="cv-header">
    <div>
      <h1>CV</h1>
    </div>
    <!-- PDF button hidden until final resume is ready
    <a href="{{ site.baseurl }}/files/cv.pdf" class="cv-pdf-btn" target="_blank">
      <svg viewBox="0 0 24 24"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8l-6-6zm-1 1.5L18.5 9H13V3.5zM6 20V4h5v7h7v9H6z"/><path d="M9 13h6v1.5H9zm0 3h6v1.5H9zm0 3h4v1.5H9z"/></svg>
      PDF
    </a>
    -->

  </div>

  <!-- Education -->
  <div class="cv-card">
    <h2>Education</h2>

    <div class="cv-entry">
      <div class="cv-entry-meta">
        <span class="cv-date-badge">Aug 2025 – Dec 2026</span>
        <div class="cv-location">
          <svg viewBox="0 0 24 24"><path d="M12 2C8.13 2 5 5.13 5 9c0 5.25 7 13 7 13s7-7.75 7-13c0-3.87-3.13-7-7-7zm0 9.5a2.5 2.5 0 1 1 0-5 2.5 2.5 0 0 1 0 5z"/></svg>
          Pittsburgh, PA
        </div>
      </div>
      <div class="cv-entry-body">
        <h3>Master of Science</h3>
        <h4>Carnegie Mellon University</h4>
        <div class="cv-subtitle">Computational Data Science</div>
        <ul>
          <li>Coursework: Deep Learning, Large Language Models, LLM Systems, AI Agents</li>
        </ul>
      </div>
    </div>

  </div>

  <!-- Professional Experience -->
  <div class="cv-card">
    <h2>Professional Experience</h2>

    <div class="cv-entry">
      <div class="cv-entry-meta">
        <span class="cv-date-badge">Jun 2026 – Aug 2026</span>
        <div class="cv-location">
          <svg viewBox="0 0 24 24"><path d="M12 2C8.13 2 5 5.13 5 9c0 5.25 7 13 7 13s7-7.75 7-13c0-3.87-3.13-7-7-7zm0 9.5a2.5 2.5 0 1 1 0-5 2.5 2.5 0 0 1 0 5z"/></svg>
          Renton, WA
        </div>
      </div>
      <div class="cv-entry-body">
        <h3>Machine Learning Research Intern</h3>
        <h4>Blue Origin</h4>
        <ul>
          <li>Built a synthetic data-generation pipeline producing natural-language queries matched to real user distributions and distilled teacher-model trajectories over them for retrieval over a hierarchical knowledge graph.</li>
          <li>Post-trained a <strong>3B-parameter</strong> code LLM into a multi-turn, tool-using retrieval agent via supervised fine-tuning on distilled teacher trajectories, raising held-out task accuracy from 37.5% to <strong>85.1%</strong> (<strong>+47.6 points</strong>).</li>
          <li>Designed a <strong>GRPO</strong> reinforcement-learning pipeline with an LLM-judge reward, vLLM rollouts, and LoRA policy updates; deployed the distilled agent at <strong>96%</strong> lower query-serving cost than the frontier baseline.</li>
        </ul>
      </div>
    </div>

    <div class="cv-entry">
      <div class="cv-entry-meta">
        <span class="cv-date-badge">Jul 2024 – Jul 2025</span>
        <div class="cv-location">
          <svg viewBox="0 0 24 24"><path d="M12 2C8.13 2 5 5.13 5 9c0 5.25 7 13 7 13s7-7.75 7-13c0-3.87-3.13-7-7-7zm0 9.5a2.5 2.5 0 1 1 0-5 2.5 2.5 0 0 1 0 5z"/></svg>
          Bangalore, India
        </div>
      </div>
      <div class="cv-entry-body">
        <h3>Machine Learning Engineer</h3>
        <h4>Securaa</h4>
        <ul>
          <li>Built and deployed an LLM-based threat analysis agent using LLaMA, automating incident triage and reducing manual analysis time by <strong>90%</strong>.</li>
          <li>Migrated threat classification from OpenAI to a self-hosted fine-tuned <strong>LLaMA-70B</strong> model, reducing inference costs while maintaining accuracy.</li>
        </ul>
      </div>
    </div>

  </div>

  <!-- Research Experience -->
  <div class="cv-card">
    <h2>Research Experience</h2>

    <div class="cv-entry">
      <div class="cv-entry-meta">
        <span class="cv-date-badge">Jun 2026 – Present</span>
        <div class="cv-location">
          <svg viewBox="0 0 24 24"><path d="M12 2C8.13 2 5 5.13 5 9c0 5.25 7 13 7 13s7-7.75 7-13c0-3.87-3.13-7-7-7zm0 9.5a2.5 2.5 0 1 1 0-5 2.5 2.5 0 0 1 0 5z"/></svg>
          Pittsburgh, PA
        </div>
      </div>
      <div class="cv-entry-body">
        <h3>NVIDIA Research — CMU</h3>
        <div class="cv-role">Advisors: Zhenzhen Li (NVIDIA); Prof. Guannan Qu (CMU)</div>
        <ul>
          <li>Developing representation-independent metrics that compare agentic code-editing policies against <strong>PPO</strong>, <strong>SAC</strong>, and <strong>MPC</strong> on common axes, measuring policy change as occupancy divergence rather than parameter distance.</li>
          <li>Building an agentic MPC system in which an LLM agent maintains cost functions, constraints, and recovery logic around a fixed optimizer, benchmarked against a differentiable-MPC-plus-RL baseline.</li>
        </ul>
      </div>
    </div>

    <div class="cv-entry">
      <div class="cv-entry-meta">
        <span class="cv-date-badge">Jan 2026 – Present</span>
        <div class="cv-location">
          <svg viewBox="0 0 24 24"><path d="M12 2C8.13 2 5 5.13 5 9c0 5.25 7 13 7 13s7-7.75 7-13c0-3.87-3.13-7-7-7zm0 9.5a2.5 2.5 0 1 1 0-5 2.5 2.5 0 0 1 0 5z"/></svg>
          Pittsburgh, PA
        </div>
      </div>
      <div class="cv-entry-body">
        <h3>TEEL Lab — CMU</h3>
        <div class="cv-role">Advisor: Prof. Jaromir Savelka</div>
        <ul>
          <li>Developed a difficulty prediction model for MCQs using Latent Profile Clustering and LLM-simulated student responses on the EEDI dataset, reducing prediction error by <strong>25%</strong> over 2PL-IRT baselines (AIED 2026).</li>
          <li>Built an async evaluation pipeline for long-context span extraction across <strong>7 LLMs</strong>, showing that ontology-grounded prompts raise F1 from <strong>1.8%</strong> to <strong>57.5%</strong> on complex tasks while degrading simpler pattern matching by <strong>3–5%</strong> (ACL 2026 SRW).</li>
          <li>Designed a four-stage pipeline distilling human preference pairs into natural-language specs for <strong>inference-time alignment</strong> without fine-tuning, outperforming LoRA-based DPO across <strong>6 domains</strong> with <strong>50× less data</strong> and a <strong>75%</strong> pairwise win rate (under review).</li>
        </ul>
      </div>
    </div>

  </div>

  <!-- Projects -->
  <div class="cv-card">
    <h2>Projects</h2>

    <div class="cv-entry">
      <div class="cv-entry-meta">
        <span class="cv-date-badge">Jan 2026 – May 2026</span>
      </div>
      <div class="cv-entry-body">
        <h3>Flash-Attention Style GQA</h3>
        <div class="cv-role">CUDA, PyTorch, JAX</div>
        <ul>
          <li>Implemented IO-aware Grouped Query Attention in CUDA with tiled attention, online softmax, and fused QKᵀ/softmax/value kernels to eliminate quadratic HBM overhead.</li>
          <li>Benchmarked across GPU (<strong>A100</strong>) and TPU (<strong>v4</strong>) via JAX/Pallas, profiling throughput and memory scaling with Nsight Compute.</li>
        </ul>
      </div>
    </div>

  </div>

  <!-- Publications -->
  <div class="cv-card">
    <h2>Select Publications</h2>
    <ul class="cv-pub-list">
      <li>Towards Spec Learning: Inference-Time Alignment from Preference Pairs. <em>Under review.</em></li>
      <li>Semantic Span Annotation: An Exploratory Study of LLM Annotation. <em>SRW ACL 2026.</em></li>
      <li>MCQ Difficulty Prediction via Persona-Driven LLM Framework. <em>AIED 2026.</em></li>
    </ul>
  </div>

  <!-- Technical Skills -->
  <div class="cv-card">
    <h2>Technical Skills</h2>
    <div class="cv-skills-grid">
      <div class="cv-skill-row">
        <span class="cv-skill-label">ML &amp; Post-Training</span>
        <span>PyTorch, CUDA, Hugging Face Transformers, PEFT/LoRA, DeepSpeed, vLLM</span>
      </div>
      <div class="cv-skill-row">
        <span class="cv-skill-label">Languages</span>
        <span>Python, C/C++, SQL</span>
      </div>
      <div class="cv-skill-row">
        <span class="cv-skill-label">Compute &amp; Tooling</span>
        <span>AWS, GCP, Docker, Git, Weights &amp; Biases</span>
      </div>
    </div>
  </div>

</div>
