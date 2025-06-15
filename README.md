## How to Download and Verify the Signed Artifact

This project publishes a signed and attested artifact to GitHub Container Registry (GHCR).  
Follow these steps to securely download and verify the artifact.

### Prerequisites

- [oras CLI](https://github.com/oras-project/oras) (for pulling artifacts)
- [cosign CLI](https://github.com/sigstore/cosign) (for verifying signatures and attestations)

### Step 1: Pull the Artifact

```bash
oras pull ghcr.io/marcdonovan/sigstore/myapp:latest -o .
```
This will save the artifact file (e.g., myapp.tar.gz) in your current directory.

Extract it with:
```bash
tar -xzf myapp.tar.gz
```

### Step 2: Verify the Signature
Run the following command to verify the artifact’s signature:
```bash
cosign verify ghcr.io/marcdonovan/sigstore/myapp:latest
```
This confirms the artifact was signed by a trusted GitHub Actions workflow using keyless signing.

### Step 3: Verify the Attestation
Check the build metadata and attestation for the artifact:
```bash
cosign verify-attestation ghcr.io/marcdonovan/sigstore/myapp:latest
```

### Step 4 (Optional): Inspect the Attestation Predicate
To view detailed build metadata, export the attestation predicate to a file:
```bash
cosign verify-attestation --output predicate.json ghcr.io/marcdonovan/sigstore/myapp:latest
cat predicate.json
```
The predicate contains information such as:
- Builder (GitHub actor)
- Commit SHA
- Repository name
- Workflow run ID and number
- Event name
- Build timestamp
- Runner OS

### Authentication (If Required)
```bash
oras login ghcr.io
```
Or login using a Personal Access Token (PAT):
```bash
echo $PAT | oras login ghcr.io -u <github-username> --password-stdin
```
Replace <github-username> and provide a valid GitHub PAT with appropriate permissions.

