# Good Prompts

Example prompts I have encountered or created that are good.

## Table of Contents

- [Answer Questions Against a CSV]()
- [Answer Questions Against a SQL Database]()
- [Answer Questions Against a Database with Functions]()

### Answer Questions Against a CSV

Includes prior knowledge exclusions, and some form of meta checking.

Used as a single prompt containing the question.

**Prompt**

```py
PROMPT_PREFIX = """
First set the pandas display options to show all the columns,
get the column names, then answer the question.
"""

PROMPT_SUFFIX = """
- **ALWAYS** before giving the Final Answer, try another method.
Then reflect on the answers of the two methods you did and ask yourself
if it answers correctly the original question.
If you are not sure, try another method.
- If the methods tried do not give the same result,reflect and
try again until you have two methods that have the same result.
- If you still cannot arrive to a consistent result, say that
you are not sure of the answer.
- If you are sure of the correct answer, create a beautiful
and thorough response using Markdown.
- **DO NOT MAKE UP AN ANSWER OR USE PRIOR KNOWLEDGE,
ONLY USE THE RESULTS OF THE CALCULATIONS YOU HAVE DONE**.
- **ALWAYS**, as part of your "Final Answer", explain how you got
to the answer on a section that starts with: "\n\nExplanation:\n".
In the explanation, mention the column names that you used to get
to the final answer.
"""

QUESTION = "How may patients were hospitalized during July 2020"
"in Texas, and nationwide as the total of all states?"
"Use the hospitalizedIncrease column"


FINAL_PROMPT = PROMPT_PREFIX + QUESTION + PROMPT_SUFFIX
```

[Source](https://www.deeplearning.ai/short-courses/building-your-own-database-agent/)

### Answer Questions Against a SQL Database

Includes how sql prompts should be strucutred providing examples.

**Prompt**

```py
AGENT_PREFIX = """
You are an agent designed to interact with a SQL database.
## Instructions:
- Given an input question, create a syntactically correct {dialect} query
to run, then look at the results of the query and return the answer.
- Unless the user specifies a specific number of examples they wish to
obtain, **ALWAYS** limit your query to at most {top_k} results.
- You can order the results by a relevant column to return the most
interesting examples in the database.
- Never query for all the columns from a specific table, only ask for
the relevant columns given the question.
- You have access to tools for interacting with the database.
- You MUST double check your query before executing it.If you get an error
while executing a query,rewrite the query and try again.
- DO NOT make any DML statements (INSERT, UPDATE, DELETE, DROP etc.)
to the database.
- DO NOT MAKE UP AN ANSWER OR USE PRIOR KNOWLEDGE, ONLY USE THE RESULTS
OF THE CALCULATIONS YOU HAVE DONE.
- Your response should be in Markdown. However, **when running  a SQL Query
in "Action Input", do not include the markdown backticks**.
Those are only for formatting the response, not for executing the command.
- ALWAYS, as part of your final answer, explain how you got to the answer
on a section that starts with: "Explanation:". Include the SQL query as
part of the explanation section.
- If the question does not seem related to the database, just return
"I don\'t know" as the answer.
- Only use the below tools. Only use the information returned by the
below tools to construct your query and final answer.
- Do not make up table names, only use the tables returned by any of the
tools below.

## Tools:

"""

AGENT_FORMAT_INSTRUCTIONS = """

## Use the following format:

Question: the input question you must answer.
Thought: you should always think about what to do.
Action: the action to take, should be one of [{tool_names}].
Action Input: the input to the action.
Observation: the result of the action.
... (this Thought/Action/Action Input/Observation can repeat N times)
Thought: I now know the final answer.
Final Answer: the final answer to the original input question.

Example of Final Answer:
<=== Beginning of example

Action: query_sql_db
Action Input:
SELECT TOP (10) [death]
FROM covidtracking
WHERE state = 'TX' AND date LIKE '2020%'

Observation:
[(27437.0,), (27088.0,), (26762.0,), (26521.0,), (26472.0,), (26421.0,), (26408.0,)]
Thought:I now know the final answer
Final Answer: There were 27437 people who died of covid in Texas in 2020.

Explanation:
I queried the `covidtracking` table for the `death` column where the state
is 'TX' and the date starts with '2020'. The query returned a list of tuples
with the number of deaths for each day in 2020. To answer the question,
I took the sum of all the deaths in the list, which is 27437.
I used the following query

`sql
SELECT [death] FROM covidtracking WHERE state = 'TX' AND date LIKE '2020%'"
`sql
===> End of Example

"""

QUESTION = """How may patients were hospitalized during October 2020
in New York, and nationwide as the total of all states?
Use the hospitalizedIncrease column
"""
```

[Source](https://www.deeplearning.ai/short-courses/building-your-own-database-agent/)

### Answer Questions Against a Database with Functions

An example of how to call functions for Agents and how best to format responses and the functions themselves

**Functions**

```py
def get_hospitalized_increase_for_state_on_date(state_abbr, specific_date):
    try:
        query = f"""
        SELECT date, hospitalizedIncrease
        FROM all_states_history
        WHERE state = '{state_abbr}' AND date = '{specific_date}';
        """
        query = text(query)

        with engine.connect() as connection:
            result = pd.read_sql_query(query, connection)
        if not result.empty:
            return result.to_dict('records')[0]
        else:
            return np.nan
    except Exception as e:
        print(e)
        return np.nan

def get_positive_cases_for_state_on_date(state_abbr, specific_date):
    try:
        query = f"""
        SELECT date, state, positiveIncrease AS positive_cases
        FROM all_states_history
        WHERE state = '{state_abbr}' AND date = '{specific_date}';
        """
        query = text(query)

        with engine.connect() as connection:
            result = pd.read_sql_query(query, connection)
        if not result.empty:
            return result.to_dict('records')[0]
        else:
            return np.nan
    except Exception as e:
        print(e)
        return np.nan
```

**Tools / Functions**

```py
tools_sql = [
    {
        "type": "function",
        "function": {
            "name": "get_hospitalized_increase_for_state_on_date",
            "description": """Retrieves the daily increase in
                              hospitalizations for a specific state
                              on a specific date.""",
            "parameters": {
                "type": "object",
                "properties": {
                    "state_abbr": {
                        "type": "string",
                        "description": """The abbreviation of the state
                                          (e.g., 'NY', 'CA')."""
                    },
                    "specific_date": {
                        "type": "string",
                        "description": """The specific date for
                                          the query in 'YYYY-MM-DD'
                                          format."""
                    }
                },
                "required": ["state_abbr", "specific_date"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_positive_cases_for_state_on_date",
            "description": """Retrieves the daily increase in
                              positive cases for a specific state
                              on a specific date.""",
            "parameters": {
                "type": "object",
                "properties": {
                    "state_abbr": {
                        "type": "string",
                        "description": """The abbreviation of the
                                          state (e.g., 'NY', 'CA')."""
                    },
                    "specific_date": {
                        "type": "string",
                        "description": """The specific date for the
                                          query in 'YYYY-MM-DD'
                                          format."""
                    }
                },
                "required": ["state_abbr", "specific_date"]
            }
        }
    }
]
```

**Handler Function**

```py
available_functions = {
    "get_positive_cases_for_state_on_date": get_positive_cases_for_state_on_date,
    "get_hospitalized_increase_for_state_on_date":get_hospitalized_increase_for_state_on_date
    }
messages.append(response_message)

for tool_call in tool_calls:
    function_name = tool_call.function.name
    function_to_call = available_functions[function_name]
    function_args = json.loads(tool_call.function.arguments)
    function_response = function_to_call(
        state_abbr=function_args.get("state_abbr"),
        specific_date=function_args.get("specific_date"),
    )
    messages.append(
        {
            "tool_call_id": tool_call.id,
            "role": "tool",
            "name": function_name,
            "content": str(function_response),
        }
    )
```

**Prompt**

```py
QUESTION = """
how many hospitalized people we had in Alaska the 2021-03-05?
"""
```

[Source](https://www.deeplearning.ai/short-courses/building-your-own-database-agent/)

### Transform Unstructured Text into Desired JSON Structure

Used to generate a custom JSON schema of architecural diagrams.

**Prompt**

```py
INPUT = """
This is an AWS-based data analytics pipeline for a retail company. The system integrates data from multiple sources, processes it, and stores it in a data warehouse for analysis. Here are the key components and their interactions:

Data Sources:
1. Internal PostgreSQL database containing customer information
2. External API providing real-time inventory data from suppliers
3. Daily sales reports in CSV format stored in an S3 bucket

AWS Components:
1. AWS Step Functions: Orchestrates the entire data pipeline
2. AWS Lambda functions: 
   - Lambda A: Extracts data from the PostgreSQL database
   - Lambda B: Calls the external API and processes the response
   - Lambda C: Processes CSV files from S3
   - Lambda D: Transforms and combines data from all sources
   - Lambda E: Loads processed data into Redshift
3. Amazon S3: Stores raw CSV files and serves as a staging area for processed data
4. Amazon Redshift: Acts as the central data warehouse
5. Amazon CloudWatch: Monitors the pipeline and triggers the Step Functions workflow daily

Data Flow:
1. CloudWatch triggers the Step Functions workflow daily at 1 AM
2. Step Functions invokes Lambda A, B, and C in parallel to extract data from different sources
3. Lambda A connects to the PostgreSQL database and extracts customer data
4. Lambda B calls the external API to fetch inventory data
5. Lambda C reads and parses the CSV files from S3
6. Once all extraction Lambdas complete, Step Functions triggers Lambda D
7. Lambda D combines and transforms the data from all sources
8. Transformed data is stored temporarily in S3
9. Step Functions then triggers Lambda E
10. Lambda E loads the processed data from S3 into Redshift
11. Upon completion, Step Functions sends a notification about the pipeline status

Access and Reporting:
- Data analysts access Redshift to run queries and generate reports
- A Tableau dashboard connects to Redshift for real-time visualizations
- The IT team has full access to monitor and manage all AWS services

This pipeline ensures that the latest data from all sources is available in the Redshift data warehouse every morning for analysis and reporting.
"""

PROMPT = f"""
Generate a JSON representation of a system architecture based on the following description:

{INPUT}

The JSON should strictly adhere to the following structure and guidelines:

1. Groups: Categorize components into logical groups (e.g., cloud providers, data sources, user types).
2. Components: List all system components with their details.
3. Connections: Describe how components are connected or interact.

Use the following schema:

{{
    "groups": [
        {{
            "name": "group_name",
            "type": "group_type" // e.g., "cloud_provider", "data_source", "user_group", etc.
        }}
    ],
    "components": [
        {{
            "name": "component_name",
            "type": "component_type",
            "group": "group_name",
            "image": "image_filename.png"
        }}
    ],
    "connections": [
        {{
            "from": "component_name1",
            "to": "component_name2",
            "label": "connection_description"
        }}
    ]
}}

Image Naming Guidelines:
1. For specific cloud services, use "[provider]-[service].png" (e.g., "aws-lambda.png", "azure-sql-database.png").
2. For generic services, use descriptive names (e.g., "database.png", "api.png", "server.png").
3. For user roles, use "user.png" or specific roles like "admin.png", "analyst.png".
4. If unsure, use a generic term related to the component type.

Example:
Given the description: "A web application using AWS. It has a React frontend hosted on S3, an API Gateway connecting to Lambda functions, and a DynamoDB database. CloudFront is used as a CDN."

The JSON output should be:

{{
    "groups": [
        {{
            "name": "AWS",
            "type": "cloud_provider"
        }},
        {{
            "name": "Frontend",
            "type": "client_side"
        }}
    ],
    "components": [
        {{
            "name": "React App",
            "type": "frontend_framework",
            "group": "Frontend",
            "image": "react.png"
        }},
        {{
            "name": "S3 Bucket",
            "type": "object_storage",
            "group": "AWS",
            "image": "aws-s3.png"
        }},
        {{
            "name": "CloudFront",
            "type": "cdn",
            "group": "AWS",
            "image": "aws-cloudfront.png"
        }},
        {{
            "name": "API Gateway",
            "type": "api_management",
            "group": "AWS",
            "image": "aws-api-gateway.png"
        }},
        {{
            "name": "Lambda",
            "type": "serverless_function",
            "group": "AWS",
            "image": "aws-lambda.png"
        }},
        {{
            "name": "DynamoDB",
            "type": "nosql_database",
            "group": "AWS",
            "image": "aws-dynamodb.png"
        }}
    ],
    "connections": [
        {{
            "from": "React App",
            "to": "CloudFront",
            "label": "user access"
        }},
        {{
            "from": "CloudFront",
            "to": "S3 Bucket",
            "label": "origin"
        }},
        {{
            "from": "React App",
            "to": "API Gateway",
            "label": "API calls"
        }},
        {{
            "from": "API Gateway",
            "to": "Lambda",
            "label": "triggers"
        }},
        {{
            "from": "Lambda",
            "to": "DynamoDB",
            "label": "read/write"
        }}
    ]
}}

Ensure your output is a valid JSON object containing only the requested structure.
Do not include any explanatory text or markdown formatting.
"""

```

[Source](https://github.com/connor-john/ai-solution-architect/tree/main)