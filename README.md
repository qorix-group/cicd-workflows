# Reusable GitHub Actions Workflows

This repository contains **reusable GitHub Actions workflows** designed to standardize CI/CD processes across multiple repositories
in `SCORE`. 
These workflows integrate with **Bazel** and provide a consistent way to run **documentation builds, license checks, static analysis, tests, and copyright verification** ( **TODO** will need to be updated)

## Available Workflows

| Workflow | Description |
|----------|------------|
| **Documentation Build** | Builds project documentation and deploys it to GitHub Pages |
| **License Check** | Verifies OSS licenses and compliance |
| **Static Code Analysis** | Runs Clang-Tidy, Clippy, Pylint, and other linters based on project type |
| **Tests** | Executes tests using GoogleTest, Rust test or pytest |
| **Copyright Check** | Ensures all source files have the required copyright headers |

---

## Using the Workflows in Your Repository

To use a reusable workflow, create a workflow file inside **your repository** (e.g., `.github/workflows/ci.yml`) and reference the appropriate workflow from this repository.

### **1️ Documentation Build Workflow**
**Usage Example**
```yaml 
name: Documentation CI

on:
  pull_request:
  push:
    branches:
      - main

jobs:
  docs:
    uses: eclipse-score/ci_cd_repo/.github/workflows/docs.yml@main
    with:
      retention-days: 3
```
This workflow:

✅ Builds project documentation  
✅ Uploads it as an artifact  
✅ Deploys it to **GitHub Pages** on push to `main`  


#### TODO
We will clarify the `docs` target after it gets decoupled from the current `score` repo.


---

### **2️ License Check Workflow**
**Usage Example**
```yaml
name: License Check CI

on:
  pull_request:
  push:
    branches:
      - main

jobs:
  license-check:
    uses: eclipse-score/ci_cd_repo/.github/workflows/license-check.yml@main
    with:
      repo-url: "${{ github.server_url }}/${{ github.repository }}"
    secrets:
      gitlab-api-token: ${{ secrets.ECLIPSE_GITLAB_API_TOKEN }}
```

This workflow:

✅ Runs **DASH license compliance checks** for **Rust, C++, and Python**

✅ Uses the **organization secret** `ECLIPSE_GITLAB_API_TOKEN` by default for DASH reviews.

✅ Comments results directly on the **Pull Request**  


### **TODO: Integration with `bazel_rules` and `bazel_registry` for License Check**

To fully integrate this workflow with `bazel_rules` and `bazel_registry`, here is a possible **example** of the steps that need to be completed:

#### **1️ Define a `license-check` Alias in `bazel_rules`**
- Add an alias in `bazel_rules/BUILD.bazel` to allow running the check using a simple command:
```starlark
alias(
    name = "license-check",
    actual = "//:detect_and_run_license_check",
    visibility = ["//visibility:public"],
)
```

✅ This ensures that **repositories can run `bazel license-check` instead of `bazel run //docs:license.check.<project-type>`**.

#### **2️ Implement Auto-Detection of Project Type in `bazel_rules`**
- Create a **Bazel rule (`license_check.bzl`)** that:
  - Detects **which programming language is used** by analyzing `BUILD.bazel` files.
  - Runs the correct `license-check` command dynamically.

**Location:** `bazel_rules/license_check.bzl`
```starlark
def detect_and_run_license_check(name):
    native.sh_binary(
        name = name,
        srcs = ["license_check.sh"],
        visibility = ["//visibility:public"],
    )
```

**Location:** `bazel_rules/license_check.sh`
```bash
#!/bin/bash

# Detect project type by checking for language-specific BUILD.bazel files
if grep -q "cc_binary" BUILD.bazel; then
    PROJECT_TYPE="cpp"
elif grep -q "rust_binary" BUILD.bazel; then
    PROJECT_TYPE="rust"
elif grep -q "py_binary" BUILD.bazel"; then
    PROJECT_TYPE="python"
else
    echo "❌ ERROR: Could not detect project type."
    exit 1
fi

echo "Detected project type: $PROJECT_TYPE"

CMD="bazel run //docs:license.check.${PROJECT_TYPE}"

echo "Executing: $CMD"
$CMD
```
✅ **Now `bazel license-check` automatically detects the project type and runs the correct check.**


---

### **3️ Static Code Analysis Workflow**
**Usage Example**
```yaml
name: Static Analysis CI

on:
  pull_request:
  push:
    branches:
      - main

jobs:
  static-analysis:
    uses: eclipse-score/ci_cd_repo/.github/workflows/static-analysis.yml@main
```
This workflow:
✅ Runs **Clang-Tidy** for C++  
✅ Runs **Rust Clippy, Cargo Audit, and Cargo Geiger**  for Rust
✅ Runs **Pylint** for Python  



## **TODO: Integration with `bazel_rules` and `bazel_registry` for Static Code Analysis**

To fully integrate this workflow with `bazel_rules` and `bazel_registry`, here is an example of the steps needed to be completed:

#### **1️ Define a `static-analysis` Alias in `bazel_rules`**
- Add an alias in `bazel_rules/BUILD.bazel` to allow running the check using a simple command:
```starlark
alias(
    name = "static-analysis",
    actual = "//:detect_and_run_static_analysis",
    visibility = ["//visibility:public"],
)
```
✅ This ensures that **repositories can run `bazel static-analysis` instead of manually specifying project types.**

#### **2️ Implement Auto-Detection of Project Type in `bazel_rules`**
- Create a **Bazel rule (`static_analysis.bzl`)** that:
  - Detects **which programming language is used** by analyzing `BUILD.bazel` files.
  - Runs the correct static analysis tools dynamically.

 **Location:** `bazel_rules/static_analysis.bzl`
```starlark
def detect_and_run_static_analysis(name):
    native.sh_binary(
        name = name,
        srcs = ["static_analysis.sh"],
        visibility = ["//visibility:public"],
    )
```

**Location:** `bazel_rules/static_analysis.sh`
```bash
#!/bin/bash

# Detect project type by checking for language-specific BUILD.bazel files
if grep -q "cc_binary" BUILD.bazel; then
    PROJECT_TYPE="cpp"
elif grep -q "rust_binary" BUILD.bazel; then
    PROJECT_TYPE="rust"
elif grep -q "py_binary" BUILD.bazel; then
    PROJECT_TYPE="python"
else
    echo "❌ ERROR: Could not detect project type."
    exit 1
fi

echo "Detected project type: $PROJECT_TYPE"

case $PROJECT_TYPE in
  cpp)
    echo "Running Clang-Tidy for C++"
    bazel run @bazel-rules//linting:clang_tidy
    ;;
  rust)
    echo "Running Clippy for Rust"
    bazel run @bazel-rules//linting:rust_clippy
    echo "Running Cargo Audit for Rust"
    bazel run @bazel-rules//linting:cargo_audit
    echo "Running Cargo Geiger for Rust"
    bazel run @bazel-rules//linting:cargo_geiger
    ;;
  python)
    echo "Running Pylint for Python"
    bazel run @bazel-rules//linting:pylint
    ;;
  *)
    echo "❌ ERROR: Unknown project type: $PROJECT_TYPE"
    exit 1
    ;;
esac
```
✅ **Now `bazel static-analysis` automatically detects the project type and runs the correct checks.**


---

### **4️ Tests Workflow**
**Usage Example**
```yaml
name: Test CI

on:
  pull_request:
  push:
    branches:
      - main

jobs:
  tests:
    uses: eclipse-score/ci_cd_repo/.github/workflows/tests.yml@main
```
This workflow:
✅ Runs **GoogleTest** for C++  
✅ Runs **Rust Unit Tests**  
✅ Runs **pytest** for Python  


---

### **5️ Copyright Check Workflow**
**Usage Example**
```yaml
name: Copyright Check CI

on:
  pull_request:
  push:
    branches:
      - main

jobs:
  copyright-check:
    uses: eclipse-score/ci_cd_repo/.github/workflows/copyright.yml@main
```
This workflow:
✅ Runs a **Bazel-based copyright verification check**  
✅ Ensures all files have the required **Eclipse Foundation** copyright headers  

#### TODO: Integration with bazel_rules and bazel_registry
To fully integrate this workflow with bazel_rules and bazel_registry, the following steps need to be completed:


**Define an Alias in bazel_rules**
Add an alias in bazel_rules/BUILD.bazel to allow running the check using a simple command, something like:

```starlark
alias(
    name = "copyright-check",
    actual = "//:copyright.check",
    visibility = ["//visibility:public"],
)
```
✅ This ensures that repositories can run `bazel copyright-check` instead of `bazel run //:copyright.check`.

Ensure that bazel_rules is properly published in bazel_registry so that all repositories can reference it.

Update bazel_registry/MODULE.bazel to include:

```starlark

bazel_dep(name = "bazel_rules", version = "1.0.0")
````
✅ This ensures that any repository using bazel_registry gets bazel_rules, including the new alias.


---

##  How to Update Workflows
Since these workflows are centralized, updates in the `ci_cd_repo` repository will **automatically apply to all repositories using them**. If you need a specific version, reference a **tagged release** instead of `main`:
```yaml
uses: eclipse-score/ci_cd_repo/.github/workflows/tests.yml@v1.0.0
```

---



### **Summary**
✅ **Standardized** CI/CD workflows across all projects  
✅ **Reusable & Maintainable** with centralized updates  
✅ **Bazel-powered** for consistent testing & analysis  


