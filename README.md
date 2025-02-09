# LLM Applications

An end-to-end guide for scaling and serving LLM application in production.

This repo currently contains one such application: a retrieval-augmented generation (RAG)
app for answering questions about supplied information. By default, the app uses
the [Ray documentation](https://docs.ray.io/en/master/) as the source of information.
This app first [indexes](./app/index.py) the documentation in a vector database
and then uses an LLM to generate responses for questions that got augmented with
relevant info retrieved from the index.

## Setup

### Compute
- Start a new [Anyscale workspace on staging](https://console.anyscale-staging.com/o/anyscale-internal/workspaces)
  using an [`g3.8xlarge`](https://instances.vantage.sh/aws/ec2/g3.8xlarge) head node on an AWS cloud.
- Use the [`default_cluster_env_2.6.2_py39`](https://docs.anyscale.com/reference/base-images/ray-262/py39#ray-2-6-2-py39) cluster environment.

### Repository

First, clone this repository.

```bash
git clone https://github.com/ray-project/llm-applications.git .
```

### Environment

Then set up the environment correctly by specifying the values in your `.env` file,
and installing the dependencies:

```bash
cp ./envs/.env_template .envs
source .envs
pip install --user -r requirements.txt
pre-commit install
pre-commit autoupdate
```

### Data

Our data is already ready at `/efs/shared_storage/pcmoritz/docs.ray.io/en/master/`
(on Staging) but if you wanted to load it yourself, run this bash command:

```bash
bash scrape-docs.sh
```

### Vector DB

<details>
<summary>Local installation with brew on MacOS</summary>

```bash
brew install postgresql
brew install pgvector
psql -c "CREATE USER postgres WITH SUPERUSER;"
# pragma: allowlist nextline secret
psql -c "ALTER USER postgres with password 'postgres';"
psql -c "CREATE EXTENSION vector;"
psql -f migrations/initial.sql
python app/index.py create-index
```
</details>

```bash
bash setup-pgvector.sh
sudo -u postgres psql -f migrations/initial.sql
python app/index.py create-index
```

### Query
Just a sample and uses the current index that's been created.
```python
import json
from app.query import QueryAgent
query = "What is the default batch size for map_batches?"
system_content = "Your job is to answer a question using the additional context provided."
agent = QueryAgent(
    embedding_model="thenlper/gte-base",
    llm="meta-llama/Llama-2-7b-chat-hf",
    max_context_length=4096,
    system_content=system_content,
)
result = agent.get_response(query=query)
print(json.dumps(result, indent=2))
```

### Experiments

#### Generate responses

```bash
python app/main.py generate-responses \
    --system-content "Answer the {query} using the additional {context} provided."
```

#### Evaluate responses

```bash
python app/main.py evaluate-responses \
    --system-content """
    Your job is to rate the quality of our generated answer {generated_answer}
    given a query {query} and a reference answer {reference_answer}.
    Your score has to be between 1 and 5.
    You must return your response in a line with only the score.
    Do not return answers in any other format.
    On a separate line provide your reasoning for the score as well.
    """
```

### Dashboard
```bash
export APP_PORT=8501
echo https://$APP_PORT-port-$ANYSCALE_SESSION_DOMAIN
streamlit run dashboard/Home.py
```

### TODO
- [x] notebook cleanup
- [x] evaluator (ex. GPT4) response script
- [x] DB dump & load
- [ ] experiments (in order and fixing choices along the way)
    - Evaluator
        - [ ] GPT-4 best experiment
        - [ ] Llama-70b consistency with GPT4
    - [ ] OSS vs. Closed (gpt-3.5 vs. llama)
    - [ ] w/ and w/out context (value of RAG)
    - [ ] # of chunks to use in context
        - Does using more resources help/harm?
        - 1, 5, 10 will all fit in the smallest context length of 4K)
    - [ ] Chunking size/overlap
        - related to # of chunks + context length, but we'll treat as independent variable
    - [ ] Embedding (top 3 in leaderboard)
        - global leaderboard may not be your leaderboard (empirically validate)
    - Later
        - [ ] Commercial Assistant evaluation
        - [ ] Human Assistant evaluation
        - [ ] Data sources
    - Much later
        - [ ] Prompt
        - [ ] Prompt-tuning on query
        - [ ] Embedding vs. LLM for retrieval
- [ ] Ray Tune to tweak a subset of components
- [ ] CI/CD workflows
