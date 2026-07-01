# Automation Workshop. CI/CD

## 1. Local automation

Imagine Git as a production line in a factory. Before a part (your commit) is packaged and sent upstream, you want it to pass a local quality control check. This is where Git Hooks come in: they are simple scripts that run automatically at key lifecycle events.

The most commonly used hook is `pre-commit`. It runs the exact second you press `git commit`. If the script detects an error (e.g., the code does not compile or is poorly formatted), it aborts the commit on the spot, saving you from pushing broken code to GitHub.

### Automatically Verify Compilation

Every Git repository contains a hidden directory named `.git/hooks/` filled with examples. We will create a custom hook from scratch.

First, make sure you are in the local directory clone of this repository (`workshop-automation`). Then, create your own repository on GitHub using `gh`. If you need help with `gh` config, check the [Github Workshop](https://github.com/rosedu/workshop-github/tree/main).

```bash
./gh-create-repo.sh
```

Check your repository on GitHub using a web browser.

Now, let's start writing our own hook.

1. Navigate to the hooks directory inside your local repository:

```bash
cd .git/hooks/
```

2. Create an empty file named exactly `pre-commit` (no `.sh`, `.txt`, or any other extension):

```bash
touch pre-commit
chmod +x pre-commit
```

> [!CAUTION]
> The name of the file should always be `pre-commit`. Any other name will be ignored by Git.


3. Add the following code to the file:

```bash
#!/bin/bash

echo "=== [Hook] Checking if code compiles before committing ==="

# Move into the project directory
cd c-bye

# Attempt to compile the program
gcc bye.c -o bye

# $? represents the exit code of the last command (gcc).
# 0 means success, anything else means an error occurred.
if [ $? -ne 0 ]; then
    echo "❌ [Error] Code has syntax errors! Commit aborted."
    rm -rf c-bye/bye
    exit 1
fi

echo "✅ [Success] Code compiles perfectly!"
rm -rf c-bye/bye
exit 0
```

4. Return to the repository root:

```bash
cd ../..
```

5. Now it's time to test it.Open `c-bye/bye.c`, delete a semicolon (`;`) to break the code, and then try to create a commit:

```bash
git add c-bye/bye.c
git commit -m "Test broken code"
```

Git will run the script, detect the compilation failure, and block the commit from being created.

### Do it yourself

1. Reset the repository:

```bash
./gh-reset-repo.sh
```

2. Write your own hook to check for linter problems. Use the `clang-format` command:

```bash
sudo apt update && sudo apt install -y clang-format
# Run clang-format on our target C file.
# --dry-run warns us about style violations without modifying the file.
# -Werror forces it to return an error code if style violations are found.
clang-format --dry-run -Werror c-bye/bye.c
```

### Using the `pre-commit` Framework

Writing Bash scripts manually for Git hooks can quickly become complicated. In production environments, developers use the `pre-commit` **framework**, which manages everything through a simple configuration file.

Instead of writing code, you just plug in ready-made validation tools.

1. Install the tool on your system:

```bash
sudo apt update && sudo apt install -y pre-commit
```

2. In the root directory of your repository, create a configuration file named `.pre-commit-config.yaml`:

```bash
touch .pre-commit-config.yaml
```

3. Open the file and add the following lines to enforce basic code hygiene (preventing large files and stripping useless trailing spaces):

```bash
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: trailing-whitespace
      - id: check-added-large-files
```

4. Register the framework with Git so it triggers automatically on every commit:

```bash
pre-commit install
```

5. Try to see if `bye.c` is correct (spoiler, it's not):

```bash
echo " " >> c-bye/bye.c
git add c-bye/bye.c
git commit -m "Added a good bye.c"
```

6. Create a **huge** file and see if it works:

```bash
truncate -s 10M huge_file.txt
git add huge_file.txt
git commit -m "Added small file"
```

## 2. GitHub Actions: Continuous Integration (CI)

While **Git Hooks** run locally on your machine, **GitHub Actions** run in the cloud on GitHub's servers every time you execute `git push` or open a *Pull Request*.

An automation configuration file (*Workflow*) is written in **YAML** format and functions like a simple recipe based on a clear hierarchy: **Event (*When?*)** ➡️ **Job (*Where?*)** ➡️ **Steps (*What actions do we execute?*)**.

### Understanding the YAML File for CI
Every line in the configuration file has a precise role:

- `on: push`: Defines the trigger. It starts automatically whenever you push code.

- `runs-on: ubuntu-latest`: Instructs GitHub to spin up a clean, isolated Linux (*Ubuntu*) virtual machine just for us.

- `uses: actions/checkout@v4`: A pre-built action that clones your repository's code inside that temporary virtual machine.

- `run: ...`: Standard terminal commands (*Bash*) that the virtual machine will execute sequentially.

### Create the Verification Pipeline

First, let's reset our repository:

```bash
pre-commit uninstall
./gh-reset-repo.sh
```

1. In the root of your repository, create the exact directory structure required by GitHub:

```bash
mkdir -p .github/workflows
```

2. Create a file named `ci.yml` inside that directory:

```bash
touch .github/workflows/ci.yml
```

3. Add the following simplified configuration to `.github/workflows/ci.yml`:

```yml
name: Code Verification (CI)

# Step 1: When should it run?
on: [push, pull_request]

# Step 2: Which runner do we use and what steps do we execute?
jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      # Clones the repository code into the virtual machine
      - name: Checkout repository code
        uses: actions/checkout@v4

      # Installs the GCC compiler on the cloud Linux runner
      - name: Install GCC
        run: sudo apt-get update && sudo apt-get install -y gcc

      # Compiles the program exactly as you would locally
      - name: Compile C program
        run: gcc c-bye/bye.c -o c-bye/bye

      # Runs the resulting binary to ensure it executes successfully
      - name: Run application
        run: ./c-bye/bye
```

4. Push the automation file to GitHub:

```bash
   git add .github/workflows/ci.yml
   git commit -m "Add CI workflow"
   git push upstream main
```

5. Open your repository in a web browser, navigate to the Actions tab, and watch the virtual machine spawn up to test your code live.

### Do it yourself

1. First, reset your repository:

```bash
./gh-reset-repo.sh
```

2. Create a CI workflow that checks for linter problems, as you did in the [hook](#automatically-verify-compilation) part of the workshop.

## 3. Containerization: Docker & GitHub Packages

Your application works on your machine and inside the GitHub virtual machine. But how do we ensure it runs flawlessly anywhere (on servers, a colleague's laptop, or in production)? We containerize it using Docker.

A container packages your application's binary together with only the bare minimum operating system files required to run it, isolated from the rest of the host system.

### The Container Recipe (Dockerfile)

Your application works on your machine and inside the GitHub virtual machine. But how do we ensure it runs flawlessly anywhere (on servers, a colleague's laptop, or in production)? We containerize it using Docker.

A container packages your application's binary together with only the bare minimum operating system files required to run it, isolated from the rest of the host system (but you already know that from the Docker workshop, hopefully).

### The Container Recipe (Dockerfile)

1. Inside the `c-bye/` directory, create a file named simply `Dockerfile`:

```bash
touch c-bye/Dockerfile
```

2. Add this minimalist, two-step recipe (Multi-stage build):

```dockerfile
# Step 1: Build environment. Use a full development image to compile the code.
FROM ubuntu:24.04 AS builder
RUN apt-get update && apt-get install -y gcc
WORKDIR /build
COPY bye.c .
# Compile statically so the binary doesn't depend on external libraries
RUN gcc bye.c -o program_bye -static

# Step 2: Final runtime environment. Extremely lightweight (around 5 MB), contains only the executable.
FROM alpine:latest
WORKDIR /app
COPY --from=builder /build/program_bye .
CMD ["./program_bye"]
```

### Automated Publishing to the Cloud (CD)
We want GitHub Actions to automatically build the Docker image above and save it to **GHCR (GitHub Container Registry)**—GitHub's built-in package manager—every time code is updated on the `main` branch.

To do this, we need a new YAML file in `.github/workflows/`. It will authenticate automatically using a temporary security token called `${{ secrets.GITHUB_TOKEN }}` that GitHub generates dynamically for each run.

1. Create the publish.yml file:

```bash
touch .github/workflows/publish.yml
```

2. Add the following configuration:

```yaml
name: Publish Docker Container

# Run only when a push happens directly on the main branch
on:
  push:
    branches: [ main ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    # Grant write permissions to the virtual machine for the repository's Packages section
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository code
        uses: actions/checkout@v4

      # Log in to the GitHub Container Registry (ghcr.io)
      - name: Authenticate with GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Builds the image using the Dockerfile and pushes it to GitHub
      - name: Build and Push Docker Image
        uses: docker/build-push-action@v5
        with:
          context: ./c-bye
          push: true
          tags: ghcr.io/${{ github.repository }}/c-bye:latest
```

3. Push the new files to the remote server:

```bash
git add c-bye/Dockerfile .github/workflows/publish.yml
git commit -m "Add Docker setup and publish workflow"
git push upstream main
```

4. Once the workflow in the **Actions** tab completes successfully, go back to the main page of your repository on GitHub. On the right-hand side, below the "Releases" section, you will see a new **Packages** section containing your container, ready to be pulled anywhere using `docker pull`.

### Do it yourself

In the `python3-hello/` directory, you already have a *Dockerfile*. Use it to create a package and post it on **GHCR**.

Be creative! Try to create your own workflows.

## 4. Do It Yourself Challenges

### Challenge 1: The Secret Sentinel

One of the most common security breaches happens when a developer accidentally pushes an AWS key or database password to a public GitHub repository. Bots scrape GitHub 24/7 looking for these leaked keys. You must stop the leak before the code ever leaves the developer's laptop.

**Goal:** Write a custom local Git Hook that prevents you from accidentally committing passwords or secret API keys.

* Create a custom `pre-commit` Bash script (do not use the `.pre-commit-config.yaml` framework for this one; write the raw script).
* The script must scan the staged changes (or the files directly) for the word `PASSWORD=`.
* If the script finds the forbidden word, it must block the commit and print a clear, highly visible warning message to the terminal.
* If the word is not found, the commit should proceed normally.


### Challenge 2: The Multi-Environment Matrix

"It works on my machine" is a banned phrase in DevOps. Your users aren't all running the exact same version of Ubuntu that you are. A robust CI pipeline tests code across multiple environments (Windows, macOS, Linux) to ensure cross-platform compatibility.

**Goal:** Upgrade your existing GitHub Actions CI workflow to test the C program on multiple operating systems simultaneously.

* Modify your `.github/workflows/ci.yml` file.
* Instead of just running on `ubuntu-latest`, the pipeline must spin up **three** different virtual machines simultaneously: `ubuntu-latest`, `macos-latest`, and `windows-latest`.
* The C program must be compiled and executed successfully on all three operating systems.


**Hints:**
* You don't need to write three separate jobs. GitHub Actions has a built-in feature to handle this automatically.
* Keep in mind that while `gcc` is standard on Linux, compiling C code on Windows might require a slightly different step, or the runner might already have a compiler installed. Check the logs if it fails!


### Challenge 3: The Complete Pipeline

A developer writes code. They try to commit it, and a local hook formats it. They push it to GitHub, and the CI server tests it. If the tests pass, the CD server packages it into a Docker image and publishes it to the registry. If any step fails, the pipeline stops, protecting production.

**Goal:** Create an end-to-end, zero-touch deployment pipeline for the `python3-hello` application using a Git Hook, a CI workflow, and a CD container push.

1. **Local (Git Hook):** Set up the `pre-commit` framework (using `.pre-commit-config.yaml`) to run a Python linter (like `flake8` or `black`) on the `python3-hello` code. You must not be able to commit badly formatted Python code.
2. **Cloud Testing (CI):** Create a *single* GitHub Actions YAML file named `python-pipeline.yml`. The first job in this file must be a `test` job that executes the Python script to ensure it doesn't crash.
3. **Cloud Publishing (CD):** In that *same* YAML file, create a second job called `build-and-push`. This job must build the `python3-hello` Docker image and push it to GHCR.
4. **The Golden Rule:** The `build-and-push` job **must not run** if the `test` job fails. It must wait for the tests to pass completely before attempting to build the container.


**Hints:**
* For the local step, search the `pre-commit` documentation for supported Python hooks. You'll need to point the YAML to the official GitHub repo of a tool like `black`.
* By default, jobs in a GitHub Actions workflow run *in parallel*. To satisfy the "Golden Rule", you need to tell GitHub that the second job *depends* on the first one. Search the GitHub Actions docs for the `needs:` keyword.
* You already have the Docker CD code from the workshop. Re-use it, but adapt the file paths to point to the `python3-hello` directory instead of the C application.