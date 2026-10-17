# Warehouse Orders — Security review (working branch)

## Scope and status

This is a preliminary source review of the repository. It is not a penetration test and does not prove that the deployed Firebase project uses these exact rules. No production Firebase resources have been changed by this branch.

## Changes in this branch

- Tighten Firestore `pushQueue` document validation:
  - bound title to 1–120 characters and body to 500 characters;
  - validate optional `orderId` and `createdAt` types;
  - cap recipient UID length and require a non-empty recipient to reference an active user;
  - retain deny-by-default client read/update/delete on the queue.

## Findings requiring follow-up

1. **Push queue can still be abused for repeated notifications.** Firestore Rules cannot implement a robust per-user rate limit. Before production deployment, the server function should validate each queued event against its source (order/chat/mention), authorize the recipient, and reject arbitrary broadcast payloads. Prefer creating notifications via a trusted callable/HTTPS function rather than accepting arbitrary client-created queue entries.
2. **Storage image MIME types are client-supplied metadata.** The current rules cap size and require an `image/*` content type, but this does not validate the actual file bytes and permits SVG. Decide which raster formats the application actually uploads, then restrict the allowed types/extensions and consider image re-encoding server-side. Keep current photo read access only if every active employee is authorized to see every order's defect photos.
3. **Roles are stored in Firestore user profiles.** Review every rule path for privilege escalation and test inactive users, missing profiles, warehouse, manager, accountant, and director separately. Admin SDK code bypasses Firestore Rules and must enforce its own authorization.
4. **Public 1C import endpoint.** The function uses the `ONE_C_API_KEY` secret, which is the correct general pattern. Verify that the secret is configured in Secret Manager, rotate it if it has ever been exposed, impose request-size and field-length limits, and avoid logging secrets or sensitive payloads.
5. **Secret history scan.** Search the full Git history, GitHub Actions logs/artifacts, Firebase configuration, and Cloud logs for service-account private keys, access tokens, passwords, and the 1C secret. Removing a secret from the latest commit does not revoke it; rotate any exposed credential.
6. **Live cloud configuration is unverified.** Confirm the deployed Firestore and Storage Rules, Authentication providers and authorized users, App Check, API key restrictions, billing alerts, and IAM roles directly in Firebase/Google Cloud Console.

## Release gate

Do not deploy this branch to the production Firebase project until the Firestore Rules Emulator tests cover:
- unauthenticated read/write attempts;
- authenticated users with no profile and inactive profiles;
- each role's permitted and forbidden order changes;
- self-promotion and modification of another user's role/admin/active fields;
- access to deleted orders and user notifications/tokens;
- queue payload bounds, inactive/nonexistent recipients, and attempts to read/update/delete queued events;
- Storage access, oversized uploads, and disallowed content types.

After emulator tests pass, deploy rules separately from application changes and verify them against the intended Firebase project ID before proceeding.
