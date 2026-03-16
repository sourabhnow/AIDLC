# AIDLC — AI Development Life Cycle

Best practices, protocols, and tools for AI-assisted software development.

These protocols are designed to be dropped into any project that uses AI coding agents — regardless of tech stack or AI tool (Claude Code, Cursor, GitHub Copilot, etc.).

## Protocols

### [AI Coding Protocol](protocols/UNIVERSAL_CODING_PROTOCOL.md)
A comprehensive coding standard that counteracts systematic biases in AI-generated code. Covers defensive coding, state management, scoring systems, security defaults, error handling, testing philosophy, and more.

### [Testing Protocol](protocols/TESTING_PROTOCOL.md)
A structured testing methodology focused on finding bugs, not confirming things work. Emphasizes end-to-end data tracing, exact evidence, and adversarial thinking.

## Usage

1. Copy the protocol files into your project
2. Reference them in your AI tool's configuration:
   - **Claude Code:** Add to `CLAUDE.md`
   - **Cursor:** Add cardinal rules to `.cursorrules`
   - **GitHub Copilot:** Use `.github/copilot-instructions.md`
   - **Any LLM tool:** Include in your system prompt

## License

Open source. Use freely in your projects.

---

*Maintained by [Sumvec AI](https://sumvec.ai)*
