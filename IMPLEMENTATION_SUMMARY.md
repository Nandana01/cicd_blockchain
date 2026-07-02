# Implementation Summary

## Project Overview

A blockchain-integrated artifact integrity verification system for CI/CD pipelines. The project implements a Flask-based web application secured by an Ethereum smart contract that immutably stores SHA-256 hashes of Docker build artifacts, ensuring tamper detection and audit transparency before deployment.

---

## Architecture

```
pipeline.ps1 (PowerShell Orchestrator)
       |
       +---> Docker Build (docker build / docker save)
       |
       +---> verify_artifact.py (Pre-check against blockchain)
       |
       +---> deploy_contract.py (Register new hash if authorized)
       |
       +---> docker-compose up (Deploy container)
       |
       +---> audit_app.py (Immutable Audit Dashboard)
```

---

## Components

### 1. Smart Contract (`ArtifactIntegrity.sol`)

Solidity contract deployed on Ethereum (Ganache local node) that stores artifact records on-chain.

- **Struct**: `ArtifactRecord` with fields: `artifactId`, `artifactHash`, `timestamp`, `signer`, `stage`
- **Functions**:
  - `storeArtifact()` - Write a new artifact hash record to the blockchain
  - `getArtifact()` - Retrieve a stored artifact record by ID
  - `verifyArtifact()` - Compare a provided hash against the stored on-chain hash
- **Event**: `ArtifactStored` emitted on every write operation for audit logging

### 2. Contract Deployment (`deploy_contract.py`)

CLI tool that compiles and deploys the Solidity contract, then registers artifact hashes.

- Installs Solidity compiler (solcx v0.8.17)
- Compiles `ArtifactIntegrity.sol` and deploys to Ethereum node
- Computes SHA-256 hash of the artifact file
- Calls `storeArtifact()` to register the hash on-chain
- Verifies the record was stored correctly
- Persists contract address and ABI to JSON files

**Usage:**
```
python deploy_contract.py --artifact-path note-app.tar --artifact-id notes-app-v1 --signer pipeline --stage build
```

### 3. Verification Gateway (`verify_artifact.py`)

CLI tool that acts as a deployment gate by verifying artifact integrity against blockchain records.

- Computes SHA-256 of the local artifact file
- Calls `verifyArtifact()` on the smart contract to compare hashes
- Returns exit code 0 (pass) or 1 (fail)
- Displays stored record details on verification result

**Usage:**
```
python verify_artifact.py --artifact-path note-app.tar --artifact-id notes-app-v1
```

### 4. CI/CD Pipeline (`pipeline.ps1`)

PowerShell script orchestrating the full build-verify-deploy cycle in up to 6 steps:

1. **Docker Build** - Builds the container image from Dockerfile
2. **Artifact Export** - Saves image as `.tar` and copies to `artifacts/` for history
3. **Blockchain Pre-check** - Verifies new artifact hash against on-chain records
4. **Authorization Gate** - If hash is new/unregistered, prompts administrator for Ethereum private key to sign and register
5. **Container Deployment** - Starts the container via `docker-compose up -d`
6. **Git Push** (optional) - Commits and pushes code to GitHub

**Modes:**
- Full pipeline (default)
- `-SkipBuild` - Reuse existing tar, skip Docker build
- `-VerifyOnly` - Only verify artifact against blockchain

### 5. Audit Dashboard (`audit_app.py` + `audit_templates/audit.html`)

Flask web application (port 5001) serving as an immutable audit ledger and forensic gateway.

**API Endpoints:**
- `/` - Dashboard frontend with system integrity status
- `/api/blockchain-records` - Fetches all `ArtifactStored` events from the smart contract
- `/api/local-files` - Scans `artifacts/` directory, computes SHA-256 of each `.tar` file, cross-references with blockchain records to determine verified/tampered status
- `/api/current-status` - Checks integrity of the live `note-app.tar` against blockchain records

**Dashboard Views:**
- Dashboard Overview - Summary statistics (blockchain registry count, local artifact count, integrity state)
- Blockchain Ledger - Table of all smart contract event logs
- Forensic Auditor - Cross-checks local `.tar` files against blockchain hashes

### 6. Application Layer (`app.py`)

Simple Flask CRUD notes application used as the target workload for the CI/CD pipeline.

- Routes: `/` (list), `/add` (create), `/edit/<id>`, `/update/<id>`, `/delete/<id>`
- Database: SQLite (`notes.db`)
- Containerized via Docker and deployed on port 5000

---

## Technologies

| Component | Technology |
|-----------|------------|
| Blockchain | Ethereum (Ganache local node) |
| Smart Contract | Solidity 0.8.17 |
| Contract Interaction | Web3.py |
| Hashing | SHA-256 |
| CI/CD Pipeline | PowerShell |
| Application | Flask + SQLite |
| Containerization | Docker + Docker Compose |
| Audit Dashboard | Flask + Jinja2 + JavaScript |

---

## Project Structure

```
Full_code/
+-- ArtifactIntegrity.sol          # Solidity smart contract
+-- deploy_contract.py             # Contract deployment and hash registration
+-- verify_artifact.py             # Artifact verification gateway
+-- pipeline.ps1                   # CI/CD pipeline orchestrator
+-- audit_app.py                   # Audit dashboard Flask app
+-- audit_templates/audit.html     # Audit dashboard frontend
+-- app.py                         # Notes application
+-- templates/index.html           # Notes app main view
+-- templates/edit.html            # Notes app edit view
+-- contract_abi.json              # Compiled contract ABI
+-- contract_address.json          # Deployed contract address
+-- docker-compose.yml             # Container orchestration
+-- Dockerfile                     # Container image definition
+-- requirements.txt               # Python dependencies
+-- notes.db                       # SQLite database
+-- note-app.tar                   # Current build artifact
+-- artifacts/                     # Historical build artifacts
+-- test.py                        # Ganache connection test
```

---

## Workflow

1. Developer runs `pipeline.ps1`
2. Pipeline builds Docker image and exports as `.tar`
3. Pipeline runs `verify_artifact.py` to check if artifact hash exists on blockchain
4. **If hash matches**: Deployment authorized, proceed to step 5
5. **If hash does not match**: Administrator prompted for Ethereum private key
   - Valid key: Hash registered on blockchain via `deploy_contract.py`, deployment proceeds
   - Invalid/empty key: Deployment blocked, pipeline aborted
6. Container deployed via `docker-compose up -d`
7. Audit dashboard on port 5001 displays blockchain records and artifact integrity status

---

## Known Limitations

- **Blockchain Platform**: Uses Ethereum (Ganache) instead of Hyperledger Fabric (permissioned)
- **No Centralized Baseline**: No traditional logging system implemented for comparative evaluation
- **No Experimental Framework**: No automated attack simulation, performance measurement, or statistical analysis
- **Public Contract**: `storeArtifact()` has no access control; any address can write records
- **Incomplete Dependencies**: `requirements.txt` missing `web3` and `py-solc-x`
- **No CI/CD Integration**: Pipeline is PowerShell-based, no Jenkins or GitHub Actions configuration
- **No Documentation**: No README, setup instructions, or integration guidelines
