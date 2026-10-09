Goal: Send prompt, auditable Telegram messages for each V2 paper entry, TP1 and final exit from `src/strategies/london-open-liquidity-macro-v2/`.
Out of scope: V1, generic paper strategies, live orders, entry/exit rules and the existing daily report format.
Steps:
1. In `paper-state.ts`, emit one TP1 transition when a partial fill occurs and snapshot the entry, TP1 and final-exit values needed for messages, including the stored cost-model P&L.
2. In `paper-repository.ts`, `london-v2-paper.schema.ts` and the next operational migration, persist a notification outbox with each trade transition in one transaction, keyed by account, symbol, London date and transition; record send attempts and success.
3. In `run-london-v2-paper-worker.ts` and a V2-only message formatter, send committed pending items to the account's Telegram chat, retry transient failures after restart, and never backfill pre-feature transitions.
4. In the V2 state, repository and notification tests, cover entry, TP1, TP2, stop, time/operator exit, same-candle TP1+TP2 ordering, P&L reconciliation and poll/restart deduplication.
5. Apply the migration, rebuild only the dedicated V2 worker, and confirm it remains entry-paused while another paper strategy is active; do not synthesize a trade to test delivery.
Verification: Typecheck, targeted tests, diff review and static traces for (a) entry then TP1 then TP2, (b) stop before TP1 on one candle and (c) Telegram failure followed by worker restart. The final message's net P&L must equal the stored V2 session ledger.
