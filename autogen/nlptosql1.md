# NLP to Data Frame Operations

## Steps

### Code

- Import pandas and open ai libraries

```
import pandas as pd
from openai import AzureOpenAI
```

- Load the dataframe from Azure Data Lake

```
air_attributes = pd.read_parquet("azureml://subscriptions/xxxxxxx/resourcegroups/rg-name/workspaces/amlworkspacename/datastores/discovery_datalake/paths/directory/name/name/data_name.parquet")
display(air_attributes)
```

- list the open ai models

```
list(air_attributes.columns)
```

```
monthly_forecast_results = pd.read_parquet("azureml://subscriptions/xxxxxxx/resourcegroups/rg-name/workspaces/amlworkspacename/datastores/discovery_datalake/paths/directory/name/name/data_name.parquet")
display(monthly_forecast_results)
```

```
project_metadata = pd.read_parquet("azureml://subscriptions/xxxxxxx/resourcegroups/rg-name/workspaces/amlworkspacename/datastores/discovery_datalake/paths/directory/name/name/data_name.parquet")
display(project_metadata)
```

```
application_audit = pd.read_parquet("azureml://subscriptions/xxxxxxx/resourcegroups/rg-name/workspaces/amlworkspacename/datastores/discovery_datalake/paths/directory/name/name/data_name.parquet")
display(application_audit)
```

```
gcp_project = pd.read_parquet("azureml://subscriptions/xxxxxxx/resourcegroups/rg-name/workspaces/amlworkspacename/datastores/discovery_datalake/paths/directory/name/name/data_name.parquet")
display(gcp_project)
```

```
gcp_project_audit_npd = pd.read_parquet("azureml://subscriptions/xxxxxxx/resourcegroups/rg-name/workspaces/amlworkspacename/datastores/discovery_datalake/paths/directory/name/name/data_name.parquet")
display(gcp_project_audit_npd)
```

- Prompt text

```
prompttext = f"""You are a Expert in python pandas dataframe.
Here are the Dataframe names and columns provided. Based on the dataframe names and columns, Analyze the question and build the 
appropriate Dataframe based operations using python pandas.
Dataframe and Columns:

Dataframe Name: air_attributes
Columns: {list(air_attributes.columns)}
Description: information about application's air id details.

Dataframe Name: monthly_forecast_results
Columns: {list(monthly_forecast_results.columns)}
Description: gcp project has gcp_project_id and forecast is available in this data source.

Dataframe Name: project_metadata
Columns: {list(project_metadata.columns)}
Description: this data source has meta data for gcp_project data source with gcp_project_number and name as common columns.

Dataframe Name: application_audit
Columns: {list(application_audit.columns)}
Description: application audit details for gcp projects, here is the application_name and air_id which should be common across datasources.

Dataframe Name: gcp_project
Columns: {list(gcp_project.columns)}
Description: Has the information about Google application details, 
environments and also current status with contact information and has gcp_project_id as project id.
Air id is the key used for each application which is unique.

Dataframe Name: gcp_project_audit_npd
Columns: {list(gcp_project_audit_npd.columns)}
Description: Has the information about audit details of Google application, 
Based on the projects details available, audit records are created.

Sample user request:
Get the application_name, business_unit, and data_classification of all applications with application_status 'Active' as a pandas DataFrame.

Sample response:
air_attributes.query("application_status == 'Active'")[['application_name','business_unit','data_classification']]

please do not explain the code. Do not add documentation.
Please create the python dataframe operations for the user requests. Use the dataframe names provided above.
If we don't know please response as "unable to create query"
"""
```

- show the prompt

```
print(prompttext)
```

- Assign the questions

```
query = "Summarize top 10 application used from gcp projects?"
```

- Set the message packet

```
message = [
    {"role": "system", "content": f"{prompttext}"},
    {"role": "user", "content": f"{query}"}
]
```

- import

```
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
```

- set the identity

```
# 2. Initialize credentials
credential = DefaultAzureCredential()

# 3. Create token provider
token_provider = get_bearer_token_provider(
    credential, 
    "https://cognitiveservices.azure.com/.default"
)
```

- setup the model

```
client = AzureOpenAI(
  azure_endpoint = "https://cs-oai-6534301d-openai01d.openai.azure.com/openai/deployments/o4-mini/chat/completions?api-version=2025-01-01-preview", 
  azure_ad_token_provider=token_provider,
  api_version = "2025-01-01-preview",
)

model_name = "o4-mini"
```

- here is the run

```
model_name_reasoning = "o4-mini"
    # model_name_reasoning = "o3"

response = client.chat.completions.create(
    model=model_name_reasoning,
    #reasoning={"effort": "high"},
    reasoning_effort="high",
    messages=message,
    # temperature=0.7,
    max_completion_tokens=4000
)
```

- Print the response

```
print(response.choices[0].message.content)
```

- extract the code

```
import re
# Extract content inside triple backticks
match = re.search(r"```(?:python)?\n(.*?)```", response.choices[0].message.content, re.DOTALL)

if match:
    code_only = match.group(1).strip()
    print(code_only)
else:
    print("No code block found.")
```

- execute the code

```
# Execute the code block
exec(code_only)
```

- Now we can run the code and get the results

```
python app.py
```

- check the output
- done.