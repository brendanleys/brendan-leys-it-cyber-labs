# AWS IoT Core — Day 1: Setup, SDK Install, and Policy Debug

Date: May 26, 2026
Environment: Chromebook (Linux dev container) + Raspberry Pi (sensors)
Time spent: ~2 hours
Lab context: Course lab, team-based — I worked on environment setup and SDK install while a teammate handled the AWS resource provisioning

---

## Goal

Configure a Raspberry Pi to publish MQTT telemetry to AWS IoT Core using X.509 mutual TLS authentication, with no long-lived AWS access keys stored on the device.

---

## What I Did

### Part 1 — Environment Setup

Set up working directories and the `TEAM_ID` environment variable on both machines.

On the Pi:

    $ mkdir -p /home/argus/Desktop/iot-lab/{certs,scripts,findings}
    $ echo 'TEAM_ID="alpha"' | sudo tee -a /etc/environment
    $ source /etc/environment && export TEAM_ID

The lab specifically puts `TEAM_ID` in `/etc/environment` rather than `~/.bashrc` because the credential helper script (used later) gets spawned as a subprocess by the AWS CLI. `/etc/environment` is inherited by every process on the system, while `~/.bashrc` only loads for interactive bash shells. Putting it in the wrong place would cause the credential helper to silently build the wrong resource names.

On the Chromebook, after installing the AWS CLI v2 and running `aws configure`:

    $ aws sts get-caller-identity
    {
        "UserId": "<redacted>",
        "Account": "<redacted>",
        "Arn": "arn:aws:iam::<account>:user/<my-username>"
    }

Then I fetched the two IoT endpoint addresses needed by the Pi:

    $ aws iot describe-endpoint --endpoint-type iot:Data-ATS ...
    $ aws iot describe-endpoint --endpoint-type iot:CredentialProvider ...

Two different endpoints because they serve different purposes:
- **Data endpoint** (`*-ats.iot.<region>.amazonaws.com`) — where MQTT messages get published
- **Credential provider endpoint** (`*.credentials.iot.<region>.amazonaws.com`) — where the Pi exchanges its X.509 cert for short-lived AWS credentials

### Part 2 — AWS IoT Device SDK Install

On the Pi:

    $ pip3 install awsiotsdk --break-system-packages
    $ python3 -c "import awsiot; print('SDK installed')"
    SDK installed

The `--break-system-packages` flag is needed on Raspberry Pi OS (Debian Bookworm and newer) because the system Python is protected from pip by default. For a lab environment this is fine; in production you'd use a virtual environment.

### Part 3 — Test Publish Script

I wrote a small script to verify the full chain: cert authentication, TLS handshake, IoT policy authorization, MQTT publish. Key things to note:

- Uses `QoS.AT_LEAST_ONCE` (QoS 1) from `awscrt.mqtt`, not the integer `1`. Passing `qos=1` causes the SDK to throw `AssertionError`.
- AWS IoT Core does not support QoS 2 at all. QoS 1 is the right choice for telemetry — guarantees delivery with possible duplicates, which is better than silent message loss.
- The client ID matters. The IoT policy scopes `iot:Connect` to a specific client ID pattern, so the script's `CLIENT_ID = f"argus-{TEAM_ID.lower()}-01"` has to match what the policy allows.

---

## What Went Wrong

When I ran the test script, I got:

    Connecting...
    Connected.
    (hang — no further output)

The connection succeeded but the publish hung indefinitely. Ctrl+C was the only way out.

---

## Debugging Process

**Initial hypothesis:** Network or firewall issue. Easy to test — but ruled out, because if the network were blocking it, the `Connecting...` step would have failed, not the publish.

**Better hypothesis:** Authentication worked (cert was valid, TLS handshake completed, client ID matched the policy), but the policy was denying the publish itself. AWS IoT Core's behavior on a policy denial is to silently drop the connection rather than return an error — so the SDK is left waiting for an ack that will never come.

I checked what policies were attached to the cert and what they actually allowed:

    $ aws iot list-thing-principals --thing-name argus-alpha-01
    $ aws iot list-attached-policies --target <cert-arn>
    $ aws iot get-policy --policy-name argus-alpha-policy --query 'policyDocument' --output text

The policy came back with this for the publish topics:

    "Resource": [
      "arn:aws:iot:us-east-1:*:topic/argus/ALPHA/telemetry",
      "arn:aws:iot:us-east-1:*:topic/argus/ALPHA/compound_event",
      "arn:aws:iot:us-east-1:*:topic/argus/ALPHA/integrity"
    ]

But my script was publishing to `argus/alpha/telemetry` (lowercase). **AWS IoT topics are case-sensitive**, so `argus/alpha/telemetry` ≠ `argus/ALPHA/telemetry`. The publish was being denied, the connection was being dropped, and the SDK was hanging on an ack that would never come.

### Root Cause

The lab's policy JSON template uses `${TEAM_ID}` (raw) for topic ARNs but `${TEAM_ID,,}` (lowercased bash parameter expansion) for client IDs. My teammate had `TEAM_ID="ALPHA"` exported when they created the policy, so:

- Client ID ARN used `${TEAM_ID,,}` → `argus-alpha-*` (lowercase, correct)
- Topic ARNs used `${TEAM_ID}` → `argus/ALPHA/...` (uppercase, mismatched)

The policy was internally inconsistent, and the inconsistency only surfaced at publish time.

### Fix

Created a new version of the IoT policy with lowercase topic ARNs and set it as the default version:

    $ aws iot create-policy-version \
        --policy-name argus-alpha-policy \
        --policy-document file://iot-policy-fixed.json \
        --set-as-default

AWS IoT policies are versioned — there's still only one policy with that name, but it now has multiple versions and only the default is enforced. The old (broken) version is preserved as inert history, which is actually useful: anyone reviewing the policy can see what changed and why.

After applying the fix, the test publish succeeded immediately:

    Connecting...
    Connected.
    Published to argus/alpha/telemetry
    Done.

And the message appeared in the AWS Console MQTT test client subscribed to `argus/alpha/#`.

---

## Key Takeaways

- **AWS IoT topics are case-sensitive.** Policy ARNs and publish targets must match exactly.
- **Policy denials manifest as silent connection drops, not error messages.** When an MQTT client hangs after `Connected.`, suspect authorization, not network.
- **The credential provider pattern is the right way to put devices on AWS.** No long-lived access keys on the Pi — only a certificate, which can be revoked instantly if compromised.
- **IoT policies are versioned.** You don't replace a policy; you create a new version and promote it. The old version sticks around as audit history.
- **QoS 1 (AT_LEAST_ONCE) is correct for telemetry.** QoS 0 risks silent message loss; QoS 2 isn't supported by AWS IoT Core anyway.
- **`/etc/environment` vs `~/.bashrc` matters when subprocesses need the variable.** Interactive shells aren't the only thing that reads environment variables.

---

## What I Would Do Differently

The case-sensitivity bug took about 20 minutes to diagnose, but it would have been instant if I had checked the policy contents before running the test script. Next time I work with an IoT policy I didn't write, the first thing I'll do is read it back with `aws iot get-policy` and verify every ARN matches what the publisher is actually going to do.

A related lesson: when something that should work hangs instead of erroring, the silence itself is information. AWS IoT Core specifically chooses to drop the connection silently on policy denial — knowing that behavior turns a frustrating hang into a clear diagnostic signal.
