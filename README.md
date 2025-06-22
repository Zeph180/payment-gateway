
# 📘 Payment Gateway – API Documentation & Deployment Guide

## 📖 Table of Contents

 - [🔹 API Endpoints](#-api-endpoints)
  - [📘 POST /api/payments](#post-apipayments)
    - [✅ Request](#-request)
    - [📥 Response](#-response)
    - [📊 Status Codes](#-status-codes)
  - [📘 GET /api/payments/{transactionReference}](#get-apipaymentstransactionreference)
    - [✅ Route Parameter](#-route-parameter)
    - [📥 Response](#-response-1)
    - [📊 Transaction Status](#-transaction-status)
    - [📊 Status Codes](#-status-codes-1)

- [🚀 Deployment Guide](#-deployment-guide)
  - [✅ Pre-requisites](#-pre-requisites)
  - [🏗️ Step 1: Clone the Repository](#️-step-1-clone-the-repository)
  - [🧱 Step 2: Configure the Database](#-step-2-configure-the-database)
    - [🔹 Create Database](#-create-database)
    - [🔹 Stored Procedures](#-stored-procedures)
    - [🔹 Update appsettingsjson](#-update-appsettingsjson)
    - [🔹 Apply Migrations](#-apply-migrations)
  - [⚙️ Step 3: Build & Run Locally](#️-step-3-build--run-locally)
  - [🖥️ Step 4: Deploy to Production](#️-step-4-deploy-to-production)
    - [🚩 Option A: IIS](#-option-a-iis)
    - [🐳 Option B: Docker](#-option-b-docker)
      - [Dockerfile Snippet](#dockerfile-snippet)
      - [Build/Run Commands](#buildrun-commands)
  - [🔐 Step 5: Secure the API](#-step-5-secure-the-api)

- [🏗️ Clean Architecture](#️-clean-architecture)
  - [🔧 Layers](#-layers)
    - [Domain](#domain)
    - [Application](#application)
    - [Infrastructure](#infrastructure)
    - [API (Presentation)](#api-presentation)
  - [🔄 Data Flow](#-data-flow)

- [📦 Project Structure](#-project-structure)

- [✅ Benefits](#-benefits)

- [🧪 Testing Strategy](#-testing-strategy)

- [📱 Mobile Client (Ionic Angular)](#-mobile-client-ionic-angular)

- [📘 Further Improvements](#-further-improvements)


---

## 🔹 POST `/initiate`

Initiates a payment transaction.

### ✅ Request

**Content-Type**: `application/json`

```json
{
  "payer": "0123456789",
  "payee": "9876543210",
  "amount": 1000.00,
  "currency": "UGX",
  "payerReference": "INV-20240508"
}
```

### 📥 Response

```json
{
  "transactionReference": "a4f1cd34-5ef9-4c38-9b87-12f98d7cfaba",
  "statusCode": 200,
  "message": "Transaction successfully processed"
}
```

### 📊 Status Codes

- `200` – Transaction Successful  
- `100` – Transaction Pending  
- `400` – Transaction Failed (e.g., insufficient funds)

---

## 🔹 GET `/status/{transactionReference}`

Retrieves the status of a transaction using its reference.

### ✅ Route Parameter

- `transactionReference`: `Guid` – Unique transaction ID

### 📥 Response

```json
{
  "transactionReference": "a4f1cd34-5ef9-4c38-9b87-12f98d7cfaba",
  "status": "SUCCESSFUL",
  "createdAt": "2025-05-08T11:30:22Z"
}
```

### 📊 Status

- `PENDING`
- `SUCCESSFUL`
- `FAILED`

### 📊 Status Codes

- `200` – Found  
- `404` – Not Found

---

# 🚀 Deployment Guide – DFCU Payment Gateway

## ✅ Pre-requisites

- [.NET 8 SDK](https://dotnet.microsoft.com/en-us/download)
- SQL Server 2019 or higher
- IIS / Docker / Azure App Service
- Git
- SMTP or Notification Service (optional)

---

## 🏗️ Step 1: Clone the Repository

```bash
git clone https://github.com/Zeph180/dfcu-payment-gateway.git
cd Dfcu
```

---

## 🧱 Step 2: Configure the Database

### 🔹 a. Create Database

```sql
CREATE DATABASE DfcuPayments;
```


### 🔹 b. Apply Migrations

```bash
cd Dfcu.PaymentGateway.Infrastructure
dotnet ef database update
```

### 🔹 c. Create stored procedures

```sql
CREATE PROCEDURE sp_AddTransaction
  @Id UNIQUEIDENTIFIER,
    @Payer NVARCHAR(100),
    @Payee NVARCHAR(100),
    @Amount DECIMAL(18,2),
    @Currency NVARCHAR(10),
    @PaymentReference NVARCHAR(50),
    @Status NVARCHAR(20),
    @CreatedAt DATETIME
AS
BEGIN
INSERT INTO Transactions
(Id, Payer, Payee, Amount, Currency, PaymentReference, Status, CreatedAt)
VALUES
  (@Id, @Payer, @Payee, @Amount, @Currency, @PaymentReference, @Status, @CreatedAt)
END

CREATE PROCEDURE sp_GetTransactionById
  @Id UNIQUEIDENTIFIER
AS
BEGIN
SELECT TOP 1
        *
FROM Transactions
WHERE Id = @Id
END
```

### 🔹 d. Update `appsettings.json` in `Dfcu.PaymentGateway.Api`

```json
{
  "ConnectionStrings": {
     "DefaultConnection": "Server=YOUR_SQL_SERVER;Database=PaymentDb;User Id=YOUR_USER_ID;Password=YOUR_PASSWORD;Encrypt=false;TrustServerCertificate=True;"
  }
}
```
---

## ⚙️ Step 3: Build & Run Locally

```bash
dotnet build
dotnet run --project Dfcu.PaymentGateway.Api
```

API runs at:

```
https://localhost:5087
```

---

## 🖥️ Step 4: Deploy to Production

### 🚩 Option A: Using IIS

1. Publish the API:

   ```bash
   dotnet publish -c Release -o ./publish
   ```

2. Copy `./publish` to IIS Server
3. Set up an IIS site pointing to that folder
4. Set environment variable:

   ```bash
   ASPNETCORE_ENVIRONMENT=Production
   ```

---

### 🐳 Option B: Using Docker

#### Dockerfile

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
USER $APP_UID
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["PaymentGateway.Api/PaymentGateway.Api.csproj", "PaymentGateway.Api/"]
COPY ["PaymentGateway.Application/PaymentGateway.Application.csproj", "PaymentGateway.Application/"]
COPY ["PaymentGateway.Domain/PaymentGateway.Domain.csproj", "PaymentGateway.Domain/"]
COPY ["PaymentGateway.Infrastructure/PaymentGateway.Infrastructure.csproj", "PaymentGateway.Infrastructure/"]
RUN dotnet restore "PaymentGateway.Api/PaymentGateway.Api.csproj"
COPY . .
WORKDIR "/src/PaymentGateway.Api"
RUN dotnet build "PaymentGateway.Api.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "PaymentGateway.Api.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "PaymentGateway.Api.dll"]

```

#### Build & Run

```bash
docker build -t dfcu/payment-gateway .
docker run -d -p 8080:80 dfcu/payment-gateway
```

---

### 🔐 Step 5: Secure the API

```dockerfile
Enforce HTTPS

Use API Key Authentication or JWT Tokens

Log all transactions and errors securely

Rotate credentials regularly
```

## Payment Gateway - Clean Architecture Overview

## 🏗️ Architecture Overview

This project implements **Clean Architecture** principles to create a modular, maintainable, and testable Payments Gateway API using **ASP.NET Core** with **SQL Server** as the database and **Ionic Angular** for the mobile client.

---

## 🔧 Clean Architecture Layers

### 1. **Domain Layer** (`PaymentsGateway.Domain`)
- **Purpose**: Represents the core business logic and rules.
- **Components**:
  - Entities (e.g., `PaymentTransaction`)
  - Interfaces (e.g., `IPaymentService`, `IPaymentRepository`)
- **No dependencies** on other layers.

---

### 2. **Application Layer** (`PaymentsGateway.Application`)
- **Purpose**: Contains business use cases and application logic.
- **Components**:
  - Use Cases (e.g., `InitiatePayment`, `CheckPaymentStatus`)
  - DTOs and ViewModels
  - Interfaces from Domain Layer
- **Depends on** Domain Layer.

---

### 3. **Infrastructure Layer** (`PaymentsGateway.Infrastructure`)
- **Purpose**: Handles interaction with external systems like databases and APIs.
- **Components**:
  - EF Core implementations
  - Repository implementations
  - Database context (`PaymentDbContext`)
- **Depends on** Application and Domain Layers.

---

### 4. **API Layer (Presentation)** (`PaymentsGateway.API`)
- **Purpose**: Entry point for HTTP requests.
- **Components**:
  - Controllers (e.g., `PaymentsController`)
  - API configuration and startup logic
- **Depends on** Application Layer.

---

## 🔄 Data Flow Example

1. API Layer receives a payment request via HTTP.
2. API Layer invokes the relevant Use Case in the Application Layer.
3. Application Layer calls the Domain Service logic and uses interfaces defined in the Domain Layer.
4. Infrastructure Layer implements these interfaces and interacts with the database.
5. Response bubbles back up to the API Layer and to the client.

---

## 📦 Project Structure

```
GATEWAY/
├── PaymentsGateway.API/           # ASP.NET Core Web API (Presentation Layer)
├── PaymentsGateway.Application/   # Application Layer (Use Cases)
├── PaymentsGateway.Domain/        # Domain Layer (Core Business Logic)
├── PaymentsGateway.Infrastructure/ # Infrastructure Layer (EF Core, DB Access)
```

---

## ✅ Benefits of This Architecture

- **Separation of Concerns**: Each layer has a clear responsibility.
- **Testability**: Domain and Application layers can be unit tested independently.
- **Maintainability**: Easy to extend, refactor, or replace components (e.g., switch from MSSQL to another DB).
- **Scalability**: Logical separation allows services to scale independently if needed.

---

## 🧪 Testing Strategy

- **Unit Tests**: Focus on Domain and Application Layers.
- **Integration Tests**: Validate data access and external system integration in Infrastructure.
- **End-to-End Tests**: Verify complete user flows via the API and mobile app.

---

## 📱 Mobile Client (Ionic Angular)

The mobile client interacts with the API via HTTP:
- Initiates new payments
- Checks payment status
- Shows transaction feedback

🔗 The test client is available here [CLIENT APP](https://github.com/Zeph180/dfcu-payment-gateway.git)

---

## 📘 Further Improvements

- Add authentication & authorization
- Implement caching for status checks
- Integrate with a real core banking system in place of mock responses
---


