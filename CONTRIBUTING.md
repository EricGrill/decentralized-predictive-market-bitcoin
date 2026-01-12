# Contributing

Thank you for your interest in contributing to the Bitcoin-Native Capsule Prediction Market Protocol!

---

## Ways to Contribute

### 1. Design Discussions

The most valuable contributions at this stage are design input:

- **Propose solutions** to [open design questions](docs/review/README.md)
- **Identify new issues** we haven't considered
- **Challenge assumptions** in the current design
- **Share relevant research** from other projects

Start by opening a [Discussion](../../discussions) before creating issues or PRs.

### 2. Specification Improvements

Help improve the protocol specification:

- **Clarify ambiguous sections** - If something is unclear, others will struggle too
- **Add examples** - Concrete examples help understanding
- **Fix errors** - Typos, incorrect calculations, logical inconsistencies
- **Add test vectors** - Crucial for implementation compatibility

### 3. Security Review

Review the protocol for security issues:

- **Attack vectors** we haven't considered
- **Trust assumption gaps** - Where does trust leak?
- **Economic exploits** - Can rational actors game the system?
- **Implementation pitfalls** - What will implementers get wrong?

### 4. Implementation

When the spec stabilizes:

- **Reference implementations** in Rust, TypeScript, etc.
- **Testing tools** for protocol compliance
- **Developer tooling** for market creation/participation

---

## Contribution Process

### For Discussions

1. Search existing discussions to avoid duplicates
2. Open a new discussion with clear title and context
3. Engage constructively with feedback
4. Summarize conclusions for future reference

### For Issues

1. Search existing issues to avoid duplicates
2. Use the appropriate issue template:
   - [Design Question](.github/ISSUE_TEMPLATE/design-question.md)
   - [Spec Improvement](.github/ISSUE_TEMPLATE/spec-improvement.md)
   - [Bug Report](.github/ISSUE_TEMPLATE/bug-report.md)
3. Provide all requested information
4. Respond to maintainer questions

### For Pull Requests

1. **Discuss first** - Open an issue or discussion before significant changes
2. **One concern per PR** - Keep PRs focused and reviewable
3. **Follow conventions** - Match existing style and structure
4. **Update related docs** - If you change the spec, update related documents
5. **Add changelog entry** - Document your changes in CHANGELOG.md

---

## Documentation Standards

### Specification Documents

- Use clear, precise language
- Define terms before using them
- Include examples where helpful
- Cross-reference related sections
- Mark open questions explicitly

### Markdown Style

- Use ATX headers (`#`, `##`, not underlines)
- Use fenced code blocks with language identifiers
- Use tables for structured comparisons
- Include navigation links (back/next)

### Diagrams

- ASCII diagrams for simple flows
- Mermaid for complex diagrams (if supported)
- Keep diagrams close to related text

---

## Code of Conduct

### Expected Behavior

- Be respectful and constructive
- Assume good faith
- Focus on technical merit
- Credit others' contributions
- Accept feedback gracefully

### Unacceptable Behavior

- Personal attacks
- Harassment of any kind
- Dismissive or condescending responses
- Off-topic promotion

---

## Decision Process

### How Decisions Are Made

1. **Discussion** - Ideas are discussed openly
2. **Rough consensus** - We seek broad agreement, not unanimous votes
3. **Maintainer approval** - Maintainers make final calls on spec changes
4. **Documentation** - Decisions are documented with rationale

### When We Disagree

- Present technical arguments
- Consider alternative perspectives
- Accept maintainer decisions gracefully
- Document dissenting views if significant

---

## Getting Help

- **Questions about the protocol**: Open a Discussion
- **Found a bug**: Open an Issue
- **Want to contribute but unsure how**: Ask in Discussions
- **Security concerns**: Contact maintainers directly

---

## Recognition

Contributors will be recognized in:

- CHANGELOG.md (for significant contributions)
- README.md acknowledgments (for major contributions)
- Git history (always)

---

## License

By contributing, you agree that your contributions will be licensed under the same license as the project (MIT).
