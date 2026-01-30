# Contributing to Claude Code Skills

Thank you for your interest in contributing to this skills collection!

## How to Contribute

### Adding a New Skill

1. **Create the skill directory structure:**
   ```
   skill-name/
   ├── SKILL.md
   └── references/  (optional)
       └── *.md
   ```

2. **Follow the SKILL.md template:**
   ```markdown
   # Skill Name

   Brief description of what this skill provides.

   ## Keywords

   comma, separated, keywords, for, activation

   ## When to Use This Skill

   - Scenario 1
   - Scenario 2

   ## Related Skills

   - [related-skill](../related-skill) - Brief description

   ## Quick Reference

   | Task | Command |
   |------|---------|
   | Task 1 | `command` |

   ## Main Content

   Detailed knowledge and examples...
   ```

3. **Update README.md** to include your skill in the appropriate section.

4. **Submit a Pull Request** with a clear description of the skill's purpose.

### Updating Existing Skills

1. **Fork the repository** and create a feature branch.
2. **Make your changes** following the existing style and format.
3. **Test your changes** by using the skill with Claude Code.
4. **Submit a Pull Request** describing your changes.

## Style Guidelines

### Content

- Use clear, concise language
- Include practical examples with real commands
- Add code blocks with proper syntax highlighting
- Reference official documentation where appropriate

### Format

- Use consistent heading levels (## for main sections, ### for subsections)
- Include Quick Reference tables for common tasks
- Use fenced code blocks with language identifiers
- Keep lines under 100 characters where practical

### Keywords

- Choose keywords that uniquely identify the skill's domain
- Include common variations and synonyms
- Avoid overly generic terms that might cause false matches

### Cross-References

- Link to related skills using relative paths
- Link to shared references for common patterns
- Keep cross-references up to date when renaming skills

## Pull Request Process

1. **Ensure your changes follow the style guidelines** above.
2. **Update documentation** including README.md if needed.
3. **Describe your changes** clearly in the PR description.
4. **Wait for review** - maintainers will review and provide feedback.

## Code of Conduct

- Be respectful and constructive in discussions
- Focus on improving the skills collection for everyone
- Welcome newcomers and help them contribute

## Questions?

Open an issue with your question and we'll be happy to help!
