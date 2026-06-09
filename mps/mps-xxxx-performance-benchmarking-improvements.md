---
MPS: "xxxx" 
Title: Perfromance Benchmarking and Improvement
Authors: Karmel E - karmoola
Status: Proposed  
Category: Core   
Created: 06-08-2026  
Requires: [none]  
Replaces: [none]  

---

# Benchmarking - Improvements

## **Abstract**

This problem statement concerns two coupled performance dimensions of the Midnight protocol: (1) the speed at which a node synchronizes the chain ("sync throughput") and (2) the growth rate of the nullifier set.

## **Goals and Success Criteria**

Satisfied when the following are established (the *how* belongs in a MIP):

1. A reproducible sync-throughput baseline on defined reference hardware at a defined chain height, expressed in × realtime.
2. A reproducible nullifier-set baseline: size and growth rate (per block / epoch / transaction), plus measured contribution to synced-state size, sync wall-clock time, and per-transaction validation cost.
3. A quantified nullifier-set growth-rate or cost-ceiling target, set once the baseline is known.
4. A defined improvement limit for each metric, justified by operator benefit and consensus-correctness constraints.

## **Motivation**

- Sync time gates participation: Every new or recovering node operator must replay and validate historical chain state before participating. Slow sync raises the barrier to running a node as history accumulates.
- No benchmarking capability: There is no standard benchmark for monitoring either sync throughput or nullifier-set cost and impact of any current/future network updates. Without a benchmark, no change can be evaluated as an improvement or a regression.