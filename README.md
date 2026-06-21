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

### Introductory exercise
Before we tackle code formatting, let's build a simple hook to prevent
us from accidentally committing unfinished work. We will write a script
that searches for the word "TODO" in our files.

First, reset the repository:

```bash
./gh-reset-repo.sh
```

Navigate to the hooks directory and create the pre-commit file:

```bash
cd .git/hooks/
touch pre-commit
chmod +x pre-commit
```

Open the pre-commit file and add the following bash script. It uses
grep to search for "TODO" inside the `c-bye` directory:

```bash
#!/bin/bash
echo "[Hook] Checking for forgotten TODOs..."

# grep -r searches recursively; -q makes it quiet.
if grep -r -q "TODO" c-bye/; then
    echo "[Hook] You left a TODO in your code! Finish it before committing."
    exit 1
fi

echo "[Hook] No TODOs found. Proceeding!"
exit 0
```

Return to the repository root:

```bash
cd ../..
```

Test the hook by adding a TODO to your code and trying to commit:

```bash
echo "// TODO: Refactor this later" >> c-bye/bye.c
git add c-bye/bye.c
git commit -m "Add a temporary fix"
```

Git will block the commit. Remove the line manually from `bye.c`, add
it again, and commit your changes!

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

### Pre-Commit Hooks

We will look now at how to enforce coding standards via pre-commit
hooks. We will write a custom hook that uses `clang-format` to block
poorly formatted C code from being committed.

First, reset the repository to ensure a clean state:

```bash
./gh-reset-repo.sh
```

Navigate to the hooks directory and create your executable hook:

```bash
cd .git/hooks/
touch pre-commit
chmod +x pre-commit
```

Open the pre-commit file and add the following bash script. It runs
`clang-format` with the `-Werror` flag, which forces an exit code of 1
if style violations are found:

```bash
#!/bin/bash
echo "[Hook] Checking code formatting..."

# Run clang-format in dry-run mode. It won't modify files, but will
# fail if they are unformatted.
clang-format --dry-run -Werror c-bye/bye.c

# $? is the exit code of our previous command, clang-format
if [ $? -ne 0 ]; then
    echo "[Hook] Code is poorly formatted! Commit aborted."
    echo "[Hook] Run 'clang-format -i c-bye/bye.c' to fix it automatically."
    exit 1
fi

echo "[Hook] Code format check passed!"
exit 0
```

Return to the repository root:

```bash
cd ../..
```

Now, intentionally mess up the formatting in `c-bye/bye.c` (for example
add random spaces or misaligned brackets) and try to commit it:

```bash
git add c-bye/bye.c
git commit -m "Test formatting hook with messy code"
```

Git will automatically run your script, detect the poor formatting, and
block the commit. Now, fix the code automatically using the in-place
`-i` flag, and successfully commit:

```bash
clang-format -i c-bye/bye.c
git add c-bye/bye.c
git commit -m "Add cleanly formatted C code"
```

### Hooks and Git Push
Hooks aren't just for commits. Another incredibly useful hook is
pre-push, which runs right before your code is sent to GitHub. Let's
create a hook that ensures our code actually compiles before we are
allowed to push it.

Ensure you are in the repository root, then navigate to the `hooks`
directory and create a pre-push file:

```bash
cd .git/hooks/
touch pre-push
chmod +x pre-push
```

Add the following script to compile the C program. If compilation
fails, the push is aborted:

```bash
#!/bin/bash
echo "[Hook] Verifying compilation before pushing..."

# Attempt to compile
gcc c-bye/bye.c -o c-bye/bye

if [ $? -ne 0 ]; then
    echo "[Hook] Code does not compile! Push aborted to protect the remote repository."
    exit 1
fi

echo "[Hook] Code compiles cleanly. Pushing to GitHub..."
rm -f c-bye/bye
exit 0
```

Return to the repository root:

```bash
cd ../..
```

Test it by intentionally breaking `c-bye/bye.c` (i.e. delete a
semicolon), committing it using `--no-verify` (to bypass the formatting
hook from earlier), and attempting to push:

```bash
git commit -a -m "Introduce a compilation error" --no-verify
git push upstream main
```

The push will fail, saving you from uploading broken code!

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

### GitHub Actions
Before setting up a linter, let's verify that we can spin up a virtual
machine in the cloud and make it execute a simple command.

First, reset the repository:

```bash
./gh-reset-repo.sh
```

Create a new directory for workflows and add a basic `.yaml` file:

```bash
mkdir -p .github/workflows
touch .github/workflows/hello.yml
```

Add the following configuration. This workflow simply boots up an
Ubuntu machine and prints a welcome message using GitHub's dynamic
environment variables:

```yaml
name: Hello Cloud

on: [push]

jobs:
  say-hello:
    runs-on: ubuntu-latest
    steps:
      - name: Print greeting
        run: echo "Hello! This job was triggered by ${{ github.actor }}."
```

Push this to GitHub:

```bash
git add .github/workflows/hello.yml
git commit -m "Add Hello Cloud workflow"
git push upstream main
```

Open your repository on GitHub, go to the Actions tab, click on the
`Hello Cloud` workflow, and look inside the `Print greeting` step to
see your username printed by the cloud server!

### Removing Hook Bypassing
Local hooks are great, but developers can easily bypass them (i.e.
using `git commit --no-verify`). To truly protect our codebase, we need
a cloud-based linter that runs automatically on GitHub Actions.

First, reset the repository:
```bash
./gh-reset-repo.sh
```

Create a new workflow file specifically for linting:
```bash
mkdir -p .github/workflows
touch .github/workflows/lint.yml
```

Add the following configuration to `.github/workflows/lint.yml`. Notice
how the steps mirror exactly what we executed manually on our local
machine:

```yaml
name: Code Linter (CI)

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository code
        uses: actions/checkout@v4

      - name: Install clang-format
        run: sudo apt-get update && sudo apt-get install -y clang-format

      - name: Verify code formatting
        run: clang-format --dry-run -Werror c-bye/bye.c
```

Commit and push your new pipeline to the cloud:
```bash
git add .github/workflows/lint.yml
git commit -m "Add cloud linter workflow"
git push upstream main
```

Test the automation: Intentionally break the formatting in `c-bye/bye.c`
again, commit, and push the changes. Open your repository in a web
browser, navigate to the Actions tab, and watch the pipeline catch the
error and fail.
Once you see the error message, run `clang-format -i c-bye/bye.c`
locally, push the fix, and watch the pipeline work!


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

### CI Actions
Continuous Integration isn't just about verifying code; it's also about
packaging the results. If our C program compiles successfully in the
cloud, we should save the compiled executable so users can download it
directly from GitHub.
We will update the `ci.yml` file you created earlier to upload the
binary as an "Artifact".
Open your existing `.github/workflows/ci.yml` and add a new step at the
very bottom. Your final steps should look like this:

```yaml
steps:
      - name: Checkout repository code
        uses: actions/checkout@v4

      - name: Install GCC
        run: sudo apt-get update && sudo apt-get install -y gcc

      - name: Compile C program
        run: gcc c-bye/bye.c -o c-bye/bye_executable

      # New step: Save the binary to GitHub
      - name: Upload Binary Artifact
        uses: actions/upload-artifact@v4
        with:
          name: compiled-program
          path: c-bye/bye_executable
```

Commit and push the update:

```bash
git add .github/workflows/ci.yml
git commit -m "Add artifact upload to CI"
git push upstream main
```

Navigate to the Actions tab on GitHub. Once the workflow completes, scroll to the bottom of the summary page. You will see an "Artifacts" section containing compiled-program.zip. You can now download the executable that the cloud just built!

### Containers
Before we containerize our own application, let's understand what lightweight containers do. We will pull and run *Alpine Linux*, a minimalist operating system that is only 5 MB in size, and execute a quick command inside it.

Make sure the Docker daemon is running on your machine, then execute this single line:

```bash
docker run --rm alpine:latest echo "Hello from inside an isolated container!"
```

What just happened?
1. Docker checked if you had the alpine:latest image locally.
1. Since you didn't, it pulled (downloaded) it from the public Docker
Hub.
1. It created an isolated container using that image.
1. It executed the echo command inside the container, printed to your
terminal, and immediately deleted the container (--rm) when it finished.

### Docker Containers and GitHub
Let's containerize the C++ application `cpp-bye`, test it locally, and
build an automated CD pipeline to publish it to the GitHub Container
Registry.

First, reset the repository:
```bash
./gh-reset-repo.sh
```

Navigate to the C++ application directory. You will find a Dockerfile
already prepared for you:

```bash
cd cpp-bye/
```

Build the container image locally. The `-t` flag tags the image with a
memorable local name (`cpp-bye-local`):

```bash
docker build -t cpp-bye-local .
```

Test the container to ensure it runs correctly and outputs its expected
message on your machine:

```bash
docker run --rm cpp-bye-local
```

Now, let's automate this deployment. Return to the root of your
repository and create a new workflow file:
```bash
cd ..
touch .github/workflows/publish-cpp.yml
```

Add the following `.yaml` configuration. It uses the same
authentication logic from the earlier tutorial but targets the
`cpp-bye` directory as the build context:
```yaml
name: Publish C++ Docker Container

on:
  push:
    branches: [ main ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository code
        uses: actions/checkout@v4

      - name: Authenticate with GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push Docker Image
        uses: docker/build-push-action@v5
        with:
          context: ./cpp-bye
          push: true
          tags: ghcr.io/${{ github.repository }}/cpp-bye:latest
```

Push the setup to GitHub:
```bash
git add .github/workflows/publish-cpp.yml
git commit -m "Add C++ Docker publish workflow"
git push upstream main
```

Navigate to the Actions tab on GitHub to watch your image build. Once
the workflow completes successfully, check the Packages section on the
main page of your repository. You will see your newly published C++
container, ready to be pulled via `docker pull`!
