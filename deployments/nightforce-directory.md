# Deployment Request: nightforce.cc Directory

## DApp name

nightforce.cc Directory

## DApp description

nightforce.cc is an unofficial public directory for Nightforce ambassadors in the Midnight ecosystem.

The DApp lets approved users create a public profile with selective field disclosure. Users can choose which profile fields are visible, including display name, country, region, role, website, socials, and optional `.night` metadata.

The Midnight contract used for this deployment request is a small Contact Mode contract. It stores a single public enum-like contact availability state for a profile:

- `NO_CONTACT`
- `PRIVATE_CONTACT_AVAILABLE`
- `PUBLIC_CONTACT_ALLOWED`

The contract does not store email addresses, private messages, personal contact data, user funds, or assets. Private email, when used, is handled off-chain by the application using app-side encryption. The on-chain Contact Mode contract only stores whether a public/private/unavailable contact mode exists.

## Contract purpose

The Contact Mode contract acts as a minimal on-chain state anchor for a profile’s contact availability.

Contact Mode is derived from the user’s saved profile state:

- `NO_CONTACT` means no public email and no encrypted private email.
- `PRIVATE_CONTACT_AVAILABLE` means a private encrypted email exists in the app backend, but the email itself is not stored on-chain.
- `PUBLIC_CONTACT_ALLOWED` means the user has chosen to show a public email in their profile.

The backend/D1 database remains the source of truth for the directory profile. The Midnight contract is a sync/attestation layer for the contact mode only.

## Contract code / repository

Repository:

https://github.com/greenpython9/nightforce-directory

Relevant contract/code paths:

- `artifacts/nightforce-directory/midnight-contact-mode/contracts/managed/contact-mode/`
- `artifacts/nightforce-directory/src/services/contactModeWrite.ts`
- `artifacts/nightforce-directory/src/pages/MyProfile.tsx`

## Self-assessment

### Privacy-at-risk: 1

Rationale:

The contract does not store private email addresses, message content, personally sensitive contact data, or hidden profile fields.

The only on-chain state is a coarse contact availability mode:

- no contact available
- private contact available
- public contact allowed

If there were a ZK or contract issue, the exposed data would be limited to this contact-mode enum, which is already intended to be reflected in the public profile UI as contact availability metadata.

Mitigations:

- Private email is not stored on-chain.
- Contact Mode does not contain encrypted email payloads.
- Contact Mode does not reveal the actual private contact address.
- Users control whether their public email is displayed.
- Backend/D1 remains the source of truth if on-chain sync fails.

### Value-at-risk: 1

Rationale:

The Contact Mode contract does not custody user funds, transfer assets, control balances, mint tokens, escrow value, or manage financial claims.

There is no direct financial value at risk in the contract. A failure would affect contact availability metadata, not user assets.

Mitigations:

- No token custody.
- No asset transfer logic.
- No payment or escrow logic.
- No admin-controlled user funds.
- Contract state is limited to a small enum.

### State-space-at-risk: 2

Rationale:

The DApp may create one Contact Mode contract per approved public profile. This creates persistent ledger state, but the growth is bounded by the app’s manual approval flow.

The directory is not an open public contract-deployment surface. Users cannot freely spam contract creation without first going through the Nightforce verification/approval process.

Expected usage is modest and profile-based, not high-frequency or transactional.

Mitigations:

- Profile creation requires verification/approval.
- Contract deployment happens only for approved profiles.
- One intended Contact Mode contract per profile.
- No public anonymous spam deployment path.
- Backend stores sync metadata and can prevent repeated unnecessary deployments.
- Later profile publishes only update the existing contract if the derived contact mode changes.

## Current mainnet status

The same Contact Mode deploy/read flow has been tested successfully on Midnight preprod with 1AM Wallet.

On mainnet, the transaction currently reaches wallet approval and submit flow, but fails at final `submitTransaction` with a generic transaction submission error. Based on guidance from the Midnight team, this may be because the DApp/contract deployment has not yet been approved or authorized for mainnet deployment.

## Requested outcome

We are requesting review and deployment authorization for the nightforce.cc Contact Mode contract on Midnight mainnet.

If approved, we would like to receive the appropriate deployment credentials/instructions needed to deploy this minimal Contact Mode contract on mainnet.
