# WireMock Service Walkthrough

## Overview
This service mocks third-party APIs (Digio, Hunter, Cashfree, NSDL) to allow QA testing without hitting production endpoints.

## Prerequisites
- Docker and Docker Compose installed.

## How to Run
1. Navigate to the `wiremock-service` directory:
   ```bash
   cd wiremock-service
   ```
2. Start the service (if not already running):
   ```bash
   docker-compose up -d
   ```
3. The mock service will be available at `http://localhost:8080`.

## Verified Endpoints & Responses

### Digio KYC
**Request:**
```bash
curl -X POST http://localhost:8080/digio/kyc
```
**Response:**
```json
{"status":"success","message":"KYC Verified Successfully","reference_id":"MOCK_DIGIO_12345"}
```

### Hunter Fraud Check
**Request:**
```bash
curl -X POST http://localhost:8080/hunter/check
```
**Response:**
```json
{"status":"approved","score":0,"message":"No Fraud Detected","transaction_id":"MOCK_HUNTER_67890"}
```

### Cashfree Disbursement
**Request:**
```bash
curl -X POST http://localhost:8080/cashfree/disburse
```
**Response:**
```json
{"status":"SUCCESS","message":"Disbursement Successful","referenceId":"MOCK_CASHFREE_54321","amount":50000}
```

### NSDL PAN Check
**Request:**
```bash
curl -X POST http://localhost:8080/nsdl/pan
```
**Response:**
```json
{"status":"E","message":"Existing and Valid","name":"MOCK USER NAME","pan":"ABCDE1234F"}
```

## Admin Interface
Access the WireMock admin interface at:
`http://localhost:8080/__admin`
