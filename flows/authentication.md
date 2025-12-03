# Authentication Flow (Devise)

## Registration & Login Flow

```mermaid
flowchart TD
    subgraph "Registration"
        R1[User submits signup form]
        R2[Devise creates user<br/>confirmed_at: nil]
        R3[Devise sends confirmation email]
        R4[User clicks confirmation link]
        R5[Devise sets confirmed_at]
        R6[User can now login]
    end

    subgraph "Login"
        L1[User submits email/password]
        L2{Email confirmed?}
        L3[Devise validates credentials]
        L4{First login?}
        L5[Redirect to /about<br/>onboarding]
        L6[Redirect to /event_posts<br/>home]
        L7[Error: Please confirm email]
    end

    R1 --> R2 --> R3 --> R4 --> R5 --> R6

    L1 --> L2
    L2 -->|No| L7
    L2 -->|Yes| L3
    L3 --> L4
    L4 -->|Yes| L5
    L4 -->|No| L6

    style L7 fill:#ff6b6b
    style R6 fill:#51cf66
    style L6 fill:#51cf66
```

## Technical Nuance

**Strict Email Confirmation**

```ruby
# config/initializers/devise.rb
config.allow_unconfirmed_access_for = 0.days  # Must confirm immediately
config.confirm_within = 3.days                 # Link expires in 3 days
```

This means:
- Users **cannot** access the app without confirming email
- No grace period for unconfirmed accounts
- Confirmation link expires after 3 days

## First-Time Login Detection

```ruby
# app/controllers/application_controller.rb
def after_sign_in_path_for(resource)
  if resource.sign_in_count == 1
    about_path  # Onboarding page
  else
    root_path   # Event listings
  end
end
```

## User Roles

| Role | Value | Permissions |
|------|-------|-------------|
| `student` | 0 | Create events, register, view leaderboard |
| `club_admin` | 1 | Same as student (reserved for future) |
| `super_admin` | 2 | Delete users/events, view audit logs |

## Session Management

- **Remember me**: Optional, extends session
- **Timeout**: Uses Rails default session expiry
- **Password recovery**: Email-based reset flow

## Security Features

1. **bcrypt** password hashing
2. **CSRF** protection (Rails default)
3. **Email confirmation** required
4. **Rate limiting** (TODO: implement)
