# epg-manager
```yaml
name: Manual EPG Operations

on:
  workflow_dispatch:
    inputs:
      operation:
        description: 'Operation to perform'
        required: true
        type: choice
        default: 'rebuild_xmltv'
        options:
          - rebuild_xmltv
          - regenerate_guide
          - test_lineup
          - debug_sd_credentials
      environment:
        description: 'Target Environment'
        required: true
        type: environment
        default: 'development'

jobs:
  execute-operation:
    name: Execute ${{ github.event.inputs.operation }}
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.13'
          
      - name: Install Dependencies
        run: |
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi

      - name: Run Operation
        env:
          SD_USERNAME: ${{ secrets.SD_USERNAME }}
          SD_PASSWORD: ${{ secrets.SD_PASSWORD }}
        run: |
          echo "Starting manual operation: ${{ github.event.inputs.operation }}"
          
          case "${{ github.event.inputs.operation }}" in
            rebuild_xmltv)
              echo "Rebuilding XMLTV..."
              # ./wrapper.sh rebuild
              ;;
            regenerate_guide)
              echo "Regenerating guide..."
              # python update_guide.py
              ;;
            test_lineup)
              echo "Testing lineup..."
              # pytest tests/integration/test_lineup.py
              ;;
            debug_sd_credentials)
              echo "Debugging SD credentials..."
              # python auth_check.py
              ;;
            *)
              echo "Unknown operation"
              exit 1
              ;;
          esac
          
      - name: Upload Artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: manual-run-${{ github.event.inputs.operation }}-${{ github.run_id }}
          path: |
            *.xml
            *.log
          retention-days: 3

```
```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
        args: ['--maxkb=500']
      - id: debug-statements

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.4
    hooks:
      - id: ruff
        args: [ --fix ]
      - id: ruff-format

``````yaml
name: Manual EPG Operations

on:
  workflow_dispatch:
    inputs:
      operation:
        description: 'Operation to perform'
        required: true
        type: choice
        default: 'rebuild_xmltv'
        options:
          - rebuild_xmltv
          - regenerate_guide
          - test_lineup
          - debug_sd_credentials
      environment:
        description: 'Target Environment'
        required: true
        type: environment
        default: 'development'

jobs:
  execute-operation:
    name: Execute ${{ github.event.inputs.operation }}
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.13'
          cache: 'pip'
          
      - name: Install Dependencies
        run: |
          if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
          if [ -f requirements-dev.txt ]; then pip install -r requirements-dev.txt; fi

      - name: Run Operation
        env:
          SD_USERNAME: ${{ secrets.SD_USERNAME }}
          SD_PASSWORD: ${{ secrets.SD_PASSWORD }}
          SD_LINEUP_ID: ${{ secrets.SD_LINEUP_ID }}
        run: |
          echo "Starting manual operation: ${{ github.event.inputs.operation }}"
          
          # Map GitHub Action inputs to the core Python pipeline
          case "${{ github.event.inputs.operation }}" in
            rebuild_xmltv|regenerate_guide)
              python scripts/fetch_epg.py
              ;;
            test_lineup)
              pytest tests/ -v
              ;;
            debug_sd_credentials)
              # Executes the script up to authentication to verify secrets
              python -c "import scripts.fetch_epg as epg; epg.main()" || exit 1
              ;;
            *)
              echo "Unknown operation"
              exit 1
              ;;
          esac
          
      - name: Upload Artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: manual-run-${{ github.event.inputs.operation }}-${{ github.run_id }}
          path: |
            data/*.xml
            logs/*.log
          retention-days: 3

```
The files should be added to the root directory of your existing repository: **service-electric-epg**.
In standard CI/CD implementations, the GitHub Actions (.github/workflows/) and pre-commit configurations (.pre-commit-config.yaml) must reside in the same repository as the application code they manage. Placing them in service-electric-epg ensures the actions/checkout@v4 step correctly pulls your scripts/fetch_epg.py engine and the pytest suite during execution.
If your architectural intention is to decouple the CI/CD orchestration and deployment manifests from the core Python logic into a standalone infrastructure repository, **epg-manager** is the most accurate naming convention. It defines the repository's role as the orchestration layer handling execution, testing, and dependency review.

