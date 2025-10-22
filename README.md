# Project Overview

This repository contains a Python script for data processing, an Excel data source, and a GitHub Actions workflow to automate the process and publish results.

## Files

- `execute.py`: A Python script that reads `data.csv`, performs aggregation, and outputs the result as JSON.
- `data.xlsx`: The original Excel data file (provided separately).
- `data.csv`: The CSV version of `data.xlsx`, used by `execute.py`.
- `.github/workflows/ci.yml`: GitHub Actions workflow for continuous integration and deployment.
- `index.html`: A simple, responsive HTML page to serve as a landing for the project and link to the results.
- `LICENSE`: The MIT License.

## Setup and Usage

### Prerequisites

- Python 3.11+
- pip (Python package installer)
- pandas 2.3+
- ruff (for linting)

### Local Development

1.  **Clone the repository:**
    ```bash
    git clone <your-repo-url>
    cd <your-repo-name>
    ```

2.  **Convert `data.xlsx` to `data.csv`:**
    The `execute.py` script operates on `data.csv`. You need to convert `data.xlsx` (which was provided separately) to `data.csv` first. It's recommended to commit `data.csv` to the repository.
    ```bash
    python -c "import pandas as pd; pd.read_excel('data.xlsx').to_csv('data.csv', index=False)"
    git add data.csv
    git commit -m "Add data.csv generated from data.xlsx"
    ```
    *Example `data.xlsx` content structure (first few rows):*
    | Category | Value | Date       |
    |----------|-------|------------|
    | A        | 100   | 2023-01-01 |
    | B        | 150   | 2023-01-05 |
    | A        | 50    | 2023-01-10 |
    | C        | 200   | 2023-01-15 |
    | B        | 75    | 2023-01-20 |

3.  **Install Python dependencies:**
    ```bash
    pip install pandas ruff openpyxl
    ```

4.  **Run the script:**
    ```bash
    python execute.py > result.json
    ```
    This will generate `result.json` with the processed data.

5.  **Run Linter:**
    ```bash
    ruff .
    ```

## `execute.py` (Fixed Content)

The `execute.py` script has been fixed to ensure robust data processing. The non-trivial error addressed was related to potential `TypeError` or `ValueError` when the 'Value' column contained non-numeric entries. The fix involves coercing the 'Value' column to a numeric type, converting non-numeric entries to `NaN`, and then dropping rows with these `NaN` values, ensuring reliable aggregation. Additionally, basic error handling for file and column existence has been added.

```python
import pandas as pd
import json
import sys

def process_data(file_path="data.csv"):
    try:
        df = pd.read_csv(file_path)
    except FileNotFoundError:
        print(f"Error: The file '{file_path}' was not found.", file=sys.stderr)
        sys.exit(1)
    except Exception as e:
        print(f"Error reading CSV file: {e}", file=sys.stderr)
        sys.exit(1)

    # Ensure required columns exist
    required_columns = ['Category', 'Value']
    for col in required_columns:
        if col not in df.columns:
            print(f"Error: Required column '{col}' not found in '{file_path}'.", file=sys.stderr)
            sys.exit(1)

    # Non-trivial error fix: Ensure 'Value' column is numeric and handle invalid data.
    # Coerce errors to NaN, then drop rows where 'Value' could not be converted.
    initial_rows = len(df)
    df['Value'] = pd.to_numeric(df['Value'], errors='coerce')
    df.dropna(subset=['Value'], inplace=True)
    if len(df) < initial_rows:
        print(f"Warning: Dropped {initial_rows - len(df)} rows due to non-numeric 'Value' entries.", file=sys.stderr)

    # Perform aggregation: group by 'Category' and sum 'Value'
    summary = df.groupby('Category')['Value'].sum().reset_index()

    # Convert to dictionary and then to JSON.
    # Ensure all values are JSON serializable (e.g., convert NumPy types to Python native types).
    result = summary.to_dict(orient='records')
    
    # Convert any pandas/numpy specific types (like numpy.int64) to native Python types
    # for broader JSON compatibility, and handle NaN values.
    for item in result:
        for key, value in item.items():
            if pd.isna(value): 
                item[key] = None # Represent NaN as null in JSON
            elif isinstance(value, (int, float)): # Ensure these are standard Python types
                item[key] = value
            # Add more specific type handling if other non-standard types are expected

    return json.dumps(result, indent=2)

if __name__ == "__main__":
    print(process_data())
```

## GitHub Actions Workflow (`.github/workflows/ci.yml`) Content

This workflow automates linting, data processing, and deployment of the `result.json` to GitHub Pages. It triggers on every push to the `main` branch.

```yaml
name: CI/CD Pipeline

on: 
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: write # Needed for actions/checkout v4 to interact with the repo
      pages: write   # Needed to deploy to GitHub Pages
      id-token: write # Needed for OIDC authentication with GitHub Pages

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Set up Python 3.11
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install ruff pandas openpyxl # openpyxl is needed for reading .xlsx files

    - name: Convert data.xlsx to data.csv
      # This step ensures data.csv is always up-to-date with data.xlsx in CI.
      # data.xlsx must be present in the repository for this step to succeed.
      run: |
        python -c "import pandas as pd; pd.read_excel('data.xlsx').to_csv('data.csv', index=False)"

    - name: Run Ruff Linter
      run: |
        ruff .
        # Optional: ruff check . --output-format=github # For annotations in PRs in PR workflows

    - name: Run execute.py and generate result.json
      run: |
        python execute.py > result.json

    - name: Setup Pages
      uses: actions/configure-pages@v5

    - name: Upload artifact
      uses: actions/upload-pages-artifact@v3
      with:
        path: 'result.json' # The directory or file to upload to GitHub Pages

    - name: Deploy to GitHub Pages
      id: deployment
      uses: actions/deploy-pages@v4
```

## GitHub Pages

After a successful workflow run, the `result.json` will be available at:
`https://<YOUR_GITHUB_USERNAME>.github.io/<YOUR_REPO_NAME>/result.json`

Remember to enable GitHub Pages for your repository in the settings, choosing the `gh-pages` branch or the `GitHub Actions` source for deployment. Also, replace `<YOUR_GITHUB_USERNAME>` and `<YOUR_REPO_NAME>` with your actual GitHub username and repository name.