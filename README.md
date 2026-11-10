# Architecture

A knowledge base of answers to common architectural questions about
PostgreSQL and the IT infrastructure around it.

The content is built up through conversations with
[Claude](https://claude.com/claude-code): questions are discussed, the
trade-offs are written down, and the outcome is kept here as a reference
document.

## Scope

Topics include, but are not limited to:

- **PostgreSQL**: auditing (e.g. pgaudit), high availability, backup and
  restore, replication, upgrades, security, performance and configuration
- **Infrastructure**: Linux, Kubernetes, storage, networking, monitoring
  and logging
- **Operations**: compliance, change management and operational best
  practices

## Structure

Each topic gets its own directory or Markdown document. A document
generally covers:

1. **Question**: what problem are we trying to solve?
2. **Context**: constraints, requirements and assumptions
3. **Options**: possible approaches, with their pros and cons
4. **Recommendation**: the preferred approach and why
5. **References**: links to upstream documentation and other sources

## Disclaimer

The content here reflects discussions and opinions at the time of
writing. Always check it against the upstream documentation and your own
requirements before applying it in production.

## License

This project is licensed under the GNU General Public License v3.0. See
[LICENSE](LICENSE) for the full text.
