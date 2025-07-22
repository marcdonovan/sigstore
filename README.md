# 🔐 Signing and Verifying Container Artifacts with GitHub Actions + Sigstore (Cosign)

This guide shows how to **sign** and **verify** an OCI artifact (e.g., `.tar.gz`) using GitHub Actions and [`cosign`](https://docs.sigstore.dev/cosign/overview), storing the signature in [GHCR](https://ghcr.io/).

---

## ✅ Requirements

* GitHub repository
* GitHub Actions enabled
* Personal Access Token (PAT) with `write:packages` and `read:packages` scopes
* [Cosign installed](https://docs.sigstore.dev/cosign/installation)
* `oras` CLI for pushing arbitrary files to OCI

---

## 📦 1. Setup: Build, Push, Sign, and Attest Workflow

Create `.github/workflows/sign-and-verify.yaml`:

```yaml
name: Build, Push, Sign, and Attest

on:
  workflow_call:
    inputs:
      artifact_name:
        required: true
        type: string
      artifact_path:
        required: true
        type: string
    secrets:
      PAT:
        required: true

jobs:
  sign-and-attest:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    env:
      IMAGE: ghcr.io/<your-username>/<repo-name>/${{ inputs.artifact_name }}:latest

    steps:
      - uses: actions/checkout@v4

      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: ${{ inputs.artifact_name }}

      - name: Install ORAS
        run: |
          curl -L -o oras.tar.gz https://github.com/oras-project/oras/releases/download/v1.2.3/oras_1.2.3_linux_amd64.tar.gz
          tar -xzf oras.tar.gz
          sudo mv oras /usr/local/bin/oras

      - name: Login to GHCR
        run: echo "${{ secrets.PAT }}" | oras login ghcr.io -u ${{ github.actor }} --password-stdin

      - name: Push artifact
        id: push
        run: |
          DIGEST=$(oras push "$IMAGE" ${{ inputs.artifact_path }}:application/gzip | tee /dev/stderr | grep -o 'sha256:[a-f0-9]\{64\}')
          echo "digest=$DIGEST" >> $GITHUB_OUTPUT

      - name: Set IMAGE_WITH_DIGEST
        run: echo "IMAGE_WITH_DIGEST=${IMAGE}@${{ steps.push.outputs.digest }}" >> $GITHUB_ENV

      - name: Install Cosign
        run: |
          curl -Lo cosign https://github.com/sigstore/cosign/releases/download/v2.2.0/cosign-linux-amd64
          chmod +x cosign
          sudo mv cosign /usr/local/bin/cosign

      - name: Sign
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: cosign sign --yes "$IMAGE_WITH_DIGEST"

      - name: Attest
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: |
          echo '{"build":"${{ github.workflow }}","commit":"${{ github.sha }}"}' > predicate.json
          cosign attest --yes --predicate predicate.json --type custom "$IMAGE_WITH_DIGEST"
```

---

## 🚀 2. Trigger the Signing Workflow

Create a workflow like this in `.github/workflows/build-and-call.yaml`:

```yaml
name: Build and Call Signing Workflow

on:
  workflow_dispatch:

jobs:
  build-artifact:
    runs-on: ubuntu-latest
    outputs:
      artifact_name: myapp
      artifact_path: myapp.tar.gz

    steps:
      - uses: actions/checkout@v4

      - name: Build artifact
        run: |
          echo "Hello world" > myapp.txt
          tar -czf myapp.tar.gz myapp.txt

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: myapp
          path: myapp.tar.gz

  call-sign-artifact:
    needs: build-artifact
    uses: <your-username>/<repo-name>/.github/workflows/sign-and-verify.yaml@main
    with:
      artifact_name: myapp
      artifact_path: myapp.tar.gz
    secrets:
      PAT: ${{ secrets.PAT }}
```

---

## 🔍 3. Verify Signature

Once the workflow runs and signs your image, run:

```sh
cosign verify \
  --certificate-identity "https://github.com/<your-username>/<repo-name>/.github/workflows/sign-and-verify.yaml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/<your-username>/<repo-name>/myapp@sha256:<digest>
```

Example output:

```
Verified OK
```

---

## ✅ Tips

* `oras` allows pushing non-container files to container registries like GHCR.
* Use `cosign sign` with OIDC/GitHub Actions for **keyless signing**.
* Signatures and attestations are stored alongside the artifact in the registry.

---
