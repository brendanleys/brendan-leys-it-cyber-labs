# GRC Evidence Mapping

This folder contains exercises focused on mapping technical evidence to security controls, risks, and mitigation strategies.

## Planned Topics

- MFA evidence
- Patch management evidence
- Endpoint security baselines
- Asset inventory review
- Risk identification
- Control mapping

## Goal

Practice explaining how technical safeguards reduce operational and security risk.

## Completed Labs
- credential-migration-bitwarden.md — full lab writeup of migrating a plaintext credentials file into Bitwarden (Password Manager + Secrets Manager), with naming convention, verification gate, and secure deletion via `shred -u`. Covers handling of a Supabase service-role key (the highest-risk credential in the capstone) under the three-locations-only rule.

## What This Folder Demonstrates
Hands-on practice with security governance controls — specifically credential hygiene, secrets-manager use, separation of human logins from machine secrets, verification before destruction of sensitive artifacts, and the rationale behind why certain credentials need stricter handling than others.

