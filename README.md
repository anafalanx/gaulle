# Gaulle

Gaulle is a compact Windows runtime for Tcl/Tk programs: one executable that
installs nothing. It holds a small host written in C, Tcl and Tk, SQLite, and
an engine written in Go. Programs are written in Tcl. The engine gives them
evented operations, parallel work, and an HTTPS client and server.

**Status: design.** There is nothing to build or run yet.
[docs/architecture.md](docs/architecture.md) says what is decided, what is
proposed and what is open.

## Licence

Apache-2.0. See [LICENSE](LICENSE).
