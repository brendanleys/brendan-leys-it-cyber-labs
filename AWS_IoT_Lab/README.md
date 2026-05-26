# AWS IoT Lab

## What This Is

A multi-skill lab combining cloud security, networking, and SOC log analysis. The work involves building an end-to-end IoT telemetry pipeline: a Raspberry Pi sensor device publishes data over MQTT to AWS IoT Core, which routes messages to CloudWatch Logs and S3, then forwards to Sumo Logic (SIEM) and Grafana Cloud (dashboards).

This lab is in progress. Files will be added as I work through each task.

## Skills Practiced

- **Cloud security:** IAM roles, X.509 device certificates, IoT policies, the AWS IoT credential provider pattern (short-lived credentials via cert-to-role assumption)
- **Networking:** MQTT pub/sub, mutual TLS, QoS levels (AT_LEAST_ONCE), topic-based authorization
- **SOC / log analysis:** CloudWatch Logs ingestion, IoT Rules engine, S3 event archival, Sumo Logic SIEM integration
- **Linux:** Raspberry Pi terminal work, SCP file transfer, environment variable management, file permissions on private keys

## Architecture

```
Raspberry Pi (sensors)
    |  MQTT over TLS, X.509 mutual auth
    v
AWS IoT Core (message broker)
    |  IoT Rules engine
    +--> CloudWatch Logs --> Sumo Logic (SIEM)
    |                    --> Grafana Cloud (dashboards)
    +--> S3 (event archival)
```

## Files in This Folder

- `day1-iot-setup-and-policy-debug.md` — initial provisioning, SDK install, test publish, and debugging a case-sensitivity bug in the IoT policy

## Why the Credential Provider Pattern Matters

The lab is built around an important security principle: the Raspberry Pi never stores long-lived AWS access keys. Instead, it presents its X.509 device certificate to a credential provider endpoint, which exchanges the cert for short-lived (1-hour) AWS credentials via IAM role assumption. If a device is lost or compromised, revoking the certificate locks it out within an hour — no other resources affected.

This is a fundamentally different security model from "put an access key on the device," and it's the pattern AWS recommends for any production IoT deployment.

## Interview Explanation

I worked through an AWS IoT Core lab where I provisioned IoT resources from scratch using the AWS CLI: an IoT Thing, X.509 device certificate, IoT policy scoping MQTT publish permissions, IAM role for the credential provider, and a role alias. The Raspberry Pi authenticates to AWS using mutual TLS with its certificate, then uses the credential provider to obtain short-lived credentials for any AWS API calls — no access keys ever live on the device. I can explain the full flow from cert presentation through IAM role assumption to signed API calls.

## Caveat

This is lab-based practice in a course AWS account. I am not claiming production IoT deployment experience — I am claiming I understand the AWS IoT security model, the MQTT pub/sub flow, and can provision, debug, and verify this stack end to end.
