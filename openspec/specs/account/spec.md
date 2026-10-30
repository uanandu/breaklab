# account Specification

## Purpose

Lets a learner create an optional hosted identity — separate from the
local, unauthenticated `breaklab serve` UI — so progress can follow them
across devices and they can opt into a public profile page.

## Requirements

### Requirement: Account signup and login
The system SHALL let a visitor create an account with an email, a
unique public username, and a password, and SHALL let an existing
account holder log in and out, independent of and without altering the
existing local `breaklab serve` flow (which remains unauthenticated).
The username SHALL be validated against the same `^[a-z0-9-]+$` rule
this project already applies to lab and variant ids, and is the handle
used by the `public-profile` capability's `/u/<name>` page.

#### Scenario: Successful signup
- **WHEN** a visitor submits a new email, an available username, and a
  password on the signup page
- **THEN** the system creates an account with a bcrypt-hashed password
- **AND** starts an authenticated session for them
- **AND** redirects them to the account dashboard

#### Scenario: Duplicate email rejected
- **WHEN** a visitor signs up with an email already registered
- **THEN** the system rejects the signup with a clear error
- **AND** does not reveal whether the existing account's password matches

#### Scenario: Duplicate username rejected
- **WHEN** a visitor signs up with a username already taken
- **THEN** the system rejects the signup with a clear error naming the
  conflict, since the username is publicly visible and collisions are
  safe to disclose (unlike email)

#### Scenario: Invalid username rejected
- **WHEN** a visitor signs up with a username that doesn't match
  `^[a-z0-9-]+$`
- **THEN** the system rejects the signup with a clear error

#### Scenario: Login with correct credentials
- **WHEN** an account holder submits their email and correct password
- **THEN** the system starts an authenticated session and redirects to
  the dashboard

#### Scenario: Login with incorrect credentials
- **WHEN** an account holder submits an incorrect password
- **THEN** the system rejects the login with a generic error that does
  not indicate whether the email exists

#### Scenario: Logout
- **WHEN** a logged-in user requests logout
- **THEN** the system invalidates their session
- **AND** subsequent requests to account-only pages redirect to login

### Requirement: API token for CLI linking
The system SHALL let a logged-in account holder generate a long-lived
API token from the account settings page, and SHALL let them revoke and
regenerate it.

#### Scenario: Generating a token
- **WHEN** a logged-in user requests a new API token on the account
  settings page
- **THEN** the system issues a token shown once in full
- **AND** stores only a hash of it, never the plaintext token

#### Scenario: Revoking a token
- **WHEN** a logged-in user revokes their API token
- **THEN** any subsequent `/sync` request bearing the old token is
  rejected

#### Scenario: Regenerating replaces the old token
- **WHEN** a logged-in user generates a new token while an old one
  exists
- **THEN** the old token stops working as soon as the new one is issued

### Requirement: Account deletion
The system SHALL let a logged-in account holder permanently delete
their account and all associated synced data.

#### Scenario: Deleting an account
- **WHEN** a logged-in user confirms account deletion
- **THEN** the system deletes the account, its sessions, its tokens,
  and every synced attempt record
- **AND** any public profile page for that account stops resolving
