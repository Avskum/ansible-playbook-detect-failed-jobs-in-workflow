# AAP Workflow Failure Analyzer

This Ansible playbook is designed to analyze a completed or running Ansible Automation Platform (AAP) workflow job. It identifies any failed jobs within that workflow, retrieves their output (stdout), parses common Ansible error messages, and provides a summary of the failures.

This is useful for:
*   Automated post-workflow checks.
*   Integrating with ticketing systems (e.g., ServiceNow) by setting an outcome status.
*   Gathering detailed failure information for easier troubleshooting.

## Prerequisites

1.  **Ansible:** The playbook is written for Ansible.
2.  **Ansible Automation Platform (AAP) / AWX / Tower:** You need an AAP (or its predecessors AWX/Tower) instance where your workflows run.
3.  **AAP Credential:**
    *   A "Controller Credential" (or "Ansible Tower"/"AWX" credential type) must be configured in AAP.
    *   This credential needs API access to your AAP instance (e.g., `https://your-aap-instance.com`) with a user that has permissions to read workflow job details and job outputs.
    *   This credential will be associated with the Job Template that runs this playbook.
4.  **Workflow Job ID:** You need the ID of the AAP workflow job you want to analyze.

## How to Use

1.  **Save the Playbook:** Save the playbook code (e.g., `analyze_workflow.yml`).
2.  **Create a Job Template in AAP:**
    *   Go to your AAP instance.
    *   Create a new Job Template.
    *   Point it to the playbook file you saved.
    *   Select an inventory (can be a dummy inventory or localhost if your AAP is set up for it, as this playbook runs tasks on `localhost` via `delegate_to`).
3.  **Assign the AAP Credential:**
    *   In the Job Template settings, assign the "Controller Credential" (created in Prerequisites) to this Job Template. This allows the playbook to securely access the AAP API using environment variables like `TOWER_HOST`, `TOWER_USERNAME`, etc.
4.  **Provide Inputs (Extra Variables):**
    *   When launching this Job Template, you **must** provide the `tower_workflow_job_id` as an Extra Variable.
        ```yaml
        tower_workflow_job_id: 12345
        ```
        (Replace `12345` with the actual ID of the workflow job you want to analyze).
    *   Optionally, you can provide an initial `wt_outcome` if integrating with systems like ServiceNow:
        ```yaml
        wt_outcome: "IN_PROGRESS"
        ```
        If not provided, it defaults to "SCHEDULED".

5.  **Run the Job Template:**
    *   Launch the Job Template. It will connect to your AAP API, fetch details about the specified workflow job, and analyze it.

## How it Works

*   **Input ID:** Takes a `tower_workflow_job_id`.
*   **API Calls:** Uses the `ansible.builtin.uri` module to make REST API calls to your AAP instance.
    *   Fetches all jobs (nodes) within the specified workflow.
    *   Identifies jobs with a "failed" status.
    *   For each failed job, fetches its standard output (stdout).
*   **Error Parsing:** Uses regular expressions to parse the stdout of failed jobs to find:
    *   Lines containing "fatal: ... FAILED!".
    *   The "PLAY RECAP" section.
    *   Specific error messages (the `msg` field in Ansible errors).
*   **Sets Outcome:**
    *   Sets a variable `wt_outcome` to "FAILED" if any job in the workflow failed, or "COMPLETED" if all jobs succeeded (and `wt_outcome` was initially 'SCHEDULED' or 'IN_PROGRESS'). This is useful for integrations.
*   **Provides Summary (`set_stats`):**
    *   Aggregates all findings into a `workflow_failures_summary` variable.
    *   Uses `ansible.builtin.set_stats` to make this summary, `wt_outcome`, and other details (like `_failed_hosts_info` and `failure_reason`) available for use by subsequent AAP workflow nodes or callback mechanisms.

## Key Outputs (available via `set_stats`)

The playbook makes the following information available as Ansible stats, which can be used by AAP:

*   `wt_outcome`: The final status ("FAILED" or "COMPLETED").
*   `failure_reason`: A high-level reason for failure, often the first detailed error message found.
*   `_failed_hosts_info`: A list of hostnames that reported failures across all failed jobs.
*   `workflow_failures_summary`: A dictionary containing:
    *   `workflow_id`: The ID of the analyzed workflow.
    *   `failed_jobs_count`: Number of jobs that failed.
    *   `failed_jobs`: A dictionary where each key is a failed job's ID, and the value contains:
        *   `job_name`
        *   `has_failures` (boolean)
        *   `failed_task_count`
        *   `failure_lines` (list of raw failure lines)
        *   `play_recap`
        *   `failed_hosts` (list for that specific job)
        *   `final_error` (extracted "msg" content)
    *   `final_wt_outcome`: The final `wt_outcome`.

## Important Notes

*   **`delegate_to: localhost` & `run_once: true`:** Most tasks in this playbook use these because they are performing API calls to the AAP controller itself or manipulating global facts for the playbook run. This ensures they run on the controller node and only once.
*   **API Permissions:** The AAP user configured in the Controller Credential needs sufficient permissions to read workflow job details and job outputs.
*   **Regex Customization:** The regular expressions used for parsing stdout are designed for common Ansible error formats. If your jobs output errors in a highly custom way, these regexes might need adjustment.

---

This playbook provides a robust way to get detailed insights into your AAP workflow job failures.
