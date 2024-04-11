# Read Committed
 When a transaction uses this isolation level, a query sees only data committed before the query began; it never sees either uncommitted data or changes committed by concurrent transactions during the query's execution
# Repeatable Read
The Repeatable Read isolation level only sees data committed before the transaction began; it never sees either uncommitted data or changes committed by concurrent transactions during the transaction's execution.
# Serializable
The Serializable isolation level provides the strictest transaction isolation. This level emulates serial transaction execution for all committed transactions; as if transactions had been executed one after another, serially, rather than concurrently.