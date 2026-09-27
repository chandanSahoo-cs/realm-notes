# Contributing to Realm Notes 📚

Thank you for taking the time to contribute to **Realm Notes**! Whether you are fixing a typo, clarifying a tricky CS concept, improving code examples, or adding new topics, your help is warmly welcomed and greatly appreciated.

---

## Table of Contents

- [Code of Conduct & Spirit](#code-of-conduct--spirit)
- [How Can You Contribute?](#how-can-you-contribute)
- [Content & Attribution Guidelines](#content--attribution-guidelines)
- [Folder Structure & Note Conventions](#folder-structure--note-conventions)
- [Local Development Setup](#local-development-setup)
- [Contribution Workflow](#contribution-workflow)
- [Commit Message Guidelines](#commit-message-guidelines)
- [Review & Merge Expectations](#review--merge-expectations)
- [Reporting Issues or Copyright Concerns](#reporting-issues-or-copyright-concerns)

---

## Code of Conduct & Spirit

Realm Notes is built as an open, accessible knowledge base for computer science fundamentals, web development, algorithms, and system design. Please keep discussions, reviews, and interactions respectful, constructive, and encouraging for learners of all levels.

---

## How Can You Contribute?

Here are some of the most impactful ways you can help:

1. **Clarify Concepts:** Rewrite confusing explanations or add clear analogies, ASCII diagrams, or Mermaid flowcharts.
2. **Enhance Code Snippets:** Provide clean, runnable code examples that illustrate edge cases or practical usage.
3. **Expand Topics:** Add missing topics or deepen concise notes in existing categories (`OS`, `DBMS`, `CN`, `system-design`, `algorithms`, `javascript`, `react`, etc.).

---

## Content & Attribution Guidelines

Because these notes are published publicly and shared with learners:

- **Original Explanations:** Please write explanations in your own words. Do not copy-paste proprietary or copyrighted text from courses, paywalled platforms, or articles.
- **Attribution:** If an example, algorithm formulation, or quote is derived from an open resource, paper, or documentation (e.g., MDN, official RFCs), provide a clear source link or attribution.
- **Concise & Direct:** Aim for concise, bulleted, interview-ready insights accompanied by practical code snippets or mental models rather than overly verbose academic prose.

---

## Folder Structure & Note Conventions

The repository uses VitePress with automated sidebar generation (`vitepress-sidebar`). To keep the sidebar neat and consistent:

### 1. Topic Folders
Place notes inside their relevant topic directories:
```
realm-notes/
├── AI/             # Artificial Intelligence & ML concepts
├── algorithms/     # Data structures & algorithm notes
├── CN/             # Computer Networks
├── DBMS/           # Database Management Systems
├── dev/            # Developer tools, Git, Linux, Docker, etc.
├── javascript/     # JavaScript fundamentals & deep dives
├── next/           # Next.js concepts & architecture
├── OOPS/           # Object-Oriented Programming
├── OS/             # Operating Systems
├── react/          # React patterns, hooks, and internals
├── SQL/            # SQL queries, indexing, and optimization
├── system-design/  # System design concepts & architectural patterns
├── typescript/     # TypeScript types, utility types, and configs
└── public/         # Static assets (images, icons)
```

### 2. File Naming
- Use lowercase **`kebab-case.md`** for filenames (e.g., `event-loop.md`, `b-plus-trees.md`, `cache-invalidation.md`).
- Avoid spaces, special characters, or uppercase letters in filenames.

### 3. Markdown Structure
- **Level-1 Heading (`#`):** Every markdown file **must** begin with a single `# Heading` at the very top. VitePress uses this title to generate the sidebar entry:
  ```markdown
  # Event Loop
  ```
- **Code Fences:** Always specify the syntax highlighting language:
  ````markdown
  ```js
  console.log("Hello, world!");
  ```
  ````
- **Diagrams:** Text-based ASCII diagrams or Mermaid diagrams (` ```mermaid `) are encouraged for visualizing flows and memory layouts.

---

## Local Development Setup

### Prerequisites
- [Node.js](https://nodejs.org/) (v18 or higher recommended)
- [npm](https://www.npmjs.com/)
- [Git](https://git-scm.com/)

### Step-by-Step Setup

1. **Fork and clone the repository:**
   ```bash
   git clone https://github.com/<your-username>/realm-notes.git
   cd realm-notes
   ```

2. **Install dependencies:**
   ```bash
   npm install
   ```

3. **Start the local development server:**
   ```bash
   npm run dev
   ```
   Open `http://localhost:5173` in your browser to view your changes live with Hot Module Replacement (HMR).

4. **Verify the build:**
   Before pushing, ensure that VitePress builds cleanly with no broken markdown links or syntax errors:
   ```bash
   npm run build
   ```

5. **(Optional) Preview production build:**
   ```bash
   npm run preview
   ```

---

## Contribution Workflow

We follow the standard GitHub Fork & Pull Request workflow:

1. **Check Existing Issues:** Before starting work on a major addition, check [existing issues](https://github.com/chandanSahoo-cs/realm-notes/issues) or open a new issue to discuss your proposal.
2. **Create a Feature Branch:**
   ```bash
   git checkout -b docs/add-page-replacement-algorithms
   ```
3. **Make Your Changes:** Edit or add markdown notes, keeping in mind the formatting rules and attribution guidelines.
4. **Test the Build:**
   ```bash
   npm run build
   ```
   *Make sure `npm run build` succeeds without any dead link errors or VitePress parsing errors.*
5. **Commit Your Changes:** Write descriptive commit messages (see conventions below).
6. **Push to Your Fork:**
   ```bash
   git push origin docs/add-page-replacement-algorithms
   ```
7. **Open a Pull Request:**
   - Go to the [realm-notes repository](https://github.com/chandanSahoo-cs/realm-notes) and click **New Pull Request**.
   - Select your branch and fill out the PR description template (explain what was added/changed and why).

---

## Commit Message Guidelines

We encourage clear and structured commit messages following conventional commits:

- `docs(os): add notes on virtual memory and paging`
- `fix(js): correct explanation of closures and lexical scope`
- `refactor(system-design): update CAP theorem diagram`

---

## Review & Merge Expectations

Realm Notes is maintained by a student balancing coursework and an internship. While every contribution is valued:
- PR reviews and merges might take a few days.
- You might receive feedback or suggested tweaks before merging.
- Your patience and understanding are sincerely appreciated!

---

## Reporting Issues or Copyright Concerns

- **Bug or Improvement:** If you find something missing or incorrect but don't have time for a PR, please [open an issue](https://github.com/chandanSahoo-cs/realm-notes/issues).
- **Attribution / Copyright:** If you identify any content, diagram, or text that lacks proper attribution or infringes on copyright, please open an issue or reach out. It will be reviewed and removed or corrected promptly.
