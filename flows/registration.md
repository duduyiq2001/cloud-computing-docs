# Event Registration Flow

## Sequence Diagram

```mermaid
sequenceDiagram
    participant U as User Browser
    participant ALB as AWS ALB
    participant Rails as Rails Pod
    participant PG as PostgreSQL
    participant Job as Background Job
    participant SG as SendGrid

    U->>ALB: POST /event_posts/:id/event_registrations
    ALB->>Rails: Forward request (TLS terminated)
    Rails->>Rails: Authenticate (Devise)
    Rails->>PG: Check event capacity
    PG-->>Rails: Event data

    alt Event Full
        Rails->>PG: Create registration (status: waitlisted)
    else Event Requires Approval
        Rails->>PG: Create registration (status: pending)
    else Event Open
        Rails->>PG: Create registration (status: confirmed)
        Rails->>PG: Increment registrations_count
    end

    PG-->>Rails: Registration created

    Rails->>Job: EnrollmentNotificationJob.perform_later
    Rails-->>ALB: 302 Redirect to event page
    ALB-->>U: Response

    Note over Job,SG: Async (background)
    Job->>PG: Load registration + event + user
    Job->>SG: Send confirmation email
    SG-->>U: Email delivered
```

## Registration Status Logic

| Condition | Status | Email Sent? |
|-----------|--------|-------------|
| Event has space, no approval required | `confirmed` | Yes - enrollment confirmation |
| Event has space, requires approval | `pending` | No - wait for organizer |
| Event is full | `waitlisted` | Yes - waitlist notification |

## Counter Cache Behavior

- Only **confirmed** registrations are counted
- `EventPost.registrations_count` is incremented/decremented automatically
- Used to determine if event is "full"

## Code Reference

```ruby
# app/models/event_registration.rb
def set_initial_status
  if event_post.full?
    self.status = :waitlisted
  elsif event_post.requires_approval?
    self.status = :pending
  else
    self.status = :confirmed
  end
end
```
