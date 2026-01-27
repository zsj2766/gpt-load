Please review the following code changes:

[Change summary]
Added RetryStrategy support to SystemSettings and GroupConfig, and ensured it propagates to EffectiveConfig.
Fixed previous issue where DB column value was unconditionally overwriting EffectiveConfig for non-aggregate groups.

[Modified Files]
- internal/types/types.go
- internal/models/types.go
- internal/services/group_manager.go

[Key changes]
1. Added `RetryStrategy` field to `SystemSettings` struct (default: "auto").
2. Added `RetryStrategy` field to `GroupConfig` struct (as *string pointer).
3. Updated `GroupManager.Initialize` to explicitly set `EffectiveConfig.RetryStrategy` from the `Group` struct's `RetryStrategy` field ONLY IF `GroupType` is "aggregate" and the strategy is not empty.

Check: correctness, requirement coverage, potential bugs, code quality, security.

**CRITICAL OUTPUT RULES:**
1. If ALL checks PASS: Output "PASS" with brief confirmation
2. If ANY issue found: You MUST provide specific Search/Replace code blocks to fix it
   - DO NOT just describe the issue in text
   - DO NOT just suggest what should be changed
   - You MUST output the exact code fix in this format:

// File: path/to/file
<<<<<<< SEARCH
[exact original code to find, 3+ lines of context]
=======
[exact replacement code]
>>>>>>>

For NEW files: leave SEARCH section empty.
For DELETING code: leave the section after ======= empty.

Without Search/Replace blocks, your review is INVALID.