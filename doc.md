# Resume as Code: Architecture & Deployment Guide

This document provides a high-level overview of how this resume project is structured, built, and hosted. It covers the technical decisions behind using LaTeX, local development workflows, and the fully automated Continuous Integration / Continuous Deployment (CI/CD) pipeline powered by GitHub Actions and GitHub Pages. 

This guide is heavily detailed to serve as a deep dive into the engineering practices applied to a simple document generation process.

---

## 1. Core Technology: LaTeX

This repository contains the source code for my resume in `resume.tex`. But what exactly is LaTeX?

**LaTeX** is a high-quality typesetting system designed for technical and scientific documentation. Instead of using a WYSIWYG (What You See Is What You Get) interface, you write plain text combined with specialized markup commands.

*   **Compilation Targets:** The `resume.tex` file acts as the source code. A compiler parses this code and generates an output document. While the primary target is **PDF**, LaTeX can also compile to **DVI**, **PostScript (PS)**, **HTML**, and even **plain text**.
*   **Why choose LaTeX over Google Docs/MS Word?**
    *   **Version Control:** Plain text is a first-class citizen in Git. You can track exact line-by-line changes, diff histories, and revert mistakes cleanly—something impossible with binary formats like `.docx`.
    *   **Consistent Rendering:** Google Docs can break formatting unpredictably depending on screen size or browser. LaTeX separates content from styling; the code produces the *exact same* mathematically calculated layout every single time it compiles, eliminating the "works on my machine" problem.
    *   **Automation:** Because the document is code, it can be built and deployed by headless servers in a CI/CD pipeline. Good luck getting Google docs to automatically generate and publish a PDF upon a git commit!
    *   **Exquisite Typography:** LaTeX algorithms optimally handle kerning, hyphenation, and text justification far better than standard word processors.

---

## 2. Local Development Environment

Editing LaTeX blindly without visual feedback is counter-productive. To build and preview the resume locally, my setup consists of:

1.  **IDE:** Visual Studio Code (VS Code).
2.  **Plugin:** [LaTeX Workshop Extension](https://marketplace.visualstudio.com/items?itemName=James-Yu.latex-workshop) (`James-Yu/latex-workshop`).
3.  **Compiler:** [TeX Live](https://www.tug.org/texlive/).

**How it works together:**
When you save the `resume.tex` file, the LaTeX Workshop extension detects the file change. It then invokes the TeX compiler installed on your operating system (via TeX Live). Specifically, it often calls modern engines like `xelatex` or `lualatex`, which better support modern system fonts. Once compiled, the extension renders the resulting PDF in a real-time side-by-side preview panel inside VS Code. 

---

## 3. Version Control

Once the document looks good locally, the changes are committed. The beauty of this setup is that we *only* track the `resume.tex` source code. We generally do not need to commit the generated `resume.pdf` to our main branch, minimizing the repository size and avoiding nasty binary merge conflicts. The heavy lifting of PDF creation is offloaded to the cloud.

---

## 4. Hosting with GitHub Pages

### What is GitHub Pages?
GitHub Pages is a static site hosting service that takes HTML, CSS, and JavaScript files (or PDFs!) straight from a repository, optionally runs them through a build process, and publishes them as a live website. 
*   **Branch Hosting:** It is conventionally configured to look at a specific branch (like `gh-pages` or `main`) and host whatever is inside it. 
*   **The Entry Point:** Web servers are designed to expect an entry point. When you navigate to a URL like `https://armanlalani.com`, the server looks for a default document, usually named `index.html`. If it exists, the server renders it.

### What is GitHub Actions?
To transition our LaTeX source code into a hosted PDF automatically, we use **GitHub Actions**. It is a CI/CD platform integrated deeply directly into GitHub. 
*   **Is it a script?** Yes and no. You define instructions in a YAML file. While not a traditional Bash script (`.sh`), the YAML file dictates a workflow of "Steps". These steps can execute arbitrary bash commands *or* run pre-packaged modular code blocks (called "Actions") created by GitHub or the open-source community. It listens for repository events (like a `push` to the `main` branch) and spins up a virtual machine (runner) to execute the workflow.

---

## 5. Demystifying `build_resume.yml`

Our workflow file (`.github/workflows/build_resume.yml`) is the heart of the automation. Let's break down exactly what happens under the hood when a change is pushed.

### Step 1: Checkout Code and Environment Setup
The runner, an isolated `ubuntu-latest` virtual machine, boots up. The first action `actions/checkout@v4` pulls the repository's code into this VM.

### Step 2: Compile LaTeX (`xu-cheng/latex-action@v3`)
This step transforms our `.tex` text file into a `.pdf`.
```yaml
uses: xu-cheng/latex-action@v3
```
**Q: Is this different from my local TeX Live setup?**
A: Conceptually, no. Practically, yes. 
Instead of relying on whatever version of TeX Live might be installed directly on the runner, this step uses an isolated Docker container configured precisely by `xu-cheng`. This container encapsulates an Alpine Linux environment with a robust and standardized version of TeX Live. 
*   **Why use this natively over local?** Consistency. It prevents the "works on my machine but failed in CI" puzzle. A containerized compiler guarantees that the document compiles identically regardless of where the action runs.

**Q: What is the entry point of these GitHub scripts?**
A: When we tell GitHub to use `xu-cheng/latex-action@v3`, GitHub fetches that specific repository at the `v3` tag. It looks for a meta-file called `action.yml` at the root of the repository. This file defines the interface (inputs like `root_file` and `latex_engine`) and specifies how to run it. For Docker-based actions, `action.yml` directs the runner to build an included `Dockerfile` and execute a specific entrypoint script (e.g., `entrypoint.sh`).

### Step 3: Generate and Commit Thumbnail
We use ImageMagick via Docker to snapshot the first page of the newly built PDF and output it as `resume.png`. 
The `stefanzweifel/git-auto-commit-action` then commits *only* this PNG back into our repository. This is useful so that the `README.md` can render an inline preview image of the resume.

### Step 4: Create Public Folder for Hosting
Because we want to deploy just the PDF (and not the raw source code) to GitHub Pages, we organize them into an isolated folder.
```bash
mkdir public
cp resume.pdf public/resume.pdf
cp resume.pdf public/index.pdf
echo '<!DOCTYPE html><html><head><meta http-equiv="refresh" content="0; url=resume.pdf"></head><body>Redirecting to resume...</body></html>' > public/index.html
```
**The HTML Redirect Trick:**
GitHub Pages acts as a web host. When a user visits the root domain (e.g., `resume.armanlalani.com`), the server returns `index.html`. 
However, we don't want a website; we want the user to see the PDF. 
The HTML string we echo into `index.html` employs a client-side redirect via the `<meta>` tag:
*   `http-equiv="refresh"` tells the browser to refresh the page.
*   `content="0; url=resume.pdf"` dictates that the refresh should happen in `0` seconds, pointing to the exact path of our `resume.pdf`.

Essentially, the moment the browser receives the `index.html`, it instantly redirects the user to the actual PDF file. This is a very standard and effective hack to serve static files at root web server domains. We optionally copy the pdf to `index.pdf` as a fallback depending on browser behavior.

### Step 5: Deploy to GitHub Pages (`peaceiris/actions-gh-pages@v3`)
Finally, this action takes everything inside our `./public` folder and commits it to an orphaned, separate branch called `gh-pages`. GitHub is configured to look at this specific branch, take its contents (`resume.pdf`, `index.html`, `index.pdf`), and host them on the global edge network tied to the custom CNAME.

---

### End Result
1. You make a small plaintext change to `resume.tex`.
2. You type `git push`.
3. Within 60 seconds, a headless server pulls the code, generates an immaculate PDF inside a Docker container, creates a PNG thumbnail, orchestrates web-server redirects, pushes the artifacts to a publishing branch, and invalidates global caches so that `resume.armanlalani.com` serves your newly updated resume. 
