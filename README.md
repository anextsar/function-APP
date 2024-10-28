# function-APP

This code defines an Azure Function app that uses Azure AI, Azure OpenAI, and Azure Cognitive Search to search and summarize document content based on user queries. Below is an explanation of each part of the code:

```python
import azure.functions as func
import logging
import json
import os
from azure.core.credentials import AzureKeyCredential
from openai import AzureOpenAI
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizedQuery
```

- **Imports**: 
    - `azure.functions` provides tools for building Azure Functions.
    - `logging` is used to log function processing details.
    - `json` helps with JSON data formatting.
    - `os` allows access to environment variables.
    - `AzureKeyCredential` handles credentials for Azure services.
    - `AzureOpenAI` from the `openai` package integrates Azure OpenAI models.
    - `SearchClient` is used for Azure Cognitive Search functionality.
    - `VectorizedQuery` enables vector search in Azure Cognitive Search.

```python
# Fetch environment variables
SEARCH_ENDPOINT = os.getenv("SEARCH_ENDPOINT")
SEARCH_KEY = os.getenv("SEARCH_KEY")
AZURE_OPENAI_ENDPOINT = os.getenv("AZURE_OPENAI_ENDPOINT")
AZURE_OPENAI_KEY = os.getenv("AZURE_OPENAI_KEY")
EMBEDDING_MODEL_NAME = os.getenv("EMBEDDING_MODEL_NAME")
AZURE_OPENAI_API_VERSION = os.getenv("AZURE_OPENAI_API_VERSION")
CHAT_COMPLETION_MODEL_NAME = os.getenv("CHAT_COMPLETION_MODEL_NAME")
```

- **Environment Variables**:
    - These lines fetch the necessary credentials and configurations for Azure services, such as endpoints, API keys, and model names from the environment variables.

```python
app = func.FunctionApp(http_auth_level=func.AuthLevel.ANONYMOUS)
```

- **Function App**:
    - Creates an Azure Function app named `app`, allowing anonymous access by setting `http_auth_level=func.AuthLevel.ANONYMOUS`.

```python
@app.route(route="http_trigger")
def http_trigger(req: func.HttpRequest) -> func.HttpResponse:
    logging.info('Python HTTP trigger function processed a request.')
```

- **HTTP Trigger Setup**:
    - Defines an HTTP trigger function named `http_trigger` that listens for requests at the `http_trigger` route.
    - Logs a message when the function processes a request.

```python
    openai_client = AzureOpenAI(
        azure_endpoint=AZURE_OPENAI_ENDPOINT,
        api_key=AZURE_OPENAI_KEY,
        api_version=AZURE_OPENAI_API_VERSION
    )
```

- **OpenAI Client**:
    - Initializes the `openai_client` using Azure OpenAI configurations for embeddings and chat completions.

```python
    try:
        req_body = req.get_json()
    except ValueError:
        return func.HttpResponse("Invalid JSON", status_code=400)
```

- **Request Parsing**:
    - Tries to parse the request body as JSON and stores it in `req_body`. If parsing fails, it returns a `400` status response with an error message.

```python
    query = req_body.get('query')
    Department = req_body.get('department')
    index_name = "vaibhav-index"
```

- **Extract Request Data**:
    - Retrieves `query` and `department` fields from the JSON request.
    - Sets `index_name` to the predefined name of the search index, `vaibhav-index`.

```python
    if req.method != 'POST' or not (query and Department):
        return func.HttpResponse(
            body=json.dumps({"status": "FAILED", "error": "ONLY POST IS ALLOWED "}),
            status_code=200,
            mimetype="application/json"
        )
```

- **Method and Parameter Validation**:
    - Checks if the request is a `POST` request and if `query` and `Department` are provided. If not, it responds with an error message and status `200`.

```python
    try:
        search_client = SearchClient(
            endpoint=SEARCH_ENDPOINT,
            index_name=index_name,
            credential=AzureKeyCredential(SEARCH_KEY)
        )
```

- **Search Client Initialization**:
    - Creates a `SearchClient` object with the configured endpoint, index, and credentials to connect to Azure Cognitive Search.

```python
        embedding_response = openai_client.embeddings.create(
            model=EMBEDDING_MODEL_NAME,
            input=query
        )
        embedding = embedding_response.data[0].embedding
```

- **Embedding Creation**:
    - Uses `openai_client` to generate an embedding for the `query` text based on the specified `EMBEDDING_MODEL_NAME`. Stores the embedding vector in `embedding`.

```python
        vector_query = VectorizedQuery(
            vector=embedding,
            k_nearest_neighbors=10,
            fields="content_vector"
        )
```

- **Vectorized Query Setup**:
    - Sets up a `VectorizedQuery` using the embedding vector for similarity search, asking for the 10 nearest neighbors in the `content_vector` field.

```python
        search_filter = None
        if Department != "ALL":
            search_filter = f"Department eq '{Department}'"
```

- **Search Filter**:
    - Defines a search filter based on `Department`. If `Department` is set to "ALL", no filter is applied.

```python
        results = search_client.search(
            search_text=None,
            vector_queries=[vector_query],
            select=["content", "file_download_url", "file_name", "Department"],
            filter=search_filter
        )
```

- **Document Search**:
    - Executes a search on `search_client` using the vector query and filter. The `select` parameter limits the returned fields to `content`, `file_download_url`, `file_name`, and `Department`.

```python
        search_results = [{"content": result["content"], "file_download_url": result["file_download_url"], "file_name": result["file_name"]} for result in results]
```

- **Format Search Results**:
    - Formats search results by extracting only the specified fields and stores them in `search_results`.

```python
        if not search_results:
            return func.HttpResponse(
                body=json.dumps({"response": None, "error": None}),
                status_code=200,
                mimetype="application/json"
            )
```

- **No Results Handling**:
    - Returns a JSON response with no results if `search_results` is empty.

```python
        answer = "\n".join(result["content"] for result in search_results)
        prompt = f"""
            You are an assistant, follow the instructions carefully and strictly give the response based on the context: {answer}.
            ...
        """
```

- **Response Generation Prompt**:
    - Concatenates the content from `search_results` to form `answer` and creates a detailed prompt instructing the assistant to generate a response based on the content.

```python
        response = openai_client.chat.completions.create(
            model=CHAT_COMPLETION_MODEL_NAME,
            temperature=0,
            max_tokens=200,
            messages=[{"role": "system", "content": prompt},
                      {"role": "user", "content": query}]
        )
```

- **Chat Completion Request**:
    - Sends the `prompt` and `query` to `openai_client` to generate a chat completion response based on `CHAT_COMPLETION_MODEL_NAME`.

```python
        response_text = response.choices[0].message.content
```

- **Extract Response Text**:
    - Extracts the content of the first choice from `response`, storing it as `response_text`.

```python
        if not response or "##Not Found##" in response_text:
            return func.HttpResponse(
                body=json.dumps({"response": None, "error": None}),
                status_code=200,
                mimetype="application/json"
            )
```

- **Handle Not Found Responses**:
    - Returns a response with no results if `response` is empty or if "##Not Found##" is in `response_text`.

```python
        files = []
        summaries = []
```

- **Initialize Results Lists**:
    - Initializes lists to store file information (`files`) and summaries (`summaries`).

```python
        for result in search_results:
            ...
            summary_response = openai_client.chat.completions.create(
                model=CHAT_COMPLETION_MODEL_NAME,
                messages=[{"role": "user", "content": f"Please summarize the following content: {content}"}],
                temperature=0,
                max_tokens=50
            )
            ...
```

- **Generate Summaries**:
    - For each result, sends the `content` to `openai_client` to create a short summary, appending the summary and file details to `files`.

```python
        logging.info("Response content: %s", response_text)
```

- **Log Response**:
    - Logs the `response_text` content.

```python
        return func.HttpResponse(
            body=json.dumps({
                "response": response_text,
                "documents": files
            }),
            status_code=200,
            mimetype="application/json"
        )
```

- **Return Final Response**:
    - Returns a JSON response with the assistant’s `response_text` and the summarized document details in `files`.

```python
    except Exception as e:
        return func.HttpResponse(
            body=json.dumps({"response": None, "error": str(e)}),
            status_code=200,
            mimetype="application/json"
        )
```

- **Exception Handling**:
    - Catches and logs errors, returning a JSON response with the error details if an exception occurs.
