# Using Mem0-GaussDB with GaussDB

This repository is Huawei Cloud's GaussDB-enabled Mem0 distribution. It adds the `gaussdb` vector-store provider to Mem0 so that agents can store, search, update, delete, and filter long-term memories in GaussDB.

> [!IMPORTANT]
> `pip install mem0ai` installs the upstream Mem0 package from PyPI. It does **not** include the GaussDB provider in this repository. Install this repository instead, as described below.

## Prerequisites

- Python 3.10 or later
- A GaussDB instance with its vector database capability enabled (`enable_vectordb`), plus `FLOATVECTOR`, `JSONB`, and `UUID`
- An application account with table, index, and read/write privileges
- A GaussDB-compatible `psycopg2` driver
- An embedding model service. The full `Memory` API additionally needs an LLM when `infer=True`.

The integration targets GaussDB O-compatible deployments. Use UTF-8 for the database and client connection. UTF-8 is especially important for Chinese `text_lemmatized` values and centralized BM25 keyword search.

## Install the GaussDB-enabled package

Create an isolated environment, clone this repository, and install it from the checked-out source:

```bash
git clone https://github.com/huaweicloud-samples/database-mem0-GaussDB.git
cd database-mem0-GaussDB

python -m venv .venv
# Linux and macOS
source .venv/bin/activate
# Windows PowerShell: .venv\Scripts\Activate.ps1

python -m pip install --upgrade pip
python -m pip install -e .
python -m pip install psycopg2-binary
python -c "from mem0.vector_stores.gaussdb import GaussDB; print('GaussDB provider is available')"
```

For a non-editable installation directly from GitHub:

```bash
python -m pip install "git+https://github.com/huaweicloud-samples/database-mem0-GaussDB.git"
python -m pip install psycopg2-binary
```

`psycopg2-binary` is convenient for local development. For production, use the `psycopg2` package or GaussDB-provided driver build that matches the target instance and its TLS requirements.

## Quick start with the Memory API

The example below uses an OpenAI-compatible embedding and LLM endpoint. Replace the model names, API endpoint, and dimensions with values supplied by your model service. `embedding_model_dims` must equal the embedding model's output dimension.

```python
import os

from mem0 import Memory

config = {
    "embedder": {
        "provider": "openai",
        "config": {
            "model": "your-embedding-model",
            "api_key": os.environ["OPENAI_COMPATIBLE_API_KEY"],
            "openai_base_url": os.environ["OPENAI_COMPATIBLE_BASE_URL"],
            "embedding_dims": 1024,
        },
    },
    "llm": {
        "provider": "openai",
        "config": {
            "model": "your-chat-model",
            "api_key": os.environ["OPENAI_COMPATIBLE_API_KEY"],
            "openai_base_url": os.environ["OPENAI_COMPATIBLE_BASE_URL"],
        },
    },
    "vector_store": {
        "provider": "gaussdb",
        "config": {
            "connection_string": os.environ["GAUSSDB_CONNECTION_STRING"],
            "collection_name": "mem0_memory",
            "embedding_model_dims": 1024,
            "deployment_mode": "centralized",
            "vector_index_type": "gsdiskann",
            "vector_metric": "cosine",
            "auto_create": True,
        },
    },
}

memory = Memory.from_config(config)
memory.add(
    "I like coffee and usually go hiking on weekends.",
    user_id="user_001",
    infer=False,
)

results = memory.search("coffee", filters={"user_id": "user_001"})
print(results)
```

Set the connection string and model service variables outside your source code. For example:

```bash
export GAUSSDB_CONNECTION_STRING="postgresql://gaussdb_user:gaussdb_password@db.example.com:19995/postgres"
export OPENAI_COMPATIBLE_API_KEY="your-api-key"
export OPENAI_COMPATIBLE_BASE_URL="https://your-openai-compatible-endpoint/v1"
```

For Windows PowerShell, use `$env:GAUSSDB_CONNECTION_STRING = "..."` and the equivalent `$env:` assignments.

## Use the provider directly

Use the provider API when the application already creates embeddings itself. Vectors and payloads have matching positions; custom IDs, when supplied, must be UUID strings.

```python
from mem0.vector_stores.gaussdb import GaussDB

db = GaussDB(
    connection_string="postgresql://gaussdb_user:gaussdb_password@db.example.com:19995/postgres",
    collection_name="mem0_provider_demo",
    embedding_model_dims=4,
    deployment_mode="centralized",
    vector_index_type="gsdiskann",
    vector_metric="cosine",
)

db.insert(
    vectors=[[1.0, 0.0, 0.0, 0.0]],
    payloads=[{
        "data": "I like coffee",
        "text_lemmatized": "I like coffee",
        "user_id": "user_001",
    }],
)

rows = db.search(
    query="coffee",
    vectors=[1.0, 0.0, 0.0, 0.0],
    top_k=5,
    filters={"user_id": "user_001"},
)
print(rows)
```

Do not put real passwords or API keys in source code. Prefer `GAUSSDB_CONNECTION_STRING` or the individual `GAUSSDB_*` environment variables described in the [detailed configuration reference](docs/components/vectordbs/dbs/gaussdb.mdx).

## Deployment checklist

| Capability | Centralized GaussDB | Distributed GaussDB |
| --- | --- | --- |
| `Memory.add`, `search`, `update`, and `delete` | Supported | Supported |
| Vector search and metadata filtering | Supported | Supported |
| Vector dimensions | Up to 4096 | Up to 1024 |
| Vector index | `gsdiskann` or `gsivfflat`; dimensions above 1024 require `gsdiskann` | `gsdiskann` or `gsivfflat`; dimensions must not exceed 1024 |
| BM25 `keyword_search` | Supported when BM25 is available and its index is usable | Not supported |
| Payload storage | `JSONB` | `JSONB` |

`auto_create` defaults to `True`: it creates a missing collection and its indexes. With `auto_create=False`, the provider only validates an existing collection and its vector index; it never creates or repairs them. Use this mode when schema changes are controlled by a DBA or migration workflow.

## More configuration and behavior details

The [GaussDB vector-store reference](docs/components/vectordbs/dbs/gaussdb.mdx) documents all connection environment variables, index settings, filter semantics, score conversion, BM25 limitations, and `auto_create` behavior.

Before production use, pin this repository to an approved release tag or commit in your dependency management workflow rather than installing an unreviewed moving branch.
