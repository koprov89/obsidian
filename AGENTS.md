**All tags use YAML frontmatter format (NOT inline hashtags).** Tags are in English with hyphen separators.
## Note Creation Guidelines

### Format Requirements
1. **Frontmatter:** Always include YAML frontmatter with tags and date
2. **Cross-links:** Use `[[Note Name]]` syntax to link to existing notes
3. **Images:** Use `![[filename.png]]` format for images


### Example Note Structure

```markdown
---
tags:
  - relevant-tag
  - another-tag
date: YYYY-MM-DD
---

# Note Title

Brief description or context.

## Section

Content with [[cross-links]] to related notes.

## Related Notes
- [[Related Note 1]]
- [[Related Note 2]]
```


**For AI Agents:** When creating new notes:
1. **File names:** Always use English for file names (e.g., "OpenCode_Configuration_Guide.md", not "OpenCode настройка.md"). Don't use spaces in file names
2. **Cross-linking (CRITICAL):** ALWAYS check if the note content can be linked to existing notes:
   - Before creating/editing a note, search the vault for related topics using glob/grep
   - Look for mentions of technologies, tools, concepts that might have existing notes
   - Add `[[wikilinks]]` to related notes in the content AND in a "Related Notes" section
3. Use appropriate existing tags
4. Place in correct folder based on content type   
5. Follow the frontmatter format consistently
6. Use available templates from `Templates/` folder. If needed, create new templates
7. If asked to save article from link into note, save the full article content, not just summary
8. if asked to summarize article from link, create a new note with full summary and proper tags.

