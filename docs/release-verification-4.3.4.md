The approved 4.3.3 release had remaining failures in caption propagation, persisted pane sizes and FloodWait handling. Version 4.3.4 completes those behaviors and fixes unbounded visible logs.

Changes:
- Monitoring dispatch reads the current rule before a linked legacy task; an upload waiting for a global slot resolves its caption after acquiring that slot.
- Tool panes restore after mapping and hidden panes do not overwrite saved sizes.
- Independent monitoring batches recover from their rule or durable snapshot without requiring a legacy upload task; recovery stays paused and idempotent.
- Every visible log stays bounded, including multiline messages.
- Admin FloodWait is persisted at controller setup, participant lookup and promotion boundaries regardless of optional safety settings. Helpers propagate FloodWait instead of retrying another RPC.

Validation: 80 tests pass on Windows with the pinned dependencies, including both caption regressions, 10 FloodWait boundary scenarios, queue recovery, log bounds and actual Tk pane tests. A separate GUI smoke opened all three tools and verified right-side logs and exact sash persistence after restart. The release rebuilds from the verified published 4.3.3 package; user profiles and sessions are not included in the overlay. Candidate artifact verification and final release manifest verification follow before completion.
