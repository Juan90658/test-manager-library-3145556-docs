# ADR-005 · Optimistic control on copies without a version column

- **Status:** accepted
- **Context:** ADR-002 says the book carries a *version token* and the save fails if the version changed.
  `data-model.md` §3 lists every `book` column and there is no `version`; the data model also says it wins over any
  other document. Adding the column would make the data model lie; ignoring ADR-002 would break RNF-2.
- **Decision:** the counter itself is the token. Lending runs a conditional update inside the loan transaction:
  `UPDATE library.book SET available_copies = available_copies - 1 WHERE id = ? AND deleted_at IS NULL AND available_copies >= 1`.
  If it affects 0 rows, another loan took the last copy: the transaction rolls back and the API answers
  **409 `BOOK_NOT_AVAILABLE`**, recording nothing (CA-06.3). Editing the total uses the same pattern with the difference.
  `ck_book_available_range` stays as the last barrier, as ADR-002 requires.
- **Alternatives discarded:** a `version` column (contradicts data-model §3 and would need the document changed for a
  technical detail); `SELECT … FOR UPDATE` on every book (pessimistic, rejected by ADR-002 itself).
- **Consequences:** no extra column, no retries needed. PostgreSQL row locking during the `UPDATE` serializes only the two
  requests competing for the same book, never the whole library.
- **How it is verified:** `api/tests/concurrency/last-copy.sh` sends two simultaneous loans for a book with one copy:
  exactly one `201`, one `409`, and `available_copies = 0`.
