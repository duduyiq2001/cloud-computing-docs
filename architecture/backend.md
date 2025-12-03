# Backend Architecture

## Rails Application Structure

```mermaid
graph TB
    subgraph "Client Layer"
        Browser[Web Browser]
    end

    subgraph "Rails Application"
        subgraph "Controllers"
            EC[EventPostsController]
            ERC[EventRegistrationsController]
            UC[UsersController]
            LC[LeaderboardController]
            AC[Admin::*Controllers]
        end

        subgraph "Models"
            User[User<br/>- email, name, e_score<br/>- role: student/club_admin/super_admin]
            EventPost[EventPost<br/>- name, event_time, capacity<br/>- location, requires_approval]
            EventReg[EventRegistration<br/>- status: pending/confirmed/waitlisted<br/>- attendance_confirmed]
            EventCat[EventCategory<br/>- name, color]
            Notif[Notification<br/>- type, sent_at]
            AuditLog[AdminAuditLog<br/>- action, target, metadata]
        end

        subgraph "Background Jobs"
            ENJ[EnrollmentNotificationJob]
            ADJ[AdminDeletionJob]
        end

        subgraph "Mailers"
            ENM[EventNotificationMailer<br/>- enrollment_confirmation<br/>- waitlist_confirmed]
        end

        subgraph "Authentication"
            Devise[Devise<br/>- Email confirmation<br/>- Password reset]
        end
    end

    subgraph "External Services"
        PG[(PostgreSQL<br/>RDS)]
        SendGrid[SendGrid SMTP]
        Datadog[Datadog APM]
    end

    Browser --> EC & ERC & UC & LC & AC
    EC --> EventPost
    ERC --> EventReg
    UC --> User
    LC --> User
    AC --> AuditLog

    User -->|organizes| EventPost
    User -->|has_many| EventReg
    EventPost -->|belongs_to| EventCat
    EventPost -->|has_many| EventReg
    EventReg -->|has_one| Notif

    EventReg -->|after_create| ENJ
    ENJ --> ENM
    ENM --> SendGrid
    AC -->|async delete| ADJ

    User & EventPost & EventReg --> PG
    EC & ERC --> Datadog
```

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Framework | Ruby on Rails 8.1 |
| Database | PostgreSQL (AWS RDS) |
| Authentication | Devise (email confirmation required) |
| Frontend | Turbo, Stimulus, Importmap |
| Asset Pipeline | Propshaft |
| Web Server | Puma |
| Async Jobs | Rails ActiveJob (async adapter) |
| Email | SendGrid SMTP |
| APM/Monitoring | Datadog |
| Testing | RSpec + FactoryBot |

---

## Database Entity Relationship Diagram

```mermaid
erDiagram
    User ||--o{ EventPost : "organizes"
    User ||--o{ EventRegistration : "registers"
    EventPost ||--o{ EventRegistration : "has"
    EventPost }o--|| EventCategory : "belongs_to"
    EventRegistration ||--o| Notification : "triggers"
    User ||--o{ AdminAuditLog : "performs"

    User {
        bigint id PK
        string email UK
        string name
        string phone_number
        integer e_score
        integer role
        datetime confirmed_at
        string encrypted_password
    }

    EventPost {
        bigint id PK
        string name
        text description
        datetime event_time
        integer capacity
        string location_name
        float latitude
        float longitude
        boolean requires_approval
        integer registrations_count
        bigint organizer_id FK
        bigint event_category_id FK
    }

    EventRegistration {
        bigint id PK
        integer status
        datetime registered_at
        boolean attendance_confirmed
        bigint user_id FK
        bigint event_post_id FK
    }

    EventCategory {
        bigint id PK
        string name UK
        string color
    }

    Notification {
        bigint id PK
        integer notification_type
        datetime sent_at
        bigint user_id FK
        bigint event_registration_id FK
    }

    AdminAuditLog {
        bigint id PK
        string action
        string target_type
        bigint target_id
        jsonb metadata
        string ip_address
        bigint admin_user_id FK
    }
```

## Key Design Patterns

### Counter Cache
- `EventPost.registrations_count` only counts **confirmed** registrations
- Used for performance optimization in event listings

### Waitlist Management
- Auto-waitlist if event is full on registration
- Auto-promote oldest waitlisted person when confirmed spot opens
- Unless event requires manual approval

### E-Score Gamification
- +10 points awarded when organizer confirms attendance
- Leaderboard ranks top 10 by E-score descending

### Admin Audit Trail
- Every admin deletion logged before async execution
- Captures IP address, user agent, deletion reason, cascade impact
