# Example values

These files are safe, generic examples for readers building a similar Travel Router. They are templates, not the deployed instance configuration.

- `example.env` — placeholder values for the USB LAN, upstream interface, NetBird relay, and local service paths.

Replace every placeholder deliberately. Do not copy private addresses, peer identities, usernames, command paths, Wi-Fi credentials, NetBird setup keys, or SSH private-key paths from another deployment.

The deployed instance record remains private because it contains the appliance's actual addresses, peer identities, control dependencies, and host-specific command paths. Runtime secrets belong on the appliance or in an approved secret store, never in this directory.
