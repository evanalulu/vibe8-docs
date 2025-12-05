# Vibe8 Code Quality Metrics

Detailed documentation for all 10 code quality metrics collected by Vibe8.

## Table of Contents
- [Overview](#overview)
- [Metrics Summary](#metrics-summary)
- [Detailed Metrics](#detailed-metrics)
  - [1. Ruff Linting](#1-ruff-linting)
  - [2. Cyclomatic Complexity](#2-cyclomatic-complexity)
  - [3. Documentation Coverage](#3-documentation-coverage)
  - [4. Dead Code Detection](#4-dead-code-detection)
  - [5. Import Dependencies](#5-import-dependencies)
  - [6. Maintainability Index](#6-maintainability-index)
  - [7. Size Metrics](#7-size-metrics)
  - [8. Duplicate Code](#8-duplicate-code)
  - [9. Security Findings](#9-security-findings)
  - [10. Git Churn](#10-git-churn)
- [Running Metrics Manually](#running-metrics-manually)

## Overview

Vibe8 performs **static code analysis** on Python files without executing user code. All metrics are collected using industry-standard tools:

- **Static Analysis Only**: No code execution for maximum security
- **Comprehensive Coverage**: 10 distinct metrics covering different code quality aspects
- **Graceful Degradation**: Individual metric failures don't stop overall analysis
- **JSONB Storage**: Results stored in PostgreSQL for historical tracking

**Important: Dual Output Structure**

Each metric collector generates both:
1. **Aggregate metrics** (total, averages, scores) - Stored in `metrics` column, returned in all API responses
2. **Per-file metrics** (file-level breakdowns) - Used by file tree generator, stored in `file_tree` column

The "Output Schema" sections below show the **collector's internal output** (includes per_file). However, API responses only return aggregate values - per-file data is accessed via the dedicated file tree endpoint (`GET /api/metrics/file-tree/{run_id}`).

### Analysis Process

1. **Code Extraction**: File uploaded or repository cloned to `tmp-work/`
2. **Metric Collection**: All 10 collectors run in sequence
3. **Result Aggregation**: Results combined into single JSON response
4. **Error Handling**: Tool failures captured in `errors` array
5. **Cleanup**: Temporary files deleted

## Metrics Summary

| Metric | Tool | Purpose | Output |
|--------|------|---------|--------|
| **Ruff Linting** | Ruff | Python code quality issues | Violation count by rule category |
| **Cyclomatic Complexity** | Radon CC | Function-level complexity | Max, avg, top complex functions |
| **Documentation Coverage** | docstr_coverage | Missing docstrings | Coverage % and missing count |
| **Dead Code** | Vulture | Unused code detection | List of unused symbols |
| **Import Dependencies** | AST Parser | Import analysis | Module counts, package health |
| **Maintainability Index** | Radon MI | Code maintainability scoring | Average and minimum MI |
| **Size Metrics** | Radon raw | SLOC and file statistics | File count, SLOC, averages |
| **Duplicate Code** | PMD CPD | Code duplication | Duplicate lines, blocks, pairs |
| **Security Findings** | Bandit | Security vulnerabilities | Issues by severity level |
| **Git Churn** | Git | Code change frequency | Commit count, lines changed, top files |

---

## Detailed Metrics

### 1. Ruff Linting

**Tool**: [Ruff](https://github.com/astral-sh/ruff) - Fast Python linter
**Collector**: `backend/src/infrastructure/metrics/collectors/ruff.py`

#### Purpose
Detects Python code quality issues based on PEP 8 and other style guides.

#### Configuration
- **Rules**: Default Ruff ruleset (excluding docstring D-rules from main lint)
- **Format**: JSON output for parsing
- **Scope**: All `.py` files in uploaded code

#### Output Schema
```json
{
  "total": 15,
  "per_file": {
    "src/main.py": 5,
    "src/utils.py": 10
  },
  "issues_per_100_loc": 1.5,
  "linting_score": 92.5
}
```

#### Calculated Fields
- **issues_per_100_loc**: Lint issues per 100 lines of code (transient, calculated at response time, not persisted)
- **linting_score**: 0-100 score (higher=better), formula: `max(0, 100 - (issues_per_100_loc * 5))` (persisted)

#### Rule Categories
- **E**: Error rules (indentation, whitespace, line length)
- **F**: Pyflakes (unused imports, undefined variables)
- **I**: Import sorting and organization
- **N**: Naming conventions (PEP 8 naming)
- **UP**: Python upgrade recommendations (modern syntax)
- **W**: Warning rules (deprecated features)
- **C**: Convention rules (code style)

#### Interpretation
- **0 violations**: Excellent code quality
- **1-10 violations**: Minor issues, easy to fix
- **10-50 violations**: Moderate issues, needs attention
- **50+ violations**: Significant quality problems

#### Manual Execution
```bash
cd backend
ruff check --output-format=json path/to/code
```

---

### 2. Cyclomatic Complexity

**Tool**: [Radon CC](https://radon.readthedocs.io/) - Complexity analyzer
**Collector**: `backend/src/infrastructure/metrics/collectors/cyclomatic_complexity.py`

#### Purpose
Measures function and method complexity to identify difficult-to-test code.

#### Configuration
- **Format**: JSON output
- **Scope**: All functions and methods in `.py` files
- **Threshold**: No enforced limits (informational only)

#### Output Schema
```json
{
  "avg_cc": 3.2,
  "per_file": {
    "src/module.py": {"max": 15, "avg": 8.5},
    "src/utils.py": {"max": 12, "avg": 6.2}
  },
  "complexity_score": 84.0
}
```

#### Calculated Fields
- **complexity_score**: 0-100 score (higher=better), formula: `max(0, 100 - (avg_cc * 5))` (persisted)

#### Complexity Levels
- **1-5**: Simple, easy to test (Low risk)
- **6-10**: Moderate complexity (Medium risk)
- **11-20**: Complex, difficult to test (High risk)
- **21+**: Very complex, refactor recommended (Very high risk)

#### Interpretation
- **avg_cc < 5**: Well-structured, maintainable code
- **avg_cc 5-10**: Reasonable complexity
- **avg_cc > 10**: Consider refactoring high-complexity functions
- **max_cc > 15**: Highest priority for refactoring

#### Manual Execution
```bash
cd backend
radon cc --json --show-complexity path/to/code
```

---

### 3. Documentation Coverage

**Tool**: [docstr_coverage](https://github.com/HunterMcGushion/docstr_coverage) - Docstring coverage analyzer
**Collector**: `backend/src/infrastructure/metrics/collectors/documentation.py`

#### Purpose
Analyzes documentation coverage by counting actual documentable items (modules, classes, functions, methods) and identifying missing docstrings.

#### Configuration
- **Library**: docstr_coverage Python library
- **Analysis**: AST-based accurate counting of documentable items
- **Scope**: All Python constructs requiring docstrings (modules, classes, functions, methods, magic methods, private methods)

#### Output Schema
```json
{
  "coverage_pct": 75.0,
  "per_file": {
    "src/main.py": {"coverage_pct": 78.9},
    "src/utils.py": {"coverage_pct": 66.7}
  }
}
```

#### Docstring Requirements (PEP 257)
- **Module-level**: File header docstring
- **Class-level**: Class purpose and behavior
- **Function/Method-level**: Purpose, parameters, return values
- **Public API**: All public constructs must have docstrings

#### Interpretation
- **100%**: Fully documented (Excellent)
- **80-99%**: Well documented (Good)
- **50-79%**: Partially documented (Needs improvement)
- **< 50%**: Poorly documented (Critical issue)

#### Manual Execution
```bash
cd backend
python -c "from docstr_coverage import get_docstring_coverage; print(get_docstring_coverage(['path/to/code']))"
```

---

### 4. Dead Code Detection

**Tool**: [Vulture](https://github.com/jendrikseipp/vulture) - Dead code finder
**Collector**: `backend/src/infrastructure/metrics/collectors/dead_code.py`

#### Purpose
Identifies unused functions, classes, variables, and imports.

#### Configuration
- **Confidence Threshold**: Minimum 60% confidence
- **Format**: JSON output
- **Scope**: All Python code elements

#### Output Schema
```json
{
  "unused_symbols": 5,
  "per_file": {
    "src/utils.py": 3,
    "src/models.py": 2
  }
}
```

#### Detection Types
- **Functions**: Defined but never called
- **Classes**: Defined but never instantiated
- **Variables**: Assigned but never read
- **Imports**: Imported but never used
- **Attributes**: Class attributes never accessed

#### Confidence Levels
- **90-100%**: Almost certainly unused
- **70-89%**: Likely unused
- **60-69%**: Possibly unused (review manually)

#### False Positives
- Functions called dynamically (e.g., `getattr()`)
- Classes instantiated via factories
- Variables used in `__dict__` or `locals()`
- External API endpoints

#### Manual Execution
```bash
cd backend
vulture --min-confidence 60 path/to/code
```

---

### 5. Import Dependencies

**Tool**: Custom AST-based parser
**Collector**: `backend/src/infrastructure/metrics/collectors/imports.py`

#### Purpose
Analyzes import statements and tracks package dependencies with health status.

#### Configuration
- **Parser**: Python AST (Abstract Syntax Tree)
- **Package Health**: Checks against `requirements.txt` if present
- **Classification**: Third-party vs internal modules

#### Output Schema
```json
{
  "total_modules": 45,
  "package_health": [
    {
      "package": "requests",
      "installed": true,
      "outdated": false,
      "current_version": "2.31.0",
      "latest_version": "2.31.0"
    },
    {
      "package": "flask",
      "installed": true,
      "outdated": true,
      "current_version": "2.0.0",
      "latest_version": "3.0.0"
    }
  ]
}
```

#### Import Classification
- **Third-Party**: External packages (e.g., `requests`, `flask`)
- **Internal**: Project modules (e.g., `src.utils`, `app.models`)
- **Standard Library**: Python built-ins (e.g., `os`, `sys`) - not counted

#### Package Health Metrics
- **Installed**: Package found in environment
- **Outdated**: Current version < latest PyPI version
- **Version Info**: Current vs latest version comparison

#### Interpretation
- **High third-party count**: Many external dependencies (risk: version conflicts)
- **High internal count**: Good code organization and reusability
- **Outdated packages**: Security and compatibility risks

#### Manual Execution
```python
import ast
with open('file.py') as f:
    tree = ast.parse(f.read())
    imports = [node for node in ast.walk(tree) if isinstance(node, (ast.Import, ast.ImportFrom))]
```

---

### 6. Maintainability Index

**Tool**: Radon MI - Maintainability Index calculator
**Collector**: `backend/src/infrastructure/metrics/collectors/maintainability.py`

#### Purpose
Scores code maintainability based on complexity, SLOC, and code volume.

#### Configuration
- **Formula**: Microsoft's MI formula
- **Range**: 0-100 (higher is better)
- **Format**: JSON output

#### Output Schema
```json
{
  "mi_avg": 65.5,
  "per_file": {
    "src/main.py": 72.5,
    "src/utils.py": 42.3
  }
}
```

#### MI Formula Components
- **Code Volume**: Measure of code information content
- **Cyclomatic Complexity**: Function complexity
- **SLOC**: Source lines of code
- **Comments**: Percentage of comment lines

#### Interpretation
- **85-100**: Highly maintainable (Green)
- **65-84**: Moderately maintainable (Yellow)
- **20-64**: Difficult to maintain (Red)
- **< 20**: Unmaintainable (Critical)

#### Recommendations by Score
- **< 20**: Immediate refactoring required
- **20-64**: Schedule refactoring in sprint planning
- **65-84**: Monitor and improve gradually
- **85-100**: Maintain current quality

#### Manual Execution
```bash
cd backend
radon mi --json --show path/to/code
```

---

### 7. Size Metrics

**Tool**: Radon raw - Source code metrics
**Collector**: `backend/src/infrastructure/metrics/collectors/size.py`

#### Purpose
Measures codebase size in terms of files and source lines of code (SLOC).

#### Configuration
- **SLOC**: Counts non-blank, non-comment lines
- **Format**: JSON output
- **Scope**: All `.py` files

#### Output Schema
```json
{
  "files": 25,
  "sloc": 3500,
  "avg_file_sloc": 140,
  "per_file": {
    "src/main.py": 250,
    "src/utils.py": 180
  }
}
```

#### Metrics Definitions
- **files**: Total number of Python files
- **sloc**: Total source lines of code (excludes blanks and comments)
- **avg_file_sloc**: Average SLOC per file

#### Interpretation
- **avg_file_sloc < 200**: Well-organized files
- **avg_file_sloc 200-500**: Moderate file size
- **avg_file_sloc > 500**: Large files (consider splitting)

#### Best Practices
- Keep files under 300 SLOC
- Split large files into modules
- Use clear file organization

#### Manual Execution
```bash
cd backend
radon raw --json path/to/code
```

---

### 8. Duplicate Code

**Tool**: [PMD CPD](https://pmd.github.io/) - Copy/Paste Detector
**Collector**: `backend/src/infrastructure/metrics/collectors/duplication.py`

#### Purpose
Detects code duplication to identify refactoring opportunities.

#### Configuration
- **Minimum Tokens**: 50 tokens (configurable)
- **Language**: Python
- **Format**: XML output (parsed to JSON)

#### Output Schema
```json
{
  "duplicate_lines": 125,
  "per_file": {
    "src/utils.py": 35,
    "src/helpers.py": 35,
    "src/models.py": 20
  },
  "duplicate_pct": 12.5,
  "duplication_score": 87.5
}
```

#### Calculated Fields
- **duplicate_pct**: Percentage of code that is duplicated, formula: `(duplicate_lines * 100) / sloc` (persisted)
- **duplication_score**: 0-100 score (higher=better), formula: `max(0, 100 - duplicate_pct)` (persisted)

#### Metrics Definitions
- **duplicate_lines**: Total lines duplicated across codebase

#### Interpretation
- **0 duplicates**: Excellent (no duplication)
- **1-5% duplication**: Acceptable
- **5-10% duplication**: Moderate issue
- **> 10% duplication**: High refactoring priority

#### Refactoring Strategies
- Extract duplicated code into functions
- Use inheritance or composition
- Create utility modules for common logic
- Apply DRY (Don't Repeat Yourself) principle

#### Manual Execution
```bash
cd backend
pmd cpd --minimum-tokens 50 --language python --dir path/to/code --format xml
```

---

### 9. Security Findings

**Tool**: [Bandit](https://bandit.readthedocs.io/) - Security linter
**Collector**: `backend/src/infrastructure/metrics/collectors/security.py`

#### Purpose
Scans code for common security issues and vulnerabilities.

#### Configuration
- **Severity Levels**: High, Medium, Low
- **Format**: JSON output
- **Scope**: All Python code

#### Output Schema
```json
{
  "high": 2,
  "medium": 5,
  "low": 8,
  "per_file": {
    "src/auth.py": {"high": 2, "medium": 1, "low": 3},
    "src/database.py": {"high": 0, "medium": 4, "low": 5}
  },
  "security_score": 55.0
}
```

#### Calculated Fields
- **security_score**: 0-100 score (higher=better), formula: `max(0, 100 - min(100, high*10 + medium*5 + low))` (persisted)

#### Security Issue Categories
- **High Severity**:
  - SQL injection
  - Command injection
  - Path traversal
  - Insecure cryptography (MD5, SHA1)
  - Hardcoded credentials

- **Medium Severity**:
  - Weak cryptographic keys
  - Insecure temp file usage
  - XML vulnerabilities (XXE)
  - Pickle usage

- **Low Severity**:
  - Assert usage
  - Exec/eval usage
  - Insecure random number generation

#### Confidence Levels
- **HIGH**: Almost certainly a security issue
- **MEDIUM**: Likely a security issue (manual review needed)
- **LOW**: Possible issue (context-dependent)

#### Interpretation
- **0 high**: Good security posture
- **1+ high**: Immediate remediation required
- **5+ medium**: Schedule security review
- **10+ low**: Review and document decisions

#### Manual Execution
```bash
cd backend
bandit --recursive --format json path/to/code
```

---

### 10. Git Churn

**Tool**: [Git](https://git-scm.com/) - Version control system
**Collector**: `backend/src/infrastructure/metrics/collectors/churn.py`

#### Purpose
Analyzes git commit history to measure code change frequency and identify high-churn files that may need refactoring or have quality issues.

#### Configuration
- **Command**: `git log --numstat` for file-level statistics
- **History**: Full repository history (requires complete clone, not shallow)
- **Recent Window**: Last 30 days tracked separately
- **Scope**: Python files only

#### Output Schema
```json
{
  "total_commits": 125,
  "total_churn": 11700,
  "per_file": {
    "src/main.py": 450,
    "src/utils.py": 320
  },
  "churn_score": 0.0
}
```

#### Calculated Fields
- **churn_score**: 0-100 score (higher=better), formula: `max(0, 100 - (totalChurn / 10))` (capped at 0 for churn >= 1000) (persisted)

#### Metrics Definitions
- **total_commits**: Total number of commits in repository history
- **total_churn**: Sum of lines added + deleted across all commits (measure of code volatility)
- **per_file**: Churn value for each file in the codebase

#### Interpretation
- **High churn**: Frequently changing code (may indicate instability or active development)
- **Low churn**: Stable code (may indicate maturity or neglect)
- **High churn files**: Files with high churn values are candidates for:
  - Refactoring (too complex, needs splitting)
  - Additional testing (high change frequency = high bug risk)
  - Code review focus (changes may introduce defects)

#### Churn Levels
- **0 churn**: No git history or file never changed
- **< 100 churn**: Stable file (Low volatility)
- **100-500 churn**: Moderate changes (Medium volatility)
- **500-1000 churn**: High activity (High volatility)
- **> 1000 churn**: Very high churn (Refactoring candidate)

#### Git Repository Requirements
- **Full clone**: Requires complete git history (not shallow clone)
- **Non-git sources**: ZIP uploads without `.git` directory return `null` values
- **Directory detection**: Only analyzes git repositories in the uploaded code itself, not parent directories

#### Null Values
When no git repository is detected (e.g., ZIP file uploads), all churn metrics return `null` instead of `0`:
```json
{
  "total_commits": null,
  "total_churn": null,
  "per_file": null,
  "churn_score": null
}
```
This distinguishes "no git history" from "zero changes".

#### Use Cases
- **Hotspot Detection**: Identify files that change frequently (high per-file churn values) and may need refactoring
- **Risk Assessment**: High-churn files have higher bug risk
- **Code Review Focus**: Prioritize review of high-churn files
- **Team Insights**: Understand which parts of codebase are actively developed based on per-file churn data

#### Manual Execution
```bash
cd path/to/repo
# Overall statistics
git log --numstat --no-merges --pretty=format:"COMMIT:%H:%ct"

# Recent changes (last 30 days)
git log --since="30 days" --numstat --no-merges --pretty=format:"COMMIT:%H:%ct"

# Per-file churn (requires manual parsing)
git log --numstat --no-merges | grep "\.py$"
```

---

## Running Metrics Manually

### Individual Metrics

```bash
# Linting
ruff check --output-format=json src/

# Cyclomatic Complexity
radon cc --json --show-complexity src/

# Documentation Coverage
python -c "from docstr_coverage import get_docstring_coverage; print(get_docstring_coverage(['src/']))"

# Dead Code
vulture --min-confidence 60 src/

# Maintainability Index
radon mi --json --show src/

# Size Metrics
radon raw --json src/

# Duplicate Code (requires PMD)
pmd cpd --minimum-tokens 50 --language python --dir src/ --format xml

# Security Scan
bandit --recursive --format json src/

# Git Churn (requires git repository)
git log --numstat --no-merges --pretty=format:"COMMIT:%H:%ct"
```

### All Metrics via API

```bash
# Upload file for async analysis
curl -H "Authorization: Bearer YOUR_TOKEN" \
     -F "file=@your_file.py" \
     http://127.0.0.1:8000/api/metrics/analyze/upload/async

# Poll job status
curl -H "Authorization: Bearer YOUR_TOKEN" \
     http://127.0.0.1:8000/api/metrics/jobs/{job_id}

# Get results
curl -H "Authorization: Bearer YOUR_TOKEN" \
     http://127.0.0.1:8000/api/metrics/jobs/{job_id}/result
```

## Related Documentation

- **[API Reference](API.md)**: Analysis endpoints and response schemas
- **[Design Patterns](DESIGN.md)**: Collector pattern implementation

---

**Last Updated**: 2025-11-20
