# Waitlist Promotion Flow

## Automatic Promotion Sequence

```mermaid
sequenceDiagram
    participant U1 as Confirmed User
    participant Rails as Rails App
    participant ER as EventRegistration
    participant EP as EventPost
    participant U2 as Waitlisted User
    participant Job as EnrollmentNotificationJob
    participant Mail as SendGrid

    U1->>Rails: DELETE /event_registrations/:id (cancel)
    Rails->>ER: Destroy confirmed registration

    ER->>ER: after_destroy :promote_potential_candidate
    ER->>EP: Check requires_approval?

    alt Event requires approval
        Note over ER: No auto-promotion<br/>Organizer must manually approve
    else Event open enrollment
        ER->>ER: Find oldest waitlisted registration
        ER->>ER: promote_from_waitlist!
        Note over ER: status: waitlisted → confirmed
        ER->>EP: Increment registrations_count
        ER->>Job: send_waitlist_to_confirmed_notification
        Job->>Mail: waitlist_confirmed email
        Mail-->>U2: "You're In!" email
    end

    Rails-->>U1: 302 Redirect
```

## Technical Nuance

**Counter Cache with Status Transition**

The counter cache only counts confirmed registrations. When promoting from waitlist:

1. Standard `counter_cache` doesn't trigger on `update` (only `create`/`destroy`)
2. We manually increment `registrations_count` in `promote_from_waitlist!`

```ruby
def promote_from_waitlist!
  self.status = :confirmed
  save!
  # Manual increment because update doesn't trigger counter cache
  event_post.increment!(:registrations_count)
end
```

## Promotion Logic

```ruby
# app/models/event_registration.rb
def promote_potential_candidate
  return if event_post.requires_approval?

  next_in_line = event_post.waitlisted_registrations
                           .order(:registered_at)
                           .first

  next_in_line&.promote_from_waitlist!
end
```

## Promotion Rules

| Scenario | Auto-Promote? | Reason |
|----------|---------------|--------|
| Open event, spot opens | Yes | Fair to waitlist |
| Approval-required event | No | Organizer decides |
| No one on waitlist | N/A | Nothing to promote |

## Email Sent

When promoted, user receives **waitlist_confirmed** email:

> Subject: You're In! Your spot for [Event Name] is confirmed
>
> Great news! A spot opened up and you've been moved from the waitlist to confirmed.
