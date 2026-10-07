# Finance and Daily Collection Management

## Project Case Study

A full-stack application for managing customers, loans, and daily collections, with a React web interface and a Flutter mobile app for field agents.

## Business Problem

Daily collection operations need consistent records of loans, payments, outstanding balances, and agent activity. Field agents also need to record collections when internet access is unavailable.

## Core Features

- Customer management
- Loan creation and lifecycle tracking
- Daily collection recording and outstanding balance tracking
- Role-based access and permission checks
- Dashboards, analytics, and reports
- CSV, Excel, and PDF exports
- Offline mobile collections with later synchronization

## Technology Stack

| Area | Technologies |
|---|---|
| Backend | Java 17, Spring Boot, Spring Security |
| Authentication | JWT, refresh tokens, BCrypt |
| Database | PostgreSQL |
| Persistence | Spring Data JPA, Hibernate |
| Database migrations | Flyway |
| Web frontend | React, TypeScript, Vite, Tailwind CSS |
| State and API calls | React Context, Axios |
| Forms and charts | React Hook Form, Recharts |
| Mobile | Flutter, Dart, Riverpod, Dio, SQLite |
| Backend testing | JUnit 5, Mockito, AssertJ |
| Build and deployment | Maven, shell and Windows batch scripts |

## Architecture

The React web application and Flutter mobile application communicate with a Spring Boot REST API. The backend uses service and repository layers to manage business rules and persist data in PostgreSQL.

The mobile application maintains a local collection queue for offline operation and synchronizes it when connectivity returns.

## Engineering Highlights

- JWT authentication with refresh-token handling
- Server-side role and permission checks
- Request rate limiting
- Versioned database migrations
- DTO mapping with MapStruct
- Service-layer transaction management
- Offline collection queue and synchronization
- Automated collection-service tests

## My Contribution

Handled requirements, application design, backend and frontend development, mobile functionality, testing, deployment, and server maintenance.

## About This Repository

This repository presents the project's business context, architecture, and engineering approach as a portfolio case study.
