# dsh-plugin-session-delete-uni-kaby
Universal session delete for DeepSeek Harness with a mandatory archive gate: a session must be archived before it can be permanently removed. Accepts any id form (uuid / session-&lt;uuid> / imported oc-* or qoder-*); clears the log, projection cache, workspace accounting and archive bookkeeping.
