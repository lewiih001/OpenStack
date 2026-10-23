# Operations

### US-020: On-call is paged when users are affected

| Field | Value |
| --- | --- |
| Story | As an operator, I am paged when the public API stops serving users, so that I start recovery before customers report it. |
| Input | The external API VIP stops answering, or the canary fails twice in a row |
| Output | A critical alert naming the service and linking its runbook reaches the on-call channel and phone |
| SLA | Page delivered within 5 min of the first failure; zero pages for alerts silenced by an announced maintenance window |
| Verified by | Lab and quarterly production test: block the VIP at the firewall and time the page |
| Phase | 7 (alerting ships with every phase from 1) |
