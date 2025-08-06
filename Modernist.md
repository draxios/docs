# Continuous Code-Modernisation with AWS Bedrock & Azure DevOps CLI

> **Purpose**  
> Modernise any Git repository—locally, from the command line—by looping AWS Bedrock over its contents, applying incremental PR-quality patches until either  
> (a) a target time limit is hit or (b) the model declares the work “DONE”.

---

## 0. Prerequisites (once per workstation)

| Tool | Reason | Install |
|------|--------|---------|
| **Python ≥ 3.11** | run the CLI | `pyenv install 3.11.9` / system pkg |
| **AWS CLI v2** | creds / region | `brew install awscli` |
| **Git** | local repo ops | already installed with dev tooling |
| **Terraform ≥ 1.6** | infra module | `brew install terraform` |
| **jq** (optional) | JSON prettifier | `brew install jq` |

Create an [AWS named profile] `bedrock-moderniser` with at least **`bedrock-agent:*`** and **`bedrock-knowledge-base:*`** plus the OpenSearch-Serverless permissions shown later.

Generate an **Azure DevOps PAT** with _Code (read/write)_ scope and store it in your OS key-chain or **`~/.config/moderniser/.env`**:

```env
AZDO_PAT=xxxxxxxxxxxxxxxxxxxxx
````

---

## 1. High-Level Flow

```
┌───────────┐    clone / pull   ┌────────────────┐
│   Dev PC  │ ───────────────▶ │  Local   Repo  │
└───────────┘                   │   (Git)        │
        ▲                       └────┬───────────┘
        │ open-source patch           │ iterate:
        │                              ▼
┌────────────┐   Bedrock     ┌──────────────────┐
│ Knowledge  │◀──────────────│  Moderniser CLI  │
│   Base     │  embeddings   │  (Python)        │
└────────────┘     ▲         └──────────────────┘
        │   RAG    │                    ▲
        └──────────┴─ Bedrock Invoke ───┘
```

1. **Ingest** – push repo files into a Bedrock Knowledge Base (KB) backed by OpenSearch-Serverless.
2. **Loop** – for each pass:

   1. Ask the model (Titan, Claude, Llama 3, etc.) for *diff-style* changes that satisfy the user’s modernisation prompts.
   2. Apply the diff with `git apply`, commit, and push to Azure DevOps.
   3. Re-ingest the changed file(s) so the KB stays fresh.
3. Stop when max-minutes reached **or** model replies `#COMPLETE`.

---

## 2. One-Time AWS Infrastructure (re-usable)

### 2.1 Terraform module snippet

```hcl
# main.tf
module "bedrock_moderniser" {
  source  = "aws-ia/bedrock/aws"         # open-source module
  version = ">= 0.4.0"

  create_default_kb = true               # provisions KB + even Titan embed
  kb_vector_store_type = "OPENSEARCH_SERVERLESS"

  opensearch_collection_name = "moderniser-os"
  opensearch_encryption_policy = "aws/opensearch"
  opensearch_network_policy   = ["0.0.0.0/0"]

  # IAM role used later by the CLI
  kb_ingestion_role_name = "kb-ingestion-role"
}
```

Key resource references:

* `aws_bedrockagent_knowledge_base` – Terraform provider resource ([Terraform Registry][1])
* `AgentsforBedrock.Client.ingest_knowledge_base_documents` – direct ingestion API ([Boto3][2])
* Direct-ingestion user-guide page ([AWS Documentation][3])

```bash
terraform init
terraform apply -auto-approve
# Output: knowledge_base_id, role_arn, opensearch_endpoints, ...
```

### 2.2 IAM extras

```jsonc
{
  "Effect":"Allow",
  "Action":[
    "bedrock-agent:InvokeAgent",
    "bedrock-agent:IngestKnowledgeBaseDocuments",
    "bedrock-agent:Retrieve",
    "bedrock-agent:ListKnowledgeBases"
  ],
  "Resource":"*"
}
```

Add the OpenSearch-Serverless data-access policy that trusts the Bedrock KB role. ([Amazon Web Services, Inc.][4])

---

## 3. Local Project Scaffold

```text
moderniser/
├─ cli.py
├─ prompts/
│   ├─ system.md
│   └─ user_default.md
├─ requirements.txt
└─ utils/
   ├─ git_utils.py
   └─ kb_utils.py
```

### requirements.txt

```
boto3>=1.34
awscli>=2.15
GitPython>=3.1
azure-devops>=7.1
python-dotenv>=1.0
rich>=13
```

---

## 4. CLI Implementation (✂ copy-paste)

`cli.py`

````python
#!/usr/bin/env python3
import argparse, os, pathlib, subprocess, textwrap, time, uuid, sys, json
from datetime import datetime, timedelta
from dotenv import load_dotenv
from rich import print

load_dotenv()

### ---- Bedrock Client Helpers ------------------------------------------- ###
import boto3
kb_id   = os.getenv("KB_ID")           # export once per workstation
agent   = boto3.client("bedrock-agent", profile_name="bedrock-moderniser")
bedrock = boto3.client("bedrock-runtime", profile_name="bedrock-moderniser")

def chunk_file(path: pathlib.Path):
    """Very naive splitter; improve with tiktoken or trafilatura for HTML."""
    with open(path, "r", errors="ignore") as f:
        data = f.read()
    max_chars = 4_000
    for i in range(0, len(data), max_chars):
        yield data[i:i+max_chars]

def ingest_paths(paths: list[pathlib.Path]):
    docs = []
    for p in paths:
        for part in chunk_file(p):
            docs.append({
                "dataSourceType":"CUSTOM",
                "document": part,
                "metadata":{"file_path":str(p)}
            })
    print(f"Ingesting {len(docs)} KB chunks …")
    agent.ingest_knowledge_base_documents(
        knowledgeBaseId=kb_id,
        documents=docs
    )

### ---- Git / Azure DevOps utils ---------------------------------------- ###
from azure.devops.connection import Connection
from msrest.authentication import BasicAuthentication
from git import Repo, GitCommandError

def ensure_git(repo_path:str):
    if os.path.isdir(os.path.join(repo_path, ".git")):
        return Repo(repo_path)
    raise SystemExit("❌ Not a git repository")

AZDO_PAT = os.getenv("AZDO_PAT")
def azdo_push(repo:Repo, message:str):
    try:
        repo.git.push()
    except GitCommandError:
        # first push needing creds
        remote_url = repo.remotes.origin.url.replace("https://",
            f"https://{AZDO_PAT}@")
        repo.git.push(remote_url, "HEAD:refs/heads/moderniser")

### ---- Bedrock Modernisation Loop -------------------------------------- ###
SYSTEM_PROMPT = pathlib.Path("prompts/system.md").read_text()
USER_PROMPT   = pathlib.Path("prompts/user_default.md").read_text()

MODEL_ID = os.getenv("MODEL_ID", "anthropic.claude-3-sonnet-20240229-v1:0")

def modernise_once(repo_path):
    repo = ensure_git(repo_path)
    changed_files = [p for p in repo_path.rglob("*") if p.suffix in
        {".py",".cs",".java",".js",".ts",".go",".sql",".html",".css"}]

    ingest_paths(changed_files)

    request = {
        "system": SYSTEM_PROMPT,
        "messages": [
            {"role":"user",
             "content": USER_PROMPT + "\n### CODE CONTEXT IS INDEXED ###"}
        ],
        "knowledgeBaseId": kb_id,
        "temperature": 0.2
    }
    resp = bedrock.invoke_model(
        modelId=MODEL_ID,
        body=json.dumps(request).encode()
    )
    suggestion = json.loads(resp["body"].read())["content"]
    if "#COMPLETE" in suggestion:
        print("[green]Model signals completion[/green]")
        return False
    patch = suggestion.split("```diff")[1].split("```")[0]
    patch_file = repo_path/"tmp.patch"
    patch_file.write_text(patch)
    try:
        repo.git.apply(patch_file)
        repo.index.add(all=True)
        repo.index.commit(f"🪄 Moderniser pass @ {datetime.utcnow():%FT%TZ}")
        azdo_push(repo, "auto-modernise")
        ingest_paths([p.relative_to(repo_path) for p in repo.untracked_files])
        print("[cyan]Applied diff, committed, re-ingested[/cyan]")
    except GitCommandError as e:
        print(f"[red]Patch failed: {e}[/red]")
    finally:
        patch_file.unlink(missing_ok=True)
    return True

def main():
    ap = argparse.ArgumentParser(
        description="Loop Bedrock moderniser over a repo")
    ap.add_argument("repo", type=pathlib.Path,
                    help="local path to cloned repo")
    ap.add_argument("--max-minutes", type=int, default=15)
    args = ap.parse_args()

    end_t = datetime.utcnow() + timedelta(minutes=args.max_minutes)
    while datetime.utcnow() < end_t:
        if not modernise_once(args.repo):
            break
        time.sleep(3)  # polite pause
    print("[bold green]Modernisation loop done[/bold green]")

if __name__ == "__main__":
    main()
````

---

## 5. Prompts (*edit to suit project*)

`prompts/system.md`

````markdown
You are **ModerniserGPT**, a senior software architect.  
Your task: transform legacy code into modern, idiomatic, secure, and
cloud-native style while preserving business behaviour.

* Always respond with a valid unified diff (`diff --git …`) inside
  ```diff fenced blocks``` (no commentary outside).
* If no change is needed, reply solely with the token `#COMPLETE`.
* Follow each of these refactoring objectives in **priority order**:
  1. Security & secret handling (env-vars, AWS Secrets Manager).
  2. Dependency upgrades to latest LTS.
  3. Test & CI improvements (.github/workflows or azure-pipelines.yml).
  4. Containerisation hints (Dockerfile).
````

`prompts/user_default.md` – e.g. “Upgrade the project to .NET 8 minimal APIs…”

---

## 6. Usage

```bash
git clone https://dev.azure.com/bogware/sample-app sample-app
cd moderniser
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export KB_ID=kb-xxxxxxxx         # from Terraform output
export MODEL_ID=anthropic.claude-3-sonnet-20240229-v1:0
python cli.py ../sample-app --max-minutes 25
```

Commits land on branch **`moderniser`** in ADO where you can review & merge.

---

## 7. Retargeting to Another Project

1. `git clone` a **new** repo locally.
2. Update `prompts/user_default.md` with project-specific goals.
3. Run **`cli.py PATH --max-minutes N`** again – everything else re-uses the same KB, saving embedding cost.

---

## 8. Security & Cost Notes

* **KB Storage** – OpenSearch-Serverless charges per GB-ingested + RU.
* **Model Cost** – pick Titan or Llama 3 for cheaper passes; switch to Claude/Command R only for final polish.
* Store PAT in OS secret store; CLI never logs it.
* IAM least-privilege: Bedrock agent role only on the single KB ARN.
* Add CloudWatch log filters for “PII” strings before shipping to Bedrock.

---

## 9. Cleanup

```bash
# remove infra
terraform destroy
# delete local embeddings index & venv
rm -rf ~/.cache/moderniser .venv
```

---

## 10. References

* Terraform resource `aws_bedrockagent_knowledge_base` (HashiCorp AWS provider) ([Terraform Registry][1])
* Direct ingestion guide – “Ingest changes directly into a knowledge base” ([AWS Documentation][3])
* Boto3 `ingest_knowledge_base_documents` API ([Boto3][2])
* OpenSearch-Serverless Terraform best practice blog ([Amazon Web Services, Inc.][4])

```

---

### How to extend

* Swap `chunk_file` for an AST-aware splitter (Babel, Tree-sitter) for deeper context.  
* Replace diff-apply with the new **GitHub Copilot-style multi-file edit** once Bedrock adds native patch ops.  
* Wrap the CLI in **AWS SAM** and expose as a Lambda‐powered API for remote kicks.

Feel free to carve this page into separate wiki topics—**Infrastructure**, **CLI Usage**, **Prompt Engineering**, etc.—but the copy above is functional as-is. Happy modernising, Old Friend!
::contentReference[oaicite:8]{index=8}
```

[1]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/bedrockagent_knowledge_base?utm_source=chatgpt.com "aws_bedrockagent_knowledge_..."
[2]: https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-agent/client/ingest_knowledge_base_documents.html?utm_source=chatgpt.com "ingest_knowledge_base_docum..."
[3]: https://docs.aws.amazon.com/bedrock/latest/userguide/kb-direct-ingestion.html?utm_source=chatgpt.com "Ingest changes directly into a knowledge base - Amazon Bedrock"
[4]: https://aws.amazon.com/blogs/big-data/deploy-amazon-opensearch-serverless-with-terraform/?utm_source=chatgpt.com "Deploy Amazon OpenSearch Serverless with Terraform - AWS"
