# Bugfix Requirements Document

## Introduction

This document specifies the requirements for fixing a critical bug in the Profile Reconciler (`internal/reconcilers/run_profile.go`) where a single entity marshalling failure permanently aborts profile initialization for all remaining entities in a project. When a new Profile is initialized or updated, the system must iterate over all entities (e.g., thousands of repositories) and publish evaluation events. Currently, if message marshalling fails for one entity, the entire loop terminates with `return nil`, causing the remaining entities to be silently skipped without any retry mechanism.

## Bug Analysis

### Current Behavior (Defect)

1.1 WHEN `entRefresh.ToMessage(m)` fails for any entity during profile initialization THEN the system returns nil and aborts the entire entity processing loop

1.2 WHEN the loop aborts on entity N out of M total entities (where N < M) THEN the remaining (M - N) entities never receive their profile evaluation events

1.3 WHEN the reconciler returns nil after a marshalling failure THEN Watermill ACKs the message as successfully handled and never retries the operation

1.4 WHEN profile initialization completes with silently skipped entities THEN users incorrectly assume their security profile was applied to all repositories

### Expected Behavior (Correct)

2.1 WHEN `entRefresh.ToMessage(m)` fails for any entity during profile initialization THEN the system SHALL log the error for that specific entity and continue processing the next entity

2.2 WHEN marshalling fails for entity N out of M total entities THEN the system SHALL continue processing entities N+1 through M

2.3 WHEN an individual entity's message marshalling fails THEN the system SHALL record the failure in logs with sufficient context (entity ID, error details) for debugging

2.4 WHEN profile initialization completes THEN the system SHALL have attempted to process all entities regardless of individual marshalling failures

### Unchanged Behavior (Regression Prevention)

3.1 WHEN `entRefresh.ToMessage(m)` succeeds for an entity THEN the system SHALL CONTINUE TO publish the message to `TopicQueueRefreshEntityByIDAndEvaluate`

3.2 WHEN `r.evt.Publish()` fails for an entity THEN the system SHALL CONTINUE TO return an error to trigger Watermill retry

3.3 WHEN `r.store.GetEntitiesByProjectHierarchy()` fails THEN the system SHALL CONTINUE TO return an error to trigger Watermill retry

3.4 WHEN all entities are processed successfully THEN the system SHALL CONTINUE TO return nil indicating successful completion
