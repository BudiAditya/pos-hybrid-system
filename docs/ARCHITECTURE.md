# POS Hybrid System Architecture

## System Overview
Modern Point-of-Sale system dengan real-time synchronization antara aplikasi desktop dan mobile.

### Technology Stack
- **Backend**: .NET 8, Entity Framework Core, SignalR dengan Redis Backplane
- **Database**: PostgreSQL (local & cloud), Redis untuk caching
- **Mobile**: Flutter 3.x (iOS & Android), Provider State Management
- **Desktop**: C# WinForms, Entity Framework, Local PostgreSQL
- **Infrastructure**: CentOS 9, Nginx, RabbitMQ untuk message queue
- **Payment**: Multi-gateway (QRIS, Credit Card, E-wallets)
- **Hardware**: Barcode scanner, receipt printer, customer display

## System Architecture Diagram

```mermaid
graph TB
    subgraph "Store Level"
        A[WinForms Desktop App] --> B[PostgreSQL Local]
        A --> C[Hardware Integration]
        C --> D[Barcode Scanner]
        C --> E[Receipt Printer]
        C --> F[Customer Display]
    end
    
    subgraph "Cloud VPS"
        G[.NET 8 API] --> H[PostgreSQL Cloud]
        G --> I[SignalR Hub]
        I --> J[Redis Backplane]
        G --> K[Payment Gateway]
        K --> L[QRIS Service]
        K --> M[Credit Card Gateway]
        K --> N[E-Wallet APIs]
    end
    
    subgraph "Mobile Apps"
        O[Flutter iOS] --> G
        O --> I
        P[Flutter Android] --> G
        P --> I
    end
    
    A --> G
    A --> I
