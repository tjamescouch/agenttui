Title: Improve agenttui usability: fuzzy filter + Enter-to-chat

Summary:
- Add a lightweight fuzzy filter for the agent list so users can type partial names and get ranked matches. The fuzzy matcher is sequential and rewards consecutive matches; it's small, deterministic, and has no extra dependencies.
- Improve chat UX: pressing Enter when the chat panel is focused opens the chat input; after submitting a message, focus remains in the chat input.

Files changed:
- index.js: added fuzzy matching logic in renderAgentList(), added Enter key handler and focus retention after chat submit
- README.md: documented the prepared branch and patch

Testing performed:
- Ran a headless smoke test (node index.js) to ensure no immediate runtime error (UI control sequences were produced; no crash)
- Committed changes on branch `oai/fuzzy-filter-enter-chat`.

Notes:
- I created patch files in /tmp/agenttui_patches (format-patch against origin/main). You can apply them locally or I can open a PR — pushing requires credentials or a fork.
- If you'd like, I can also extract the fuzzy matcher into a small module or add unit tests for fuzzyScore.

PR ready to open from branch: oai/fuzzy-filter-enter-chat

PR checklist (suggested):
- [ ] Confirm CI (if any) passes
- [ ] Run interactive manual test with agents directory
- [ ] Squash/adjust commits if desired

PR created by automated agent (OAI). Please let me know if you want me to attempt to push/open the PR.
