---
layout: default
title: did:huuid Method Specification
---

# did:huuid Method Specification

**Version:** 0.1.0-draft  
**Status:** Draft  
**Author:** HUUID Protocol Working Group  
**Contact:** josephtdnarnor@gmail.com  

## Abstract

The `did:huuid` DID method is a health-domain decentralized 
identifier protocol. It maps a cryptographically-anchored patient 
identifier to health record endpoints across disparate clinical 
systems globally. HUUID functions as the "DNS of Health" — 
resolving patient identity without storing medical data.

## Method Syntax

    did:huuid:[country-code]:[base58-encoded-public-key-hash]
    
    Example: did:huuid:gh:7X29ALPHAxyz4Kf9mR2vNbQs

## Resolution Endpoint

    GET https://resolver.huuid.health/1.0/identifiers/{did}

## Status

Full specification: HUUID-RESOLUTION-SPEC-v0.1.1  
Active development by the HUUID Protocol Working Group.
