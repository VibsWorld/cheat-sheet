

### Command Description
`az account show`

- Retrieves details about the **currently signed‑in Azure account**.  
- The output typically includes:
  - **User information** (username/email)
  - **Subscription ID and name**
  - **Tenant ID**
  - **Cloud environment** (e.g., AzureCloud)  

👉 Useful for verifying which account and subscription context you’re operating in before running other `az` CLI commands.

`az devops project list --org https://dev.azure.com/ParcelVision/ --output table`

- **`az devops project list`** → Lists all Azure DevOps projects within a specified organization.  
- **`--org https://dev.azure.com/ParcelVision/`** → Targets the organization `ParcelVision` hosted on Azure DevOps.  
- **`--output table`** → Formats the output as a readable table, showing project names, IDs, and other metadata in a structured layout.  

👉 This command is useful for quickly viewing all projects under the `ParcelVision` organization in a clean tabular format.

### Command Description
`az pipelines list --output table`

- **`az pipelines list`** → Lists all Azure Pipelines available in the current Azure DevOps project context.  
- **`--output table`** → Formats the results into a clean, tabular view for easier readability.  

👉 This command is typically used to quickly see all pipelines (with their names, IDs, and other metadata) in a structured table format.

### Command Description
`az pipelines runs list --pipeline-ids 60 --top 5 --output table`

- **`az pipelines runs list`** → Lists the execution history (runs) of a specified Azure Pipeline.  
- **`--pipeline-ids 60`** → Filters results to show runs only for the pipeline with ID `60`.  
- **`--top 5`** → Limits the output to the most recent 5 runs.  
- **`--output table`** → Formats the results into a readable table view.  

👉 This command is useful for quickly checking the latest 5 runs of pipeline ID `60` in a clean tabular format.
