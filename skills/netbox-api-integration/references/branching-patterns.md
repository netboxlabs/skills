# Branching Patterns

Reference for the [netbox-branching](https://github.com/netboxlabs/netbox-branching) plugin API: lifecycle, context headers, and async operations.

> **Plugin Required:** All patterns here require the netbox-branching plugin.

## Branch Lifecycle

Create → Wait (PROVISIONING → READY) → Work → Sync (optional) → Merge

### States

| State | Description | Allowed Operations |
|-------|-------------|-------------------|
| NEW | Just created | None |
| PROVISIONING | Schema being created | None |
| READY | Ready for use | Read, write, sync, merge, revert |
| SYNCING | Pulling from main | Read only |
| MERGING | Applying to main | Read only |
| REVERTING | Rolling back | Read only |
| MERGED | Complete | Read only (historical) |
| ARCHIVED | Inactive | None |

## Complete Workflow

```python
import requests, time

NETBOX_URL = "https://netbox.example.com"
HEADERS = {"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"}

def wait_for_branch_ready(branch_id, timeout=120):
    start = time.time()
    while time.time() - start < timeout:
        branch = requests.get(
            f"{NETBOX_URL}/api/plugins/branching/branches/{branch_id}/",
            headers=HEADERS).json()
        if branch["status"]["value"] == "ready":
            return branch
        if branch["status"]["value"] in ("archived", "merged"):
            raise RuntimeError(f"Branch in terminal state: {branch['status']['value']}")
        time.sleep(2)
    raise TimeoutError("Branch did not become ready")

def wait_for_job(job_url, timeout=300):
    start = time.time()
    while time.time() - start < timeout:
        job = requests.get(job_url, headers=HEADERS).json()
        if job["status"]["value"] == "completed":
            return job
        if job["status"]["value"] in ("errored", "failed"):
            raise RuntimeError(f"Job failed: {job.get('data', {}).get('error', 'Unknown')}")
        time.sleep(2)
    raise TimeoutError("Job did not complete")

# Step 1: Create branch
branch = requests.post(
    f"{NETBOX_URL}/api/plugins/branching/branches/",
    headers=HEADERS,
    json={"name": "feature-network-updates", "description": "Q1 updates"}
).json()
schema_id = branch["schema_id"]  # 8-char ID for X-NetBox-Branch header

# Step 2: Wait for READY
wait_for_branch_ready(branch["id"])

# Step 3: Work in branch context
branch_headers = {**HEADERS, "X-NetBox-Branch": schema_id}
requests.post(f"{NETBOX_URL}/api/dcim/devices/", headers=branch_headers,
    json={"name": "new-switch-01", "site": 1, "device_type": 1, "role": 1, "status": "planned"})

# Step 4: Sync (optional)
sync_job = requests.post(
    f"{NETBOX_URL}/api/plugins/branching/branches/{branch['id']}/sync/",
    headers=HEADERS, json={"commit": True}).json()
wait_for_job(sync_job["url"])

# Step 5: Merge
merge_job = requests.post(
    f"{NETBOX_URL}/api/plugins/branching/branches/{branch['id']}/merge/",
    headers=HEADERS, json={"commit": True}).json()
wait_for_job(merge_job["url"])
```

## Context Header

Switch between main and branch using `X-NetBox-Branch`:

```
X-NetBox-Branch: {schema_id}
```

**Use the 8-char `schema_id`**, not the branch name or numeric ID:

```python
branch = {"id": 42, "name": "feature-x", "schema_id": "a1b2c3d4"}

# CORRECT
headers = {**HEADERS, "X-NetBox-Branch": branch["schema_id"]}

# WRONG — these will fail
headers = {**HEADERS, "X-NetBox-Branch": str(branch["id"])}     # numeric ID
headers = {**HEADERS, "X-NetBox-Branch": branch["name"]}         # name
```

Without the header: operations target main. With the header: operations target the branch schema.

### Session Wrapper

```python
class BranchSession:
    def __init__(self, netbox_url, token, schema_id):
        self.netbox_url = netbox_url.rstrip("/")
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/json",
            "X-NetBox-Branch": schema_id,
        })

    def get(self, endpoint, **kwargs):
        return self.session.get(f"{self.netbox_url}/api/{endpoint.lstrip('/')}", **kwargs)

    def post(self, endpoint, **kwargs):
        return self.session.post(f"{self.netbox_url}/api/{endpoint.lstrip('/')}", **kwargs)

    def patch(self, endpoint, **kwargs):
        return self.session.patch(f"{self.netbox_url}/api/{endpoint.lstrip('/')}", **kwargs)

    def delete(self, endpoint, **kwargs):
        return self.session.delete(f"{self.netbox_url}/api/{endpoint.lstrip('/')}", **kwargs)
```

### Comparing Branch and Main

```python
def compare_branch_main(endpoint, params, schema_id):
    main = requests.get(f"{NETBOX_URL}/api/{endpoint}/", headers=HEADERS, params=params).json()["results"]
    branch = requests.get(f"{NETBOX_URL}/api/{endpoint}/",
        headers={**HEADERS, "X-NetBox-Branch": schema_id}, params=params).json()["results"]
    main_ids = {r["id"] for r in main}
    branch_ids = {r["id"] for r in branch}
    return {"added": branch_ids - main_ids, "deleted": main_ids - branch_ids, "both": main_ids & branch_ids}
```

### GraphQL with Branch Context

The header works with GraphQL too:

```python
headers = {"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json",
           "X-NetBox-Branch": schema_id}
requests.post(f"{NETBOX_URL}/graphql/", headers=headers, json={"query": query, "variables": vars})
```

## Async Operations

Sync, merge, and revert return **Job objects** — they don't complete immediately. Always poll.

### Job Status Values

| Status | Terminal? |
|--------|-----------|
| pending | No |
| scheduled | No |
| running | No |
| completed | Yes ✓ |
| errored | Yes ✗ |
| failed | Yes ✗ |

### Dry-Run Validation

Always validate before committing:

```python
# Dry-run (commit: false)
dry_job = requests.post(
    f"{NETBOX_URL}/api/plugins/branching/branches/{branch_id}/merge/",
    headers=HEADERS, json={"commit": False}).json()
job = wait_for_job(dry_job["url"])

if job.get("data", {}).get("conflicts"):
    print(f"Conflicts: {job['data']['conflicts']}")
else:
    # Actual merge
    merge_job = requests.post(
        f"{NETBOX_URL}/api/plugins/branching/branches/{branch_id}/merge/",
        headers=HEADERS, json={"commit": True}).json()
    wait_for_job(merge_job["url"])
```

### Async with Callbacks

```python
import threading

def poll_job_async(job_url, on_complete, on_error, timeout=300):
    def poll():
        try:
            result = wait_for_job(job_url, timeout)
            on_complete(result)
        except Exception as e:
            on_error(e)
    thread = threading.Thread(target=poll, daemon=True)
    thread.start()
    return thread
```

### Job Progress Monitoring

```python
def poll_with_progress(job_url, timeout=300):
    start, last_status = time.time(), None
    while time.time() - start < timeout:
        job = requests.get(job_url, headers=HEADERS).json()
        status = job["status"]["value"]
        if status != last_status:
            print(f"[{int(time.time()-start)}s] {status}")
            last_status = status
        if status == "completed":
            return job
        if status in ("errored", "failed"):
            raise RuntimeError(f"Failed: {job.get('data', {}).get('error')}")
        time.sleep(2)
    raise TimeoutError(f"Timed out after {timeout}s")
```

### Review Changes Before Merge

```python
diff = requests.get(
    f"{NETBOX_URL}/api/plugins/branching/branches/{branch_id}/diff/",
    headers=HEADERS).json()
print(f"Changes to merge: {len(diff)} modifications")
```

## References

- [NetBox Branching Plugin](https://github.com/netboxlabs/netbox-branching)
- [NetBox Branching Documentation](https://netboxlabs.com/docs/netbox-branching/)
- [NetBox Jobs Framework](https://netboxlabs.com/docs/netbox/en/stable/features/background-jobs/)
