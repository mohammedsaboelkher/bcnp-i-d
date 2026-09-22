<!-- regenerate: off -->

# Brain Control Network Protocol (BCNP)

This is the working area for the individual Internet-Draft, "Brain Control Network Protocol (BCNP)".

* [Editor's Copy](https://mohammedsaboelkher.github.io/bcnp-i-d/#go.draft-salaheldin-bcnp.html)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-salaheldin-bcnp)
* [Individual Draft](https://datatracker.ietf.org/doc/html/draft-salaheldin-bcnp)
* [Compare Editor's Copy to Individual Draft](https://mohammedsaboelkher.github.io/bcnp-i-d/#go.draft-salaheldin-bcnp.diff)

## Future Work

Deliberately out of scope for the current draft, tracked here rather than in the I-D itself:

* **Multi-controller support.** The current design assumes a single Controller per Device; a Device only ever holds one Device Key/Collection Key at a time. Supporting multiple independent Controllers would need explicit device-ownership-conflict handling (what happens when a second Controller tries to register a Device already claimed by another).
* **Joining a pre-existing, general-purpose WiFi network.** v1 assumes the Controller hosts its own WPA2/3-protected network and Devices join it. Letting BCNP run on a network the Controller doesn't control is a larger trust-model change.
* **Per-device authentication.** v1 accepts, as a residual risk, that any device already on the Controller's WLAN can impersonate the Controller or another Device at the BCNP layer. A planned mitigation is a per-device shared secret issued at Register time, used as an HMAC key on subsequent messages -- paired with replay protection (a monotonic Sequence ID check), since HMAC alone doesn't stop a captured message being replayed verbatim.
* **Reconciliation-discovered identity conflicts.** The reconciliation mechanism assumes at most one Device answers for a given (Collection Key, Device Key) pair. Two Devices genuinely claiming the same identity is an edge case that only becomes reachable once multi-controller support exists, and isn't handled yet.

## Command Line Usage

Formatted text and HTML versions of the draft can be built using `make`.

```sh
$ make
```

Command line usage requires that you have the necessary software installed.  See
[the instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md).

## Contributing

See the
[guidelines for contributions](https://github.com/mohammedsaboelkher/bcnp-i-d/blob/main/CONTRIBUTING.md).