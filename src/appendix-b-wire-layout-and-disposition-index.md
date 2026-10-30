# Appendix B — Wire-layout and disposition index

This appendix indexes authoritative tables rather than redefining them.

**Informative table.**

| Object/failure | Authoritative section | Disposition class |
|---|---|---|
| Feature account malformed | 17.1 | Treat feature as inactive; audit malformed account |
| Transaction packet exceeds limit | 6–7 | Drop/reject before execution |
| Transaction signature failure | 7 | Reject before execution; no committed transaction effects |
| Sanitization failure | 7 | Reject before execution |
| Executed instruction error | 11–13 | Record failed execution, roll back instruction writes, retain specified fee/nonce effects |
| Shred signature/version/variant failure | 9 | Drop or mark shred invalid; do not use for reconstruction |
| Incomplete erasure set | 9 | Retain/repair; not a complete slot |
| Entry PoH failure | 8/10 | Reject entry sequence; candidate slot is not votable |
| Competing block ID | 9/10/15 | Keep versions separate; verify the exact certificate; repair and replay before use |
| Unsafe vote or certificate threshold failure | 15 | Abstain or reject the certificate; block validity is unchanged |
