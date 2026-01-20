# Contributing to FEBO

Thank you for your interest in contributing to the FEBO specification!

## Ways to Contribute

### 1. Report Issues

- **Specification clarity**: If something is unclear or ambiguous
- **Errors**: Mathematical errors, typos, inconsistencies
- **Missing features**: Capabilities you need that FEBO doesn't support

Open an issue at [GitHub Issues](https://github.com/xtellix/febo-spec/issues).

### 2. Propose Extensions

If you want to propose new features:

1. Open an issue describing the use case
2. Explain why existing features don't cover it
3. Propose a solution that fits FEBO's philosophy

### 3. Submit Examples

We welcome real-world problem formulations:

1. Fork the repository
2. Add your example to `examples/`
3. Include comments explaining the problem
4. Submit a pull request

### 4. Improve Documentation

- Fix typos
- Clarify explanations
- Add diagrams or visualizations

## Guidelines

### Philosophy Alignment

FEBO contributions should align with the core philosophy:

> **Terms** define the basic energy contribution per variable set.
> **Interaction Structures** define which variable sets to include.
> **Aggregators** define how to combine term outputs.
> **Coupling Hamiltonians** define how subsystems interact.
> **Everything else is execution.**

### Specification Changes

For changes to the core specification:

1. **Backward compatibility**: Prefer additive changes
2. **Minimalism**: Only add what's necessary
3. **Orthogonality**: New features should be independent of existing ones
4. **Clear semantics**: Every feature must have precise meaning

### Code Style (for examples)

- Use clear, descriptive names
- Include comments explaining the problem
- Show the energy decomposition structure

## Review Process

1. Submit pull request
2. Maintainers review for alignment with FEBO philosophy
3. Community feedback period (for significant changes)
4. Merge or request revisions

## Code of Conduct

- Be respectful and constructive
- Focus on technical merit
- Welcome newcomers

## Questions?

Open a discussion on GitHub or contact the maintainers.

---

*Thank you for helping make FEBO better!*
