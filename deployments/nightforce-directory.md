# Deployment Request: nightforce.cc Directory

## DApp name

nightforce.cc Directory

## DApp description

nightforce.cc is an unofficial public directory for Nightforce ambassadors in the Midnight ecosystem.

The DApp lets approved users create a public profile with selective field disclosure. Users can choose which profile fields are visible, including display name, country, region, role, website, socials, and optional `.night` metadata.

The Midnight contract used for this deployment request is a small Contact Mode contract. It stores minimal contact availability state for approved profiles:

- `NO_CONTACT`
- `PRIVATE_CONTACT_AVAILABLE`
- `PUBLIC_CONTACT_ALLOWED`

The contract does not store email addresses, private messages, personal contact data, user funds, or assets. Private email, when used, is handled off-chain by the application using app-side encryption.

## Updated Contact Mode architecture

Nightforce Contact Mode has been migrated from a per-profile contract model to a single developer-owned global Contact Mode contract.

Users no longer deploy their own Contact Mode contracts.

Instead:

1. The DApp developer deploys one global Contact Mode contract.
2. Each approved profile receives an active app-issued Contact Mode entry key.
3. Approved users register or update their own entry inside the global contract.
4. The backend/D1 database remains the source of truth for profile ownership, wallet binding, and active entry metadata.

## Preprod global contract

Network: Midnight Preprod

Global Contact Mode contract address:

```text
e0eb2b63c970003356754cdddaf1b4f3e9f6e8d4f7dda18f030aa5f102865424
```

## Contract purpose

The Contact Mode contract acts as a minimal on-chain state anchor for a profile’s contact availability.

Contact Mode is derived from the user’s saved profile state:

- `NO_CONTACT` means no public email and no encrypted private email.
- `PRIVATE_CONTACT_AVAILABLE` means a private encrypted email exists in the app backend, but the email itself is not stored on-chain.
- `PUBLIC_CONTACT_ALLOWED` means the user has chosen to show a public email in their profile.

The backend/D1 database remains the source of truth for the directory profile. The Midnight contract is a sync/attestation layer for contact mode only.

## On-chain data

For each active profile entry key, the global contract stores:

- `profileKey -> ownerCommitment`
- `profileKey -> contactMode`
- registered / not registered state

The contract does not store:

- profile content
- email addresses
- encrypted email payloads
- private messages
- social links
- wallet binding records
- user funds or assets

## Off-chain / app-side data

The app database stores:

- verification requests
- approved profile records
- wallet bindings
- active Contact Mode entry metadata
- public profile fields
- field disclosure settings
- encrypted private contact payloads when private contact is used

Private email is not uploaded on-chain.

If private contact is used, the email is encrypted app-side before being stored in the app database. Midnight only stores the minimal Contact Mode status and owner commitment used for sync.

## Authentication pattern

The global Contact Mode contract does not use `ownPublicKey()` for authentication.

The global flow uses:

- `localSecretKey` witness
- `persistentHash`
- `ownerCommitment`

The user proves control of an entry by reproducing the stored `ownerCommitment` for the active `profileKey`.

The global Contact Mode write/deploy flow avoids `withVacantWitnesses` and binds witnesses explicitly.

## Register / update flow

For a newly approved profile:

1. The backend issues an active Contact Mode entry key.
2. The user registers that entry in the global contract.
3. The contract stores the owner commitment and initial Contact Mode value.
4. Later profile publishes update the same global entry when Contact Mode changes.

End users do not deploy contracts.

## Recovery / rotation flow

If a user loses the local Contact Mode secret:

1. The user reconnects the wallet already bound to the approved profile.
2. The backend verifies the wallet owns the approved profile binding.
3. The app issues a new active Contact Mode entry key.
4. The user registers the new entry on-chain with a new owner commitment.
5. D1 tracks the new active entry.

Older entries are not treated as active by the app after rotation.

## Account deletion flow

When a user deletes their account:

1. If Contact Mode is registered and not already `NO_CONTACT`, the app first asks the wallet to reset the active global Contact Mode entry to `NO_CONTACT`.
2. After the reset succeeds, the app deletes the user’s app-side profile records.
3. The profile, wallet binding, and verification request are removed from the app database.
4. The same wallet can request verification again from the beginning.

The app does not claim to erase historical on-chain state. It resets the active Contact Mode entry to `NO_CONTACT` before deleting app records.

## Contract code / repository

Repository:

https://github.com/greenpython9/nightforce-directory

Relevant contract/code paths:

- `artifacts/nightforce-directory/midnight-contact-mode/contracts/contact-mode-global.compact`
- `artifacts/nightforce-directory/midnight-contact-mode/contracts/managed/contact-mode-global/`
- `artifacts/nightforce-directory/src/services/contactModeGlobalWrite.ts`
- `artifacts/nightforce-directory/src/services/contactModeGlobalConfig.ts`
- `artifacts/nightforce-directory/src/pages/MyProfile.tsx`
- `artifacts/api-server/src/routes/nightforce.ts`
- `artifacts/nightforce-worker/src/index.ts`

## Self-assessment

### Privacy-at-risk: 1

Rationale:

The contract does not store private email addresses, message content, personally sensitive contact data, hidden profile fields, or encrypted contact payloads.

The only on-chain state is a coarse contact availability mode:

- no contact available
- private contact available
- public contact allowed

If there were a ZK or contract issue, the exposed data would be limited to this contact-mode enum and owner commitment metadata. The actual private email remains off-chain and encrypted app-side.

Mitigations:

- Private email is not stored on-chain.
- Contact Mode does not contain encrypted email payloads.
- Contact Mode does not reveal the actual private contact address.
- Users control whether their public email is displayed.
- Backend/D1 remains the source of truth if on-chain sync fails.
- Account deletion resets the active Contact Mode entry to `NO_CONTACT` before deleting app records.

### Value-at-risk: 1

Rationale:

The Contact Mode contract does not custody user funds, transfer assets, control balances, mint tokens, escrow value, or manage financial claims.

There is no direct financial value at risk in the contract. A failure would affect contact availability metadata, not user assets.

Mitigations:

- No token custody.
- No asset transfer logic.
- No payment or escrow logic.
- No admin-controlled user funds.
- Contract state is limited to Contact Mode metadata.

### State-space-at-risk: 2

Rationale:

The DApp now uses one developer-owned global Contact Mode contract instead of one contract per approved profile.

This reduces contract deployment surface area. However, the global contract still stores persistent per-profile entry state for approved profiles.

Expected usage is modest and profile-based, not high-frequency or transactional. Growth is bounded by the app’s manual verification and approval flow.

Mitigations:

- End users do not deploy contracts.
- Profile creation requires verification and approval.
- One global contract is used for all approved profiles.
- Each approved profile gets one active Contact Mode entry key.
- Recovery/rotation issues a new active entry only after the bound wallet is verified by the backend.
- Account deletion resets the active Contact Mode entry to `NO_CONTACT`.
- Backend/D1 tracks the active entry and prevents repeated unnecessary writes.

## Preprod QA summary

Preprod QA passed for:

- fresh entry registration
- Contact Mode updates
- private contact
- recovery / rotation after local secret loss
- account deletion with Contact Mode reset
- re-requesting verification after deletion

## Current mainnet status

Mainnet deployment has not started yet.

The current preprod deployment is used as implementation evidence for the contract deploy rubric and guarded launch review.

After the rubric review passes, the project will follow the Midnight team’s guarded mainnet deployment procedure for deploying the single developer-owned global Contact Mode contract.

## Requested outcome

We are requesting review of the updated global Contact Mode architecture for the nightforce.cc Directory deployment request.

If the deployment request passes the rubric, we would like to follow the appropriate guarded mainnet deployment procedure for this single developer-owned global Contact Mode contract.

