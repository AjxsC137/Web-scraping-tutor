# 🐍 Apache Jira Issue Data Pipeline for LLM Training

This project contains a Python script delivered as a Jupyter/Colab notebook (`Web_scraping_Tutor-1.2.ipynb`) designed to ingest public issue data from the Apache Jira instance (issues.apache.org/jira/), process it, and transform it into a JSONL corpus.

This corpus is structured to be immediately usable for instruction-tuning Large Language Models (LLMs) on software development and issue tracking tasks.

## 🏗️ Pipeline Summary

The provided notebook executes a full data pipeline, leveraging the official Jira REST API:

### 1. Project Discovery
Accesses the `/project` API endpoint to find all available Apache projects.

### 2. Project Selection
Selects key projects (specifically KAFKA, SPARK, and HADOOP) for data collection.

### 3. Scraping (ETL - Extract)
Uses the `/search` API endpoint with JQL (`project=KAFKA ORDER BY key ASC`) to fetch all issues for a project (shown for KAFKA). This step includes robust features:

- **Pagination**: Fetches issues in batches (`maxResults=500`) by incrementing the `startAt` parameter.
- **Resumability**: Saves and loads progress (`startAt` offset) using a `kafka_scrape_progress.json` file to restart scraping exactly where it left off.
- **Error Handling**: Implements `try...except` blocks and a retry loop (`max_retries = 5`) for network failures and bad responses.

### 4. Transformation (ETL - Transform)
For each raw Jira issue:

- **Data Extraction**: Extracts core fields: `key`, `summary`, `description`, `status`, `reporter`, `created`, and `updated`.
- **Cleaning**: Uses `re.sub(r'<.*?>', '', ...)` to strip HTML/Jira markup tags from the summary, description, and comments fields.
- **Task Derivation (LLM-ready format)**: Generates three structured task examples, suitable for instruction tuning:
  - `summarization_task` (Input: Summary/Description/Comments, Output: Placeholder)
  - `classification_task` (Input: Summary/Description, Output: Status)
  - `qna_task` (Input: Issue text + Question, Output: Summary)

### 5. Loading (ETL - Load)
Writes the transformed data to a JSONL file (`kafka_issues_transformed.jsonl`).

### 6. Combination
Merges the JSONL files from different projects (SPARK, HADOOP, KAFKA) into one final corpus: `all_issues_transformed.jsonl`.

## 💡 Key Features for Data Quality and Reliability

| Feature | Implementation in Code | Benefit |
|---------|------------------------|---------|
| **Resumable Scraping** | Checks for `kafka_scrape_progress.json` to set the starting `start_at` offset and reloads `kafka_issues_raw.json` to prevent fetching duplicates. | Allows the process to be interrupted and restarted without losing progress, saving time and bandwidth. |
| **Retry Logic** | An inner `for attempt in range(max_retries)` loop with `time.sleep(retry_delay)` and `response.raise_for_status()`. | Makes the scraper resilient to transient network errors or temporary API rate limits. |
| **Data Cleaning** | Uses `re.sub(r'<.*?>', '', ...)` to remove HTML/Jira markup. | Creates a cleaner, text-only corpus, which improves the quality of the LLM training data. |
| **LLM Task Derivation** | Generates `summarization_task`, `classification_task`, and `qna_task` dictionaries for every issue. | Converts a general dataset into a high-value instruction-tuning corpus for a variety of NLP tasks. |
| **JSONL Output** | Writes each transformed issue as a single JSON object per line. | Standard, memory-efficient format for massive LLM training datasets. |

## 📦 Output Files

The execution of the notebook generates the following key files:

| Filename | Description | Size (Example) |
|----------|-------------|----------------|
| `kafka_issues_raw.json` | The full, raw JSON response for all KAFKA issues. | 284 MB |
| `kafka_scrape_progress.json` | Tracks the last successful `start_at` value for resuming the scrape. | Small |
| `kafka_issues_transformed.jsonl` | KAFKA issues transformed into the final LLM-ready JSONL format. | 270.89 MB |
| `all_issues_transformed.jsonl` | The final combined corpus from all selected projects (SPARK, HADOOP, KAFKA). | 812.67 MB |

## 📄 Sample Transformed Issue

The final `all_issues_transformed.jsonl` contains objects like this:

```json
{
    "key": "SPARK-290",
    "summary": "Use SPARK_MASTER_IP if it is set in start-slaves.sh.",
    "description": "Also check to prompt user if it is not set and the script cannot figure out the master's ip.",
    "status": "Resolved",
    "summarization_task": {
        "instruction": "Summarize the following Jira issue:",
        "input": "Summary: ...\nDescription: ...\nComments: ",
        "output": "Generated summary goes here."
    },
    "classification_task": {
        "instruction": "Classify the type of the following Jira issue:",
        "input": "Summary: ...\nDescription: ...",
        "output": "Resolved"
    },
    "qna_task": {
        "instruction": "Answer the following question based on the Jira issue:",
        "input": "Issue: ...\nQuestion: What is the main problem described in this issue?",
        "output": "Use SPARK_MASTER_IP if it is set in start-slaves.sh."
    }
}
```

## 🚀 Getting Started

1. Open `Web_scraping_Tutor-1.2.ipynb` in Jupyter or Google Colab
2. Run all cells to execute the complete pipeline
3. The final corpus will be saved as `all_issues_transformed.jsonl`

## 📋 Requirements

- Python 3.x
- `requests` library
- `re` (built-in)
- `json` (built-in)
- `time` (built-in)
